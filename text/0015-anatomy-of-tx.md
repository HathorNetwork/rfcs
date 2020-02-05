- Feature Name: Anatomy of a transaction
- Start Date: 2019-08-07
- RFC PR:
- Hathor Issue:
- Author: Yan Martins <yan@hathor.network>

# Summary
[summary]: #summary

The main goal here is to document the current structure of a transaction in Hathor and invite developers to debate
possible improvements.

# Motivation
[motivation]: #motivation

This RFC will serve as documentation about our transactions and blocks.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Like Bitcoin, Hathor has the concept of blocks and transactions, although they are arranged in a different architecture.
In traditional blockchains, transactions are grouped inside blocks and these form a chain, meaning that every new block
points to the previous one.

In Hathor, transactions are not grouped inside a block. Instead, transactions and blocks form a direct acyclic graph
(DAG), with blocks also forming a blockchain. Transactions and blocks point to 2 previous transactions and blocks also
point to the previous block. This last part is responsible for forming the blockchain in the DAG. When transaction A
points to transaction B, we say that B is a parent of A.

```
                                                                 +-------------+
                                                                 |             |
  +-------------+                                                |     txB     |
  |             |                                                |             | <-----+
  |     txB     | <-----+                                        +-------------+       |
  |             |       |                                                              |
  +-------------+       |    +-------------+                     +-------------+       |    +-------------+
                        |    |             |                     |             |       |    |             |
                        +--+ |     txA     |                     |   blockY    | <-----+--+ |   blockX    |
                        |    |             |                     |             |       |    |             |
  +-------------+       |    +-------------+                     +-------------+       |    +-------------+
  |             |       |                                                              |
  |     txC     | <-----+                                        +-------------+       |
  |             |                                                |             | <-----+
  +-------------+                                                |     txC     |
                                                                 |             |
                                                                 +-------------+

        txB and txC are parents of txA                               A block points to 2 transactions and
                                                                              the previous block
```

Transactions have a list of inputs and outputs and also use a stack based scripting language for validation. The
input's data must correctly "solve" the referenced output's script. With a few exceptions, the sum of inputs and
outputs must be the same (we'll detail the exceptions later).

On traditional blockchains, transactions usually pay a fee to be included in a block and there are only so many
transactions a block can include (this is limited by the block size). In Hathor, since transactions are not contained
inside blocks and can be easily propagated, there must be a spam prevention mechanism. That's why we've added
proof-of-work (PoW) for transactions also, similar to a block PoW. This means some work has to be done before
propagating the transaction.

Hathor blocks are very simple. As there are no transactions inside, they are pretty lightweight and fast to propagate.
Blocks also have to solve a PoW to be valid, which is adjusted depending on the network hash rate so one new block is
created every 30 seconds, on average.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This table summarizes all fields that are common for blocks and transactions:

| Field             | Size                          | Description          
| ----------------- | ----------------------------- | --------------------------
| version           | 2 bytes                       | Integer with version
| timestamp         | 4 bytes                       | UTC timestamp in seconds from epoch
| hash              | 32 bytes                      | sha256d hash
| nonce             | 4 or 16 bytes                 | The nonce which solves the PoW correctly
| weight            | 8 bytes                       | Double, with the block/transaction weight
| parents           | 64 or 96 bytes                | Hash of the parents (2 for transactions and 3 for blocks)
| inputs            | variable                      | List of transaction inputs (blocks don't have any)
| outputs           | variable                      | List of outputs

The table gives us a high level overview of the anatomy of blocks and transactions. There are some fields that exist
only for one of them. These will be detailed in later sections.

From now on, we'll use _tx_ or _txs_ to refer to both blocks and transactions. If we want to refer to only one, we'll
explicitly use either block or transaction.

## Version

The version field takes up 2 bytes. The left-most byte is reserved for future use and ignored for now. This means that
a version with value [0xAA, 0x01] should be interpreted just as `version=1`.

Each version defines a diferent type of tx, usually with different serialization and particular validations. So far,
these are the existing versions and txs:  
0: block  
1: regular transaction  
2: token creation transaction  
3: merged mining block  

## Timestamp

All txs are timestamped in UTC time from epoch. A timestamp of 1565738290 corresponds to August 9th, 2019 23:18:10 UTC.

## Hash

The hash of a tx uniquely identifies it. It's a sha256d hash, which means sha256 is applied twice (double sha256).

The steps for finding the hash are:
1. Calculate the tx weight. It's different for blocks and transactions and discussed later on;
2. From the weight, derive the target;
3. Mine the tx until the hash meets the target;

### Weight

The weight simbolizes how much work has been put in mining a tx. The larger it is, the harder it is to solve the PoW.

##### Blocks

The weight for blocks is adjusted with the network hash rate, so the network produces a new block every 30 seconds, on
average.
TODO write more about hash rate estimation and block weight. Maybe a separate document?

##### Transactions

For transactions, there's no influence from the network when calculating the weight. It only takes into account its
size, funds moved and some other parameters from the transaction.

TODO add formula and example

### Target

The target is derived from the weight following this formula:

```
target = 2**(256 - weight) - 1
```

It follows that, the larger the weight, the smaller the target is. This means txs with larger weight have a smaller
target to reach and have to do more PoW to find the hash.

This code in C details how to calculate the target from weight:
```c
void weight_to_target(uint32_t* target, double weight) {
    double calc = pow(2, 256 - weight);
    for(int i=0; i<8; ++i) {
        target[i] = fmod(calc, 4294967296L);
        calc /= 4294967296L;
    }
    for(int i=0; i<8; ++i) {
        if (target[i]) {
            target[i] = target[i] - 1;
            break;
        }
        target[i] = 4294967295L;
    }
}
```

### Mining

This step is very similar to Bitcoin's mining process. We increment the nonce until `sha256d(tx) < target`.

A question that follows from `sha256d(tx)` is how to apply the hash function to a tx. Initially, the function was
applied to the serialized format of the tx. This was changed to adapt to the existing `sha256d` mining software. They
are optimized to mine over 80 bytes, which is the size of the Bitcoin header. Therefore, we adjusted to also mine over
a 80-byte structure.

The solution for this was splitting the tx in two structures:
- funds_struct: has the fields related to the token transfer, as inputs and outputs;  
- graph_struct: has the fields related to the DAG, such as weight and parents;  

We apply `sha256d` to the hash of each structure and the nonce (which has 16 bytes):
```
             32 bytes               32 bytes        16 bytes
                 |                      |               |
        _________.__________   _________._________    __.__
sha256d(sha256(funds_struct) + sha256(graph_struct) + nonce)
```

The graph and funds structures are different for each type of tx (transactions or blocks) and will be detailed in the
Serialization section.

> NOTE: for transactions, the actual size of the nonce is 4 bytes. However, to mine it following this same idea, we use
a padding of 12 null-bytes, e.g., `nonce = 0x12345678` will be serialized as `[0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
0x00, 0x00, 0x00, 0x00, 0x00, 0x00 0x12, 0x34, 0x56, 0x78]`. This padding is used only when mining.

## Inputs

An input is a reference to an output from a previous transaction. We say that the input spends this output. There are
only 3 fields:

| Field     | Size              | Description          
| --------- | ----------------- | --------------------------
| tx_id     | 32 bytes          | Hash of the tx being spent
| index     | 1 byte            | Index of the output being spent in the above tx
| data      | variable          | Data that successfully solves the output script

We often talk about the sum of inputs, even though inputs don't have a value set explicitly. In this case, we're
refering to the value of the output spent by this input.

##### Input data

There's no fixed format for input data. It's used in conjunction with the script of the output being spent. More details on the "Output script verification" section.

## Outputs

Outputs have the same fields `value` and `script` as Bitcoin and work the same way. However, there's an extra `token_data`
field, responsible for token operations.

| Field         | Size              | Description          
| ------------- | ----------------- | --------------------------
| value         | 4 or 8 bytes      | Amouont of tokens that this output will be worth when claimed
| token_data    | 1 byte            | Information about the token this output references
| script        | variable          | Script that must be satisfied to spend this output

`value` can have 4 or 8 bytes, depending on the amount it holds. The first bit is used to signal if it has 4 or 8 bytes.
When it's 0, it uses 4 bytes; otherwise, 8 bytes. This means that a 4-byte value can hold an amount of 2^31. Anything
larger than that needs 8 bytes. It's also important to notice that using 8 bytes for an amount that fits in 4 bytes is
invalid and the tx will be rejected. For eg, using `0b1000...0001` (8 bytes) to represent a value of 1 is invalid.

The `script` and its matching input `data` will be discussed in more detail in a later section.

`token_data` merges two fields: first bit is the authority flag, while remaining bits represent the token index.
- authority flag: if 0, this is a regular output; if 1, it's an authority output;
- token index: indicates which token this output references. If 0, it's the native Hathor token (HTR). Otherwise, we get
the token's UID from the tokens array in the transaction.

For more information about tokens, check the [tokens RFC](https://gitlab.com/HathorNetwork/rfcs/blob/master/text/0004-tokens.md).

## Blocks

| Field             | Size                          | Description          
| ----------------- | ----------------------------- | ------------------------------------
| version           | 2 bytes                       | Integer with version
| timestamp         | 4 bytes                       | Timestamp in seconds from epoch
| hash              | 32 bytes                      | sha256d hash
| nonce             | 16 bytes                      | The nonce which solves the PoW correctly
| weight            | 8 bytes                       | Float with the block weight
| parents           | 96 bytes (3\*32)              | Hash of the 3 parents
| outputs           | variable                      | List of outputs
| data              | variable (100-byte limit)     | Used for mining

A block has `version=0`.

Blocks have exactly 3 parents: 1 block followed by 2 transactions. The former is responsible for creating the
blockchain inside the DAG and must be the first one in the parents list.

The outputs can only contain Hathor tokens and their sum must be equal to the reward amount.

The data field is used in the mining process. Miners can put whatever information they want, such as their identifier
or their lat/long coordinates. It may also be used to allow others to do merged mining with Hathor, just like we're
doing with Bitcoin.

### Merged mining blocks

Merged mining blocks use `version=3` and are slightly different from regular blocks. Details can be found on the [Merged mining RFC](https://gitlab.com/HathorNetwork/rfcs/blob/master/text/0006-merged-mining-with-bitcoin.md).

## Transactions

| Field             | Size              | Description          
| ----------------- | ----------------- | --------------------------
| version           | 2 bytes           | Integer with version
| timestamp         | 4 bytes           | Timestamp in seconds from epoch
| hash              | 32 bytes          | sha256d hash
| nonce             | 4 bytes           | The nonce which solves the PoW correctly
| weight            | 8 bytes           | Float with the transaction weight
| parents           | 64 bytes (2\*32)  | Hash of the 2 parents
| tokens            | 32 * n            | List of token UIDs referenced in this transaction
| inputs            | variable          | List of inputs
| outputs           | variable          | List of outputs

Transactions use `version=1` and its two parents are both earlier transactions.

The token field is an array of 32-byte token UIDs (`[uid1, uid2, uid3, ..., uidN`).

### Special transactions

In general, transactions follow the rule that the sum of outputs should be equal to the sum of inputs, for each token.
We validate the sum for each token, not the total sum - e.g., the following transaction is invalid:

Inputs: token A (200) / token B (300)  
Outputs: token A (50) / token B (450)

Also, when considering token authority operations, the outputs can only have the same or less authorities than the
inputs. A transaction with only mint authority, for instance, cannot have melt authority.

There are, however, special transactions that are allowed to break the rules.

##### Token mint/melt

When minting or melting, we break the rule for 2 different tokens.

Minting means we're creating new tokens, so the balances for the minted token won't match. The sum on the outputs is
larger than the inputs. Besides that, we also have the HTR deposit when minting, so the sum of inputs and outputs for
HTR tokens is also not the same.

When melting, the opposite happens. The sum of the melted token is larger on the inputs. We also withdraw HTR tokens,
so the sum of HTR is larger on the outputs.

More about token deposit can be found on the [token deposit RFC](https://gitlab.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md).

##### Token creation

A token creation transaction breaks the rules in various ways. There's the initial mint of tokens in this transaction,
so it's the same case as explained in the last section, when minting tokens. Besides that, we might also create new
authority outputs "from thin air".

This transaction also has extra information about the token being created and needs other types of validation. For
this reason, token creation transactions have `version=2`. The fiels for this transaction are:

| Field              | Size                      | Description          
| ------------------ | ------------------------- | --------------------------
| version            | 2 bytes                   | Integer with version
| timestamp          | 4 bytes                   | Timestamp in seconds from epoch
| hash               | 32 bytes                  | sha256d hash
| nonce              | 4 bytes                   | The nonce which solves the PoW correctly
| weight             | 8 bytes                   | Float with the transaction weight
| parents            | 64 bytes (2\*32)          | Hash of the 2 parents
| token_info_version | 1 byte                    | Version of the token information
| token_name         | variable (30-byte limit)  | Name of the created token
| token_symbol       | variable (5-byte limit)   | Symbol of the created token
| inputs             | variable                  | List of inputs
| outputs            | variable                  | List of outputs

For more details, read the [tokens RFC](https://gitlab.com/HathorNetwork/rfcs/blob/master/text/0004-tokens.md).

## Validation

There are two types of rules analyzed in all txs. The first ones are considered "local rules", as they can be enforced
without any knowledge about the DAG state. They are things like weight, nonce, number of parents, balance of inputs and
outputs, etc - more details in the next few sections. A tx is considered invalid if it breaks any of these rules. In
general, we enforce them when we receive the tx. It it's invalid, it's immediately discarded and won't be stored or
propagated.

After a tx is validated, it'll be verified against consensus rules. These depend on the DAG topology. That's what
happens, for example, when two transactions try to spend the same output. In this situation, the consensus protocol
decides which one is executed by the network - i.e. the "winner". The other conflicting tx is considered to be voided.
The status of a tx as voided or executed may change as new txs arrive and the DAG is updated. That's why we propagated
and store all valid txs, either voided or executed.

We'll only deal here with the first set of rules. Consensus rules are detailed in the [consensus RFC](TODO add link).

First we'll address rules that are common to blocks and transactions and later go into specific rules for each.

### Parents

All tx parents must exist. A tx with any unknown parent is not valid.

Moreover, blocks have exactly 3 parents, the first of which must be another block while the 2 others are transactions.
Transactions, on the other hand, only have 2 parents and both are other transactions.

### Timestamp

A tx timestamp must be greater than its inputs and parents. For example, if all parents and inputs have
timestamp=1565738290, the tx timestamp must be 1565738291 or greater. This rule has important consequences to the
network behavior: no tx can spend another tx that was created on the same second.

The network also does not allow txs with timestamp too far in the future. We currently only accept txs that are up to
5 minutes ahead of the current time. Therefore, it's important to keep the fullnode clock updated.

### Proof-of-Work

All txs must have done its PoW before being propagated. PoW validation is very simple:
1. Calculate tx weight, as described earlier. We'll call it `calculated_weight`;
2. Verify the weight field value on the tx is equal or greater than `calculated_weight`;
3. Derive the target from the weight value;
4. Finally, the tx hash must be smaller that the target.

### Number of inputs and outputs

Transactions and blocks can have at most 255 outputs.

Blocks have no inputs and transactions can have at most 255 inputs.

### Output serialization

The value of outputs are serialized using either 4 or 8 bytes. The specifics are detailed in the 'Output value' serialization
section, but it's important to keep in mind that it's invalid to serialize in 8 bytes a value that fits in 4 bytes.

#### Blocks

The specific rules for validating blocks are:

##### Timestamp

Blocks have one extra timestamp rule. The maximum time interval between consecutive blocks is `30*64` seconds. This
prevents some DoS attacks exploiting the calculation of the score of a side chain. More details [here](TODO write about it).

The only exception to this rule is the genesis and the block right after it. In that case, we allow any amount of time
between them.

##### Outputs

The sum of outputs must be exactly equal to the block rewards. The correct reward amount depends on the height of the
block, as Hathor has annual halvings until its fourth year:
Year 1: 64 HTR
Year 2: 32 HTR (from block height 1,051,200)
Year 3: 16 HTR (from block height 2,102,400)
Year 4 onwards: 8 HTR (from block height 3,153,600)

Also, the outputs cannot reference any token other than HTR.

##### Data field

We enforce the data field has at most 100 bytes.

#### Transactions

The specific rules for validating transactions are:

##### Output script verification

Hathor uses a scripting system for transactions, very similar to Bitcoin's. It is simple, stack-based, and processed from left to right. It is intentionally not Turing-complete, with no loops.

When a transaction arrives, as part of the validation, each input data is merged with the corresponding spent output script and evaluated. If the stack is empty or the final item is True, it means the input correctly spends the output. Otherwise, the whole transaction is invalid.

We execute evaluation on `[input_data + output_script]`. For more details, refer to the [Bitcoin script wiki](https://en.bitcoin.it/wiki/Script).

Hathor does not support all OPCODE operations existing in Bitcoin. For a list of supported OPCODEs, check [here](https://github.com/HathorNetwork/hathor-core/blob/master/hathor/transaction/scripts.py#L123).

##### Inputs and outputs sum

In general, the sum of inputs and outputs, for each token, must be equal. Breaking this rule, except for the cases
discussed in the Special Transactions section, means the transaction is not valid.

## Serialization

The way txs are serialized has major implications. Two of the most important are:
- space efficiency: a good serialization format will not waste unnecessary bytes, which affects both network and
storage requirements. A few bytes doesn't seem like a big deal but, considering Hathor's claims of high TPS, it can
have significant impact;
- node interoperability: it's crucial that nodes in the network can serialize and deserialize txs in the same way,
otherwise they won't be able to cooperate;

Expanding on this last item, as Hathor gets more support, developers will have to deal with serialization in languages
other than Python, the language for Hathor's official full-node implementation. This is already true for the wallet
library, which is developed in Javascript. Therefore,a clear serialization structure is fundamental.

When serializing a tx, we do it in two main pieces: the funds and graph structures. Apart from those, we only have the
nonce. In summary, a tx is serialized with these three parts:

| Part          | Description                           |
| ------------- | ------------------------------------- |
| funds struct  | Info about tokens, inputs and outputs |
| graph struct  | Info about parents and weight         |
| nonce         | Used for mining                       |

As explained on the Hash section, we have these two structures to adapt to the existing Bitcoin mining software. We
also use them for serialization for simplicity (reusing existing code).

The next few sections will detail all aspects regarding serialization, with examples.

### Endianness

Hathor uses big-endian representation - i.e. the 16-bit integer `0x1234` is serialized as `[ 0x12, 0x34 ]`.

## Double precision values

Double precision numbers use the [IEEE 754 binary64](https://en.wikipedia.org/wiki/Double-precision_floating-point_format#IEEE_754_double-precision_binary_floating-point_format:_binary64)
format.

### Variable length content

Most fields have fixed length. For example, the version field always has 2 bytes. Some fields, however, have variable
length. The block data field, for example, can have any length between 0 and 100 bytes.

For these fields we prefix the length when serializing. So for block data field with content `[0xE2, 0x02, 0x91]` its
length is 3 (0x03 in hex) and the serialized form is `[0x03, 0xE2, 0x02, 0x91]`. On this case we used 1 byte for the
length prefix, meaning the maximum number we can represent is 255 (1-byte unsigned integer). In case the length is
greater than 255, more bytes may be used.

The output script field, for instance, uses 2 bytes for its length prefix. It's always 2 bytes, no matter the length of
the script. If it's only 11 bytes, the prefix will be `[0x00, 0x0B]`. The size of this prefix length is fixed for each
field. There's *no* variable length integer serialization like [Bitcoin's](https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer).
The following table details the size of the prefix for all variable length fields in a tx.

| Field             | Prefix        |
| ----------------- | ------------- |
| input.data        | 2 bytes       |
| output.script     | 2 bytes       |
| block.data        | 1 byte        |
| token_name        | 1 byte        |
| token_symbol      | 1 byte        |

### Inputs

The input fields are:

| Field     | Format    | Size      |
| --------- | --------- | --------- |
| tx_id     | bytes     | 32 bytes  |
| index     | integer   | 1 byte    |
| data      | bytes     | variable  |

Both `tx_id` and `data` are already bytes, so there's no serialization. `index` is easy to be converted, as it's a
1-byte unsigned integer.

For example:
```
tx_id: [0x83, 0x46, ..., 0xF2, 0xBC] (32 bytes)
index: 5
data: [0x99, 0xAE, 0x54, 0xC3]
```

The serialized format is:

```
           tx_id             index  length prefix       data
             |                 |         |                |
 ____________.______________   .    _____.____  __________.___________
[0x83, 0x46, ..., 0xF2, 0xBC, 0x05, 0x00, 0x04, 0x99, 0xAE, 0x54, 0xC3]
```

Note: the above data with 4 bytes was used only for easier understanding. Although we don't have lower limits for this
field, it usually contains the tx signature, so it'll be much larger.

### Outputs

The output fields are:

| Field         | Format    | Size          |
| ------------- | --------- | ------------- |
| value         | integer   | 4 or 8 bytes  |
| token_data    | integer   | 1 byte        |
| script        | bytes     | variable      |

There's no need to convert `script` as it's already in bytes. `token_data` is used for token information and serialized
as an unsigned integer. The `value` field has a particular serialization depending on the amount.

#### Output value

Output values may be serialized in 4 or 8 bytes, both using big-endian representation. When serializing, the first bit
indicates which size is used: if it's 0, it uses 4 bytes; if it's 1, it uses 8 bytes. The maximum value that can be
serialized in 4 bytes is 2\*\*31 - 1, which is 2147483647. Values larger than that, up to the maximum of
9223372036854775808 (2\*\*63), are encoded using 8 bytes.

Serializing using 4 bytes is pretty straight-forward. Some examples:
```
        4 = [0x00, 0x00, 0x00, 0x04]
   123456 = [0x00, 0x01, 0xe2, 0x40]
123456789 = [0x07, 0x5b, 0xcd, 0x15]
```

When using eight bytes, the negative of the number is serialized. This makes sure the first bit is 1. For example, to
serialized the value 21474836470, we get the serialization of -21474836470.
```
21474836470 = [0xff, 0xff, 0xff, 0xfb, 0x00, 0x00, 0x00, 0x0a]
81321474836470 = [0xff, 0xff, 0xb6, 0x09, 0xde, 0x61, 0x38, 0x0a]
```


Considering the following output:
```
value: 50
token_data: 0
script: [0xB1, 0x08, 0x33, 0xD7, 0x45]
```

The serialized format is:

```
value  token_data  length prefix   script
  |     |          |                 |
  .     .    ______.___  ____________._________
[0x32, 0x00, 0x00, 0x04, 0x99, 0xAE, 0x54, 0xC3]
```

### Blocks

Serialized format is `[funds_struct, graph_struct, nonce]`, where nonce has 16 bytes.

#### Funds struct

| Field             | Size                  |
| ----------------- | --------------------- |
| version           | 2 bytes               |
| outputs_length    | 1 byte                |
| outputs           | variable              |

Block version is 0. Considering only one output, the funds struct serialized format is:
```
  version    outputs_length
     |         |
 ____._____  __._
[0x00, 0x00, 0x01, <output0>]
```

`<output0>` is the serialized format of the output, described earlier.

#### Graph struct

| Field             | Size            |
| ----------------- | --------------- |
| weight            | 8 bytes         |
| timestamp         | 4 bytes         |
| parents_length    | 1 byte          |
| parents           | 96 bytes        |
| data              | variable        |

Let's suppose we have a *block* with weight 17.23, timestamp 1566222309 and `data = [0xE3, 0x97]`. Version is 0. The
graph struct serialized format is:
```
                    weight                             timestamp    parents_length         parents          data_length   data
                       |                                   |              |                   |                  |         |
 ______________________._______________________  __________.___________  _.__  _______________._______________  _.__  _____.____
[0x40, 0x31, 0x3A, 0xE1, 0x47, 0xAE, 0x14, 0x7B, 0x5D, 0x5A, 0xA7, 0xE5, 0x03, <parent0>, <parent1>, <parent2>, 0x02, 0xE3, 0x97]
```

`weight` is a double precision number. `timestamp` is an unsigned integer, so the conversion is pretty straight-forward
(1566222309 = 0x5D5AA7E5). `<parentN>` is the parent's hash, which is already in bytes format.

### Transactions

Serialized format is `[funds_struct, graph_struct, nonce]`, where nonce has 4 bytes.

#### Funds struct

| Field             | Size                  |
| ----------------- | --------------------- |
| version           | 2 bytes               |
| tokens_length     | 1 byte                |
| inputs_length     | 1 byte                |
| outputs_length    | 1 byte                |
| tokens            | 32 * tokens_length    |
| inputs            | variable              |
| outputs           | variable              |

Let's suppose we have a transaction with 1 input, 2 outputs and 1 token UID. Transaction version is 1. The funds struct
serialized format is:
```
                inputs_length
       tokens_length |
  version     |      |   outputs_length
     |        |      |     |
 ____._____  _.__  __._  __._
[0x00, 0x01, 0x01, 0x01, 0x02, <token_uid0>, <input0>, <output0>, <output1>]
```

#### Graph struct

| Field             | Size          |
| ----------------- | --------------|
| weight            | 8 bytes       |
| timestamp         | 4 bytes       |
| parents_length    | 1 byte        |
| parents           | 64 bytes      |

Let's suppose we have a transaction with weight 17.23 and timestamp 1566222309. The graph struct serialized format is:
```
                    weight                             timestamp    parents_length    parents
                       |                                   |              |              |
 ______________________._______________________  __________.___________  _.__  __________._________
[0x40, 0x31, 0x3A, 0xE1, 0x47, 0xAE, 0x14, 0x7B, 0x5D, 0x5A, 0xA7, 0xE5, 0x02, <parent0>, <parent1>]
```

`weight` is a double precision number. `timestamp` is an unsigned integer, so the conversion is pretty straight-forward
(1566222309 = 0x5D5AA7E5). `<parentN>` is the parent's hash, which is already in bytes format.

The token UID is already in bytes, so there's no conversion necessary. `<inputN>` and `<outputN>` are the serialized
forms of input and output already detailed earlier.

#### Token creation transaction

This special transaction has the **same graph structure**, but a slightly **different funds struct**. The funds struct
is:

| Field                 | Size            |
| --------------------- | --------------- |
| version               | 2 bytes         |
| inputs_length         | 1 byte          |
| outputs_length        | 1 byte          |
| inputs                | variable        |
| outputs               | variable        |
| token_info_version    | 1 byte          |
| token_name_length     | 1 byte          |
| token_name            | variable        |
| token_symbol_length   | 1 byte          |
| token_symbol          | variable        |

Let's suppose we have a token creation transaction with 1 input and 2 outputs. Transaction version is 2 and token info
version is 1. The token name is MyToken (`[0x4D, 0x79, 0x54, 0x6F, 0x6B, 0x65, 0x6E]`) and symbol is MTK (`[0x4D, 0x54,
0x4B]`). The serialized format is:
```
         inputs_length
  version      |   outputs_length          token_info_version  token_name_length    token_name    token_symbol_length    token_symbol
     |         |     |                                    |      |                      |                       |           |
 ____._____  __._  __._                                  _.__  __._  ___________________.____________________  _.__  _______.________
[0x00, 0x02, 0x01, 0x02, <input0>, <output0>, <output1>, 0x01, 0x07, 0x4D, 0x79, 0x54, 0x6F, 0x6B, 0x65, 0x6E, 0x03, 0x4D, 0x54, 0x4B]
```


# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives


# Prior art
[prior-art]: #prior-art

Traditional blockchains have transactions grouped inside blocks. That is the main difference when comparing the
structures. Other parts are very similar, like the structure of a transaction (basically a list of inputs and outputs),
the hash mechanism, mining, etc.

IOTA is one of the few cryptos with a different architecture, using a DAG of transactions. In their case, there are no
blocks, so there's no blockchain in the middle of the DAG.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

- should we use variable length integer serialization like Bitcoin? https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer;


# Future possibilities
[future-possibilities]: #future-possibilities

- we can use segwit to reduce storage requirements: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
