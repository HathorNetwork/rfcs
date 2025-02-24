# Fee Custom Tokens RFC

- Feature Name: Fee custom tokens
- Start Date: 2025-02-24
- RFC PR: https://github.com/HathorNetwork/rfcs/pull/94
- Hathor Issue:
- Author: Raul Soares de Oliveira

## Summary

[summary]: #summary

Currently, when minting X tokens, a [deposit of P% (1%) of X in HTR is required](./0011-token-deposit.md). This deposit can later be withdrawn when the tokens are melted.

This proposal suggests an alternative mechanism where, instead of requiring an upfront deposit of HTR when minting new tokens, a transaction fee is charged on each transfer of the newly minted tokens.

For nano contracts, the fee is charged from the caller, whether it's a transaction or another nano contract. The charging mechanism follows the same pattern as deposit-based tokens: deposits are considered as outputs and withdrawals as inputs. For actions called from within a nano contract, fees are charged per action from the caller's balance. Fees are calculated per action and transfer, not as computational gas costs like in Ethereum.

## Motivation
[motivation]: #motivation

Dozer suggested an alternative to the HTR deposit requirement when minting tokens. The idea is to create a new type of custom token where tokens would be minted for free (i.e., no deposits) and fees would be charged for transactions. [RFC](https://github.com/Dozer-Protocol/hathor-rfcs/blob/new-token-economics/projects/new-token-economics/token-economics.md)

This change would reduce the upfront cost of minting tokens, making them more accessible to users who may not have sufficient HTR at the time of minting.

## Guide-level Explanation

When minting tokens, creators can choose between two transaction models: Deposit-Based and Fee-Based. Each model offers different benefits, depending on how the tokens will be used. To handle this, we will use the same definitions from both [Token Creation Transaction](https://github.com/HathorNetwork/rfcs/blob/master/text/0004-tokens.md) and this proposal.

### Deposit-Based Model (Current Implementation)

Currently, creators must [deposit a percentage](https://github.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md) (P%) of the minted tokens' value in HTR. For example, if P% is set at 1% and a user mints 100 tokens, they must deposit 1 HTR. This deposit acts as a reserve and is fully refundable if the tokens are later melted.

In nano contracts, the caller (whether a UTXO transaction or another nano contract) pays the deposit. For UTXO transactions, the deposit is charged from the imbalance between inputs and outputs when a token creation transaction occurs. In nano contracts, the deposit is charged from the contract's balance when it calls the syscall to create deposit-based tokens.

### Fee-Based Model

In the fee-based model, no upfront deposit is required — each transfer simply incurs a transaction fee. This gives token creators flexibility to tailor their strategy to specific use cases, such as minting large quantities of memecoins without tying them to HTR value. The model also supports other scenarios that benefit from predictable, low-cost transactions. For example, it enables the creation of scalable in-game currencies or loyalty tokens, where frequent, small transactions don't incur high fees, and projects can monetize their services more effectively by using custom tokens for fees.

#### Fee Header

A fee header will be added to transactions to signal which tokens will be used for fee payment.

In the example below we'll use Fee-based Token (FBT), and Deposit-based Token (DBT) as our tokens.

> **Note:** In the Guide-level examples, fees are shown as `(amount, token_index)` tuples for readability. The actual implementation uses `FeeHeaderEntry(token_index, amount)` dataclass - see [Reference-level Explanation](#fee-header-1) for details.

```
inputs: [100 DBT, 1000 FBT]
outputs: [1000 FBT]
tokens: [DBT, FBT]
fee_header:
    fees: [(100, 1)] ← 100 units, index 1 (DBT) in the tokens array
```

#### Fee Cost

Fees will be proportional to the number of outputs with fee-based tokens. For instance, if there are 3 HTR outputs, 2 outputs with deposit-based tokens, and another 5 with fee-based tokens, only the latter 5 will count towards the fee.

For melting operations that don't contain any outputs, we'll count them as 1 output in the fee calculation.

This proposal suggests **0.01 HTR per output**.

Apart from accepting HTR for fee payment, any deposit-based token will be accepted. In this case, since the token was created with a 100:1 ratio of HTR ([deposit model](#deposit-based-model-current-implementation)), the fee needs to be 100x the HTR rate. That means **0.01 HTR or 1.00 deposit-based-token**.

##### Melting Operations

Melting tokens without authority is forbidden. Fee payments will be accepted as long as their total is exactly equal to the required fee. Any other case will be rejected.

In the examples below we'll use Fee-based Token (FBT), and Deposit-based Token (DBT) as our tokens.

In this case the fee is 1 HTR or 100 DBT, but the user is providing 150 DBT as payment, which will cause this transaction to be rejected.

```
inputs: [FBT MELT AUTHORITY, 100 FBT, 200 DBT]
outputs: [100 FBT, 50 DBT]
tokens: [DBT, FBT]
fee_header: 
    fees: [(150, 1)] ← index of DBT in the tokens array
```

**Example 2: Valid fee payment**
For instance, if there is a transaction with:

```
inputs: [1000 FBT, FBT Melt Authority, 500 DBT]
outputs: [500 FBT, 400 DBT]
tokens: [DBT, FBT]
fee_header: 
    fees: [(100, 1)] ← index of DBT in the tokens array
```

In this scenario, 100 DBT is used to pay the fee without requiring any authority.

**Example 3: Combined fee payment and token melting**
For a combination of paying the fee and also melting the token, we'll have the following:

```
inputs: [1000 FBT, FBT Melt Authority, 500 DBT, DBT Melt Authority]
outputs: [500 FBT, FBT Melt Authority, 300 DBT, DBT Melt Authority]
tokens: [DBT, FBT]
fee_header: 
    fees: [(100, 1)] ← index of DBT in the tokens array
```

Here, 100 DBT is used to pay the fee, and there is a melt of 500 FBT and 100 DBT. For melting tokens, the behavior remains the same, requiring authority.

##### Nano Contracts

It's important to mention that this fee should behave similarly to **deposit-based tokens and not like gas fees in Ethereum**. The fee is charged per action and syscall, following the same pattern already established for nano contracts with deposit-based tokens, rather than being a computational resource cost like gas.

We'll support fee payments with deposit tokens that will be created in the same nano contract execution.

Based on this [issue](https://github.com/HathorNetwork/rfcs/issues/97), the items below will be charged **0.01 HTR** when dealing with Fee-based tokens:

**Syscalls:**
- `create_fee_token`
- `mint_tokens`
- `melt_tokens`

**Actions:**
- Deposit
- Withdraw

For transactions that are calling a Deposit/Withdraw action, they will be charged both from the outputs count and by the action cost. Nano contracts transferring funds between each other will pay fees based on the actions they might call.

**Fee calculation formulas:**

Transaction depositing funds in a nano contract:
```
0.01 * len(outputs) + len(deposits)
```

If the same fee-based token is in the inputs but not in the outputs, we'll consider the same fee used for melting operations: 0.01 HTR (check [Fee cost](#fee-cost)).

Transaction withdrawing funds from a nano contract:
```
0.01 * len(outputs) + len(withdraws)
```

Nano contracts transferring funds within a nano contract:
```
0.01 * (len(deposits) + len(withdraws))
```

**Practical Example:**

To demonstrate the above with an example we will consider:
- A nano contract (nc1) that is already initialized
- The initial balance of the nc1 is 0.00 HTR
- The nc1 doesn't call any other contract

```python
class MyBlueprint(Blueprint):
    @public(allow_deposit=True, allow_withdrawal=True)
    def create_tokens(self, ctx: Context):
        fbt = self.syscall.create_fee_token(5000, 'FBT')
    
    @public
    def no_op(self):
        pass
```

###### Creating a Fee-based token (FBT) - UTXO → Nano contract

```
inputs: [200.01 HTR]
outputs: [5000.00 FBT, 100.00 HTR]
nano_header:
    nc_id: nc1,
    action: deposit (100.00 HTR), withdraw (5000.00 FBT),
    method: create_tokens
fee_header: 
    fees: [(0.01, 0)] ← HTR index in the tokens array
```

In this case, the caller is the transaction, so the fee will be charged from it in the form of an output.

**Fee breakdown:**
- 0.01 HTR from the outputs count, provided by the withdraw action
- 0.01 HTR from the `create_fee_token` syscall ← charged from the contract's balance

**Contract State:**
- balance:
    - HTR: 99.99
    - FBT: 0.00

Although HTR was used in the example, deposit-based tokens are also accepted.

###### Deposit a Fee-based token (FBT) - UTXO → Nano contract

In this example, we'll deposit part of the fee-based tokens created into a contract.

```
inputs: [5000.00 FBT, 100.00 HTR]
outputs: [4000.00 FBT, 99.98 HTR]
nano_header:
    nc_id: nc1,
    action: deposit (1000.00 FBT),
    method: no_op
fee_header: 
    fees: [(0.02, 0)] ← HTR index in the tokens array
```

**Fee breakdown:**
- 0.01 HTR from the outputs count
- 0.01 HTR from the `deposit` action

**Contract State:**
- balance:
    - HTR: 99.99
    - FBT: 1000.00

###### Creating a Fee-based token (FBT) - Nano contract → Nano contract

This is a pseudo code example:

```python
class BlueprintA(Blueprint):
    def initialize(self, ctx: Context, token_creator_nc_id: ContractId) -> None: 
        self.token_creator_nc_id = token_creator_nc_id
    
    @public(allow_deposit=True, allow_withdrawal=True)
    def create_tokens(self, ctx: Context):
        ctx = {
            'actions': [Withdraw(5000, FBT), Deposit(0.01, HTR)],
            'fees': [(0.01, HTR)]
        }
        fbt = self.syscall.call_public_method(self.token_creator_nc_id, 'create_fee_tokens')
    
    @public
    def no_op(self):
        pass
```

```python
class TokenCreator(Blueprint):
    @public(allow_deposit=True, allow_withdrawal=True)
    def create_fee_tokens(self, ctx: Context):
        fbt = self.syscall.create_fee_token(5000, 'FBT')
    
    @public
    def no_op(self):
        pass
```

In this example, we'll call contract "nc_1" which will call "token_creator_nc_1" that is responsible for creating the tokens.

```
inputs: [0.02 HTR]
outputs: []
nano_header:
    nc_id: nc1, # using BlueprintA
    action: deposit (0.02 HTR),
    method: create_tokens
```

After executing the `create_tokens` method from `nc_1` (using BlueprintA), we'll have the following balances:

**nc_1:**
- HTR: 0.00
- FBT: 5000.00
- fees: 0.01 HTR from the withdraw action

**token_creator_nc1:**
- HTR: 0.00
- FBT: 0.00
- fees: 0.01 HTR from the create fee tokens syscall

With this example we can see that each action (DEPOSIT or WITHDRAW) using fee tokens charges 0.01 HTR.

#### Fee Destination

The fees will be burned.

### Affected Projects

The implementation in the following projects does not support fee payments with custom deposit-based tokens initially, although it will be available at the protocol level.

#### Wallet-lib

This project handles methods that are responsible for preparing a transaction and choosing UTXOs that will serve as inputs. With the Fee-based token version, those methods need to be refactored to handle it and prepare the UTXOs according to the requirements described in this document.

#### Wallet-service

The wallet service has its own database with the token details data stored. It will be necessary to add the token `version` field introduced by this document, since before it wasn't used and was hardcoded.

#### Explorer

The Explorer service ingestors are prepared to bring all columns from the wallet-service database into Elasticsearch, where they will be used by the Explorer service to provide data for the token detail API. We will update the token list view, token detail view, and transaction detail view by adding fee data and token version descriptions.

#### Wallet-headless

This is the official headless wallet of Hathor. We'll need to update the create tokens methods (both UTXO and nano) arguments to support the new token version.

#### RPC-lib

The RPC lib acts as a bridge between the wallet-lib and the desktop and mobile clients. It's required to update the create tokens handlers (UTXO and nano) to accept the token version argument.

#### Wallet (Desktop, Mobile)

The wallets will have a new create token experience. It will allow the user to choose between the token versions (deposit or fee based), see the fees and check the token detail data.

Also, we'll provide information about the transaction fee while approving a transaction and before sending it. The user will be able to check the paid fee in the transaction detail screen, similar to explorer.

## Reference-level Explanation

In this section, technical details are expanded for what was described above.

### Changes in the Transaction Anatomy

#### Token Info Version (aka Token Version)

Given Hathor's [Anatomy of a Transaction RFC](https://github.com/HathorNetwork/rfcs/blob/master/text/0015-anatomy-of-tx.md#token-creation), it is reasonable to suggest that the byte used by the token creation transaction `token_info_version` will be used to determine fee-based tokens.

Since each custom token `id` is the hash of the token creation transaction that created it, we can assume the enum values below can be assigned to the `token_info_version` byte in the token creation tx and then we can retrieve it. We'll discuss token creation in nano contracts in a later section.

So, by adding a TokenVersion enum we have:

- `NATIVE = 0` (internal)
- `DEPOSIT = 1` (current implementation)
- `FEE = 2` (new)

Then, we must allow the versions above in the `deserialize_token_info` method by checking the enum values.

#### Fee Header

To correctly validate a transaction, we need to know the version of the tokens we're transacting with (which wasn't necessary before) and which token and what amount of that token the user wants to pay the fee with.

The fee header comes to resolve the explicit intention of how the transaction wants to pay the fee.

##### Data Structures

```python
@dataclass(slots=True, kw_only=True, frozen=True)
class FeeHeaderEntry:
    token_index: int  # index in tx.tokens array
    amount: int       # fee amount in smallest unit

@dataclass(slots=True, kw_only=True)
class FeeHeader(VertexBaseHeader):
    tx: Transaction
    fees: list[FeeHeaderEntry]
    settings: HathorSettings
```

##### Serialization

```
[header_id: 1 byte][fees_len: 1 byte][fee_entries...]
  fee_entry: [token_index: 1 byte][amount: variable (output_value encoding)]
```

The maximum number of fee entries is 16 (limited by 1 byte for length).

##### Example

```
Transaction with fee-based tokens:
input: [1.00 dbt, 100.00 fbt]
tokens: [dbt, fbt]
fee_header:
    fees: [FeeHeaderEntry(token_index=1, amount=1)] ← index of the deposit token
```

### Transaction Class

Inside the transaction class we can obtain a dictionary `token_dict: dict[TokenUid, TokenInfo]` which is generated by the `get_complete_token_info()` method. This method iterates over the inputs and the outputs consolidating the amount, and checking the authorities of each token.

To obtain the expected result, we'll consider fees as outputs.

```
inputs: [1_000 FBT, 1_00 DBT]
outputs: [1_000 FBT]
tokens: [DBT, FBT]
fee_header:
    fees: [FeeHeaderEntry(token_index=1, amount=100)]  ← DBT at index 1
```

In this case we'll have the following equation:

```
sum(inputs) = sum(outputs)
(1000 - 1000)FBT = (100 - 100)DBT
0 = 0 ← Valid
```

#### Calculate Fee Method

This method will be called during block confirmation after nano execution, as a guarantee that we'll have access to all tokens and their respective versions, both in tx_storage and in nc_block_storage. It will be used to confirm if the fee that the transaction specified is correct; otherwise, the transaction will be rejected.

##### Data Structures

The `TokenInfo` dataclass holds per-token information including fee-related counters:

```python
@dataclass(slots=True, kw_only=True)
class TokenInfo:
    version: TokenVersion | None
    amount: int = 0
    can_mint: bool = False
    can_melt: bool = False
    chargeable_outputs: int = 0  # count of non-authority outputs
    chargeable_inputs: int = 0   # count of non-authority inputs
```

The `TokenInfoDict` class extends `dict[TokenUid, TokenInfo]` and provides fee calculation:

```python
class TokenInfoDict(dict[TokenUid, TokenInfo]):
    fees_from_fee_header: int = 0
```

Since the `get_complete_token_info()` method iterates over the inputs and outputs, the `chargeable_outputs` and `chargeable_inputs` counters in each `TokenInfo` are incremented during this iteration, saving resources by avoiding the need to retrieve the information elsewhere.

With the counters in hand we just need to add a method to calculate the fee. Below is the method implementation:

```python
def calculate_fee(self, settings: 'HathorSettings') -> int:
    fee = 0

    for token_uid, token_info in self.items():
        if token_info.chargeable_outputs > 0:
            fee += token_info.chargeable_outputs * settings.FEE_PER_OUTPUT
        elif token_info.chargeable_inputs > 0:
            fee += settings.FEE_PER_OUTPUT

    return fee
```

As mentioned in the [fee cost](#fee-cost) section:

> "For melting operations that don't contain any outputs, we'll count them as 1 output in the fee calculation."

### Transaction Verifier

#### verify_sum Method

Previously, only token creation transactions could create tokens and these had only one version, so it wasn't necessary to search for the token creation transaction that created that token to validate its version.

The problem arises when we need to query the token version to validate fee payment. With the advent of nano contracts, we don't depend only on the UTXO universe and token creation transactions, but also on the nano universe.

Tokens in nano contracts have deterministic IDs and can be passed to contracts and transactions normally; however, we don't know their true versions until the end of their execution in the block.

The `verify_sum` is executed during the verification phase. In the verification phase, we cannot guarantee which block is correct for querying the nc_block_storage.

To resolve these issues, we'll call `verify_sum` in two different phases: one during the verification phase and another after nano execution when the block confirms the transactions.

##### verify_sum_without_storage Method

This method will be executed during the verification phase and will rely on the `token_dict` already aggregated with the `fee_header`. In this step, we'll maintain behavior similar to the current `verify_sum`, analyzing what's possible between unauthorized mint/melt attempts and invalid quantities.

##### verify_sum Method

This method will be executed after nano execution in a block. To fully validate the transaction, we need both `tx_storage` and `nc_block_storage`. We'll receive the block ID to access the `nc_block_storage`.

With both storages in hand, we can generate a complete `token_dict` with the versions of each token, and consequently calculate the fee using `calculate_fee`.

In this validation, we'll check if the fee value calculated using the token versions matches those provided by the transaction. If equal, the validation passes; otherwise, we reject the transaction.

```
sum(fee_header) = calculated_fee
```

As mentioned in the guide-level explanation:

> Fee payments will be accepted as long as their total is exactly equal to the required fee. Any other case will be rejected.

##### Examples

Tokens: Deposit-Based Token (DBT), and Fee-Based Token (FBT).

**Example 1: Transferring a fee token and paying with HTR**

```
inputs: [
    0.02 HTR,
    1000.00 FBT
]
outputs: [
    0.01 HTR,
    1000.00 FBT,
]
tokens: [HTR, DBT, FBT] ← HTR(0), DBT(1), FBT(2)
fee_header: 
    fees: [
        (0.01, 0) ← HTR
    ]
```

**Verifications:**
- sum(fee_header): 0.01 HTR
- calculated_fee: 0.01 HTR

**Example 2: Transferring a fee token and paying with deposit token and HTR**

```
inputs: [
    0.01 HTR,
    1.00 DBT,
    1000.00 FBT
]
outputs: [
    500.00 FBT,
    500.00 FBT,
]
tokens: [HTR, DBT, FBT] ← HTR(0), DBT(1), FBT(2)
fee_header: 
    fees: [
        (0.01, 0), ← HTR
        (1.00, 1), ← DBT
    ]
```

**Verifications:**
- sum(fee_header): 0.02 HTR
- calculated_fee: 0.02 HTR

### Fee Payment with Deposit-Based Tokens

When paying fees with deposit-based custom tokens instead of HTR, the conversion uses the `TOKEN_DEPOSIT_PERCENTAGE` setting (default: 0.01 = 1%):

```python
FEE_DIVISOR = int(1 / TOKEN_DEPOSIT_PERCENTAGE)  # = 100
```

The `total_fee_amount` method in `FeeHeader` converts all fee entries to their HTR equivalent:

```python
def total_fee_amount(self) -> int:
    """Sum fees amounts in this header and return as HTR equivalent."""
    total_fee = 0
    for fee in self.get_fees():
        if fee.token_uid == self.settings.HATHOR_TOKEN_UID:
            total_fee += fee.amount
        else:
            # Deposit token: use withdraw formula (floor)
            total_fee += floor(TOKEN_DEPOSIT_PERCENTAGE * fee.amount)
    return total_fee
```

**Example:**
- Fee required: 1 HTR unit (0.01 HTR)
- Paying with HTR: amount = 1
- Paying with deposit token: amount = 100 (since `floor(0.01 * 100) = 1`)

### Fee Amount Validation

Before accepting a fee payment, the following validations are performed:

```python
def validate_fee_amount(settings: HathorSettings, token_uid: TokenUid, amount: int) -> None:
    # Rule 1: Amount must be positive
    if amount <= 0:
        raise InvalidFeeAmount('fees should be a positive integer')

    # Rule 2: Deposit tokens must use amounts that are multiples of FEE_DIVISOR
    if token_uid != settings.HATHOR_TOKEN_UID and amount % settings.FEE_DIVISOR != 0:
        raise InvalidFeeAmount(
            f'fees using deposit custom tokens should be a multiple of {settings.FEE_DIVISOR}'
        )
```

**Rationale:** Since deposit tokens are converted to HTR using `floor(TOKEN_DEPOSIT_PERCENTAGE * amount)`, amounts that are not multiples of `FEE_DIVISOR` (100) would result in precision loss. By requiring exact multiples, we ensure deterministic fee calculations.

**Examples:**
- Valid amounts: 100, 200, 300... (multiples of 100)
- Invalid amounts: 50, 150, 99... (would lose precision in conversion)

### Feature Flag in Settings

For development purposes, this feature will be feature-flagged to run only on the local network by setting the `ENABLE_FEE_TOKENS` in settings to true.

### Feature Activation

For production, we'll rely on feature activation to release this feature.

### Transaction Resource

We should provide a field with the transaction fee and the token version in the detailed response.

## Drawbacks

A drawback would be an increase in CPU consumption because the verification algorithm will be more complicated, but it seems to be a minor drawback.

## Prior Art

### How Bitcoin Deals with Fees

Bitcoin handles transaction fees through a system based on competition for limited block space. Each transaction has a weight measured in vBytes, and users set their fees by offering satoshis per vByte, which influences the confirmation speed. During periods of high demand, fees increase due to competition, while in lower traffic periods, they decrease.

Wallets and services use fee estimators to suggest optimal fees based on the mempool. The protocol does not enforce a minimum fee, but nodes can set a "relay fee" to prevent spam from very low-value transactions.

### How Ethereum Deals with Fees

Ethereum handles transaction fees through the concept of Gas, where each operation consumes a specific amount of computational resources. Before EIP-1559, users set the Gas Price, and those who paid more had priority. With EIP-1559, a dynamically adjusted Base Fee was introduced based on demand, along with an optional Priority Tip to incentivize validators.

The Base Fee is burned, reducing the Ether supply, while the Priority Tip goes to validators. This model improves fee predictability, making costs more stable and transparent for users.

# Business unresolved questions

- ~~Should we allow deposit-based tokens to collect fees?~~
- ~~Should we burn or move it to a burn address?~~
- ~~How will the fee be calculated?~~
- ~~Should we only consider the outputs in the fee calculation?~~

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- ~~How should melt operations be handled for fee-based custom tokens?
- ~~How will fee adjustments be governed?
- How transactions without a deeper verification will behave in the long term, since we are splitting the verify_sum in two steps?

## Future Possibilities
[future-possibilities]: #future-possibilities

- Custom Fee Markets (letting users optimize fees)

## References

- https://github.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md
- https://github.com/HathorNetwork/rfcs/blob/master/text/0004-tokens.md
- https://github.com/HathorNetwork/rfcs/blob/master/text/0015-anatomy-of-tx.md
- https://bitcoin.org/bitcoin.pdf
- https://eips.ethereum.org/EIPS/eip-1559
- https://ethereum.org/en/developers/docs/gas/
- https://developer.bitcoin.org/devguide/transactions.html#transaction-fees-and-change
- https://beta-test.docs.hathor.network/explanations/features/nano-contracts/
- https://beta-test.docs.hathor.network/explanations/features/nano-contracts/how-it-works/