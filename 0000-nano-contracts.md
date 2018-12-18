- Feature Name: nano_contracts
- Start Date: 2018-11-07
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)

# Summary
[summary]: #summary

Support very simple Smart Contracts, in which two or more people transfer their tokens to a contract and the winner takes all tokens. A contract is simply a set of rules applied to decide the final distribution of tokens.

# Motivation
[motivation]: #motivation

The motivation is to allow people to settle contracts using Hathor's network. A contract is a set of rules that can't be changed (as it's written on the Hathor network) and will be applied to decide the final distribution of tokens.

There are multiple applications for nano contracts, such as hedging (options and future contracts) and betting.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A Nano Contract is a special output in a transaction. This output can only be spent when a set of rules is valid. This set of rules is flexible and configured by the creators of the contract. 

There will exist an external source of truth, which will provide data that will be used to execute or terminate the contract. This external source of truth is called Oracle.

A transaction will only be considered a Nano Contract if it depends on data from oracles for its execution. Elaborate scripts can be created for other situations and not need oracle information (for eg, token inheritance after X years if not spent), so they're not Nano Contracts.

Differently from Ethereum's Smart Contracts, Nano Contracts are not Turing complete. Most importantly, it's not possible create loops in the contract execution code. Hathor's Nano Contracts uses a simple stack based language, similar to the what is already used in a transaction output script.

Imagine that two people would like to bet 1 token each in which country will win the World Cup. The rules would be: each of them will choose a country and the winner receives 2 tokens. If both are wrong, each one gets their 1 token back. After the World Cup final, the oracle will publish the result and the interested parties can use this data to execute the contract.

It's important to understand that nano contracts don't take any action by themselves. The interested party in the outcome of the event has to create a transaction that will settle the contract, with the data provided by the oracle.

In the World Cup example, that means someone (most likely, but not necessarily, the bet winner) would create a transaction with the World Cup result and send to the network in order to have the contract tokens transferred to his account.

// TODO having an expiration date should be mandatory or just good practice?

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## ORACLE

The oracle has a somewhat rigid format for its data. It should be in the format `value1:value2:value3:...:valueN`.

An example of data is `key:timestamp:payload`. where:
`key`: identifies the event this nano contract is about. In a stock futures contract, it could be the stock ticker symbol;
`timestamp`: when was this information provided. A stock can have value X at time T1 and value Y at time T2;
`payload`: the actual value, the stock price.

Another example: let's say it's a bet on a game result (Brazil x Argentina). The oracle's id for this game is 55 (some unique identifier), and it will return the results as follows:
0 - if Brazil wins;
1 - if Argentina wins;
2 - it's a draw.
If Brazil wins, the oracle returns `55:0`, for example.

This data format should be agreed between the oracle and contract participants upon contract creation. It's recommended that the oracle provides the data specification and also signs it, to prove its identity. Contract participants should save the message and signature in case there is any dispute in the future.

## Opcodes

The Nano Contract has a few special opcodes in the scripting language of the outputs.

### OP_CHECKDATASIG

It consumes three parameters from the stack: `<data> <signature> and <pubKey>`. It checks the data and signature against the pubkey. If the signature is valid, it pushes DATA to the stack. Otherwise, it fails.

It's important that the contract's script contains the oracle public key hash, to appoint who is the oracle for this contract. For example, the following script would check whether the data was signed by the correct oracle or not: `OP_DUP OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKSIG_ORACLE`.

The expected input data must be: `<data> <signature> <pubKey>`.

After running, the stack would be: `<data>`.

This should be the first step when creating a nano contract most of the times.

### OP_MATCH_INTERVAL

It consumes a dynamic number of parameters from stack: `<x> <pubkeyHash1> <a1> <pubkeyHash2> <a2> <pubkeyHash3> … <pubkeyHashN> <an> <pubkeyHash(N+1)> <n>` and checks in which interval the number `<x>` belongs to. `<n>` is the number of points in the interval (1 fewer than public keys).

If it belongs to (-inf, a1], pubkeyHash1 wins, if it belongs to (a1, a2], pubkeyHash2 wins, if it belongs to (a2, a3], pubkeyHash3 wins, and so on, until (a_(n-1), an] and (an, inf).

After running, it pushes only the winner `<pubkeyHash>` to the stack.

### OP_MATCH_VALUE

It consumes a dynamic number of parameters from stack: `<x> <fallbackPubkeyHash> <a1> <pubkeyHash1> <a2> <pubkeyHash2> <a3> <pubkeyHash3> … <an> <pubkeyHashN> <n>` and checks which number `<x>` is equal to. `<n>` is the number of values to be checked.

If it's equal to a1, pubkeyHash1 wins, if equals to a2, pubkeyHash2 wins, and so on. If it doesn't match anything, fallbackPubkeyHash wins.

After running, it pushes only the winner `<pubkeyHash>` to the stack.

During contract creation, it should be enforced that each `<an>` is different; in other words, no one can bet on the same value.

### OP_GET_DATA_STR, OP_GET_DATA_INT

It consumes two parameters from the stack: `<data> <k>`. Extracts the kth value from `<data>` as a string (_STR) or an integer (_INT). Remember that the oracle data has the format `value1:value2:value3:...:valueN`

Leaves on the stack <oracleData> <extractedValue>

### OP_DATA_STREQUAL

Equivalent to an OP_GET_DATA_STR followed by an OP_EQUALVERIFY. Consumes three parameters from stack: `<data> <k> <value>`. Gets the kth value from `<data>` as a string and verifies it's equal to `<value>`.

Leaves on stack `<data>`; or fails, if not equal.

Eg: `<data> <0> <key> OP_DATA_STREQUAL` gets the first value from `<data>` and compares it to `<key>`. Leaves `<data>` on stack.

### OP_DATA_GREATERTHAN

Equivalent to an OP_GET_DATA_INT followed by an OP_GREATERTHAN. Consumes three parameters from stack: `<data> <k> <n>`. Gets the kth value from `<data>` as an integer and verifies it's greater than `<n>`.

Leaves on stack `<data>`; or fails, if not greater than `<n>`.

Eg: `<data> <1> <n> OP_DATA_STREQUAL` gets the second value from `<data>` and checks it's greater than `<n>`. Leaves `<data>` on stack

### OP_DATA_MATCH_INTERVAL

Equivalent to an OP_GET_DATA_INT followed by an OP_MATCH_INTERVAL.

Leaves on stack `<pubkeyHash>`.

Eg: `<data> <2> <PubKeyHash_A> <5.00> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL` gets third value from `<data>` as an integer and finds the matching interval. Leaves `<pubkeyHash>` on stack.

### OP_DATA_MATCH_VALUE

Equivalent to an OP_GET_DATA_INT followed by an OP_MATCH_VALUE.

Leaves on stack `<pubkeyHash>`; or fails, if no value matches.

Eg: `<data> <2> <PubKeyHash_A> <5.00> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL` gets value 2 from data as an integer and finds the matching interval. Leaves `<pubkeyHash>` on stack

## Examples

### Bet on a value

Example of a contract in which two persons (A and B) bet 1 token each on the price of Hathor 2 months from now. If the price of Hathor is above $5.00, B receives 2 tokens. Otherwise, A receives 2 tokens. Then, the script of the output would be:

First part: `OP_DUP OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG`.
Confirms the data actually came from the oracle identified by `<oraclePubKeyHash>`

Then: `<0> <key> OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN`.
Checks that the first value from oracle data (field 0) is equal to `<key>`. Then, checks second value (field 1) is greater than the timestamp agreed on the contract (2 months from now in our example)

And: `<2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL`
Checks in which interval the oracle payload data (third value) is and leaves the winner `pubkeyHash` on the stack

Finally: `OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`
Similar to the P2PKH sequence, but we use `OP_OVER` because top item stack will be public key hash.

The data of the input spending this output would be:

`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey>`, where `<oracleData>` is in the format `key:timestamp:payload` and `<txSignature>` is the transaction signature, like regular P2PKH transactions.

Putting everything together, the execution would be:

Stack=`[]`  
Script=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey> OP_DUP OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey>`  
Script=`OP_DUP OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey> <oraclePubKey>`  
Script=`OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey> <oraclePubKeyHash>`  
Script=`<oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey> <oraclePubKeyHash> <oraclePubKeyHash>`  
Script=`OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey>`  
Script=`OP_CHECKDATASIG <0> <key> OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData>`  
Script=`<0> <key> OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <0> <key>`  
Script=`OP_DATA_STREQUAL <1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData>`  
Script=`<1> <timestamp> OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <1> <timestamp>`  
Script=`OP_DATA_GREATERTHAN <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData>`  
Script=`<2> <PubKeyHash_A> <5> <PubKeyHash_B> <1> OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <2> <PubKeyHash_A> <5> <PubKeyHash_B> <1>`  
Script=`OP_DATA_MATCH_INTERVAL OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

(Suppose data says price is below $5)

Stack=`<txSignature> <pubkey> <PubKeyHash_A>`  
Script=`OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <PubKeyHash_A> <pubkey>`  
Script=`OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <PubKeyHash_A> <pubkeyHash>`  
Script=`OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey>`  
Script=`OP_CHECKSIG`

Stack=`<TRUE>`  
Script=

### Bet on a game result

Example of a contract in which two persons (A and B) bet 1 token each on the Brazil x Argentina game result. If Argentina wins, A gets the tokens; if Brazil wins, B gets the tokens. Oracle will return 0 if Argentina wins; 1 if Brazil wins. Then, the script of the output would be:

First part: `OP_DUP OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG`.
Confirms the data actually came from the oracle identified by `<oraclePubKeyHash>`

Then: `<0> <key> OP_DATA_EQUAL_INT`.
Checks that the first value from oracle data (field 0) is equal to `<key>`. We assume `<key>` is of type integer (it might be the id of this game in the oracle's database system). In this case, we don't care about timestamp. We assume the id is unique (Brazil x Argentina games in different dates would have different ids).

And: `<1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE`
Checks which value matches the oracle payload data (second value) leaving the winner `pubkeyHash` on the stack.

Finally: `OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`
Similar to the P2PKH sequence, but we use `OP_OVER` because top item stack will be public key hash.

The data of the input spending this output would be:

`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey>`, where `<oracleData>` is in the format `key:payload` and `<txSignature>` is the transaction signature, like regular P2PKH transactions.

Putting everything together, the execution would be:

Stack=`[]`  
Script=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey> OP_DUP OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKSIG_ORACLE OP_CHECKDATASIG <0> <key> OP_DATA_EQUAL_INT <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey>`  
Script=`OP_DUP OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_EQUAL_INT <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey> <oraclePubkey>`  
Script=`OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_EQUAL_INT <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey> <oraclePubkeyHash>`  
Script=`<oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_EQUAL_INT <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey> <oraclePubkeyHash> <oraclePubkeyHash>`  
Script=`OP_EQUALVERIFY OP_CHECKDATASIG <0> <key> OP_DATA_EQUAL_INT <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <oracleSig> <oraclePubKey>`  
Script=`OP_CHECKDATASIG <0> <key> OP_DATA_EQUAL_INT <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData>`  
Script=`<0> <key> OP_DATA_EQUAL_INT <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <0> <key>`  
Script=`OP_DATA_EQUAL_INT <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData>`  
Script=`<1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2> OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <oracleData> <1> <FallbackPubKeyHash> <0> <PubKeyHash_A> <1> <PubKeyHash_B> <2>`  
Script=`OP_DATA_MATCH_VALUE OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

(Suppose data says Brazil won - payload is 1)

Stack=`<txSignature> <pubkey> <PubKeyHash_B>`  
Script=`OP_OVER OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <PubKeyHash_B> <pubkey>`  
Script=`OP_HASH160 OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey> <PubKeyHash_B> <pubkeyHash>`  
Script=`OP_EQUALVERIFY OP_CHECKSIG`

Stack=`<txSignature> <pubkey>`  
Script=`OP_CHECKSIG`

Stack=`<TRUE>`  
Script=

# Drawbacks
[drawbacks]: #drawbacks

Contracts might be as complex as their designers want, which may lead to bugs in their implementation. Even though this would not be a bug in the Hathor core code, this would reflect badly in our platform.

The script verification also becomes a bit more complex, which means more CPU usage, but this should not be a problem.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
