- Feature Name: transaction_fee_minting
- Start Date: 2025-02-24
- RFC PR: (to be created)
- Hathor Issue: (leave this empty)
- Author: Raul Soares de Oliveira

# Summary
[summary]: #summary
Currently, when minting X tokens, a [deposit of P% (1%) of X in HTR is required](./0011-token-deposit.md). This deposit can later be withdrawn when the tokens are melted. 

This proposal suggests an alternative mechanism where, instead of requiring an upfront deposit of HTR when minting new tokens, a transaction fee is charged on each transfer of the newly minted tokens.

# Motivation
[motivation]: #motivation

Dozer suggested an alternative to the HTR deposit requirement when minting tokens. The idea is to create a new type of custom tokens where tokens would be minted for free (i.e., no deposits) and fees will be charged for transactions. [RFC](https://github.com/Dozer-Protocol/hathor-rfcs/blob/new-token-economics/projects/new-token-economics/token-economics.md)

This change would reduce the upfront cost of minting tokens, making it more accessible to users who may not have sufficient HTR at the time of minting. The network would still benefit from a fee mechanism that contributes to miners’ incentives and overall network security.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When minting tokens, creators can choose between two transaction models: Deposit-Based and Fee-Based. Each model offers different benefits, depending on how the tokens will be used. For handle it we will use the same definitions of the [Token Creation Transaction](https://github.com/HathorNetwork/rfcs/blob/master/text/0004-tokens.md), possibly adding a variation of it using a new version.

## Deposit-Based Model (as is)

Currently, creators must [deposit a percentage](https://github.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md) (P%) of the minted tokens’ value in HTR. For example, if P% is set at 1% and a user mints 100 tokens, they must deposit 1 HTR. This deposit acts as a reserve and is fully refundable if the tokens are later melted.

## Fee-Based Model

Adding the fee-based model the platform won't require an upfront deposit. Instead, each transfer of the minted tokens incurs a transaction fee, which is deducted automatically. The exact fee rate will be defined at a later stage.

By selecting the appropriate model, token creators can optimize their minting strategy based on their specific needs and usage scenarios. A key use case is the creation of memecoins.

### How to approach

An alternative to Bitcoin and Ethereum approchaes for blockchain fees is a hybrid model that combines fees based in the transaction size, with priority tip payments.

This model offers several advantages. First, it simplifies the user experience by providing a predictable fee without major fluctuations. Additionally, it enables a filtering mechanism to prevent excessively large or complex transactions, protecting the network from spam. Finally, its structure promotes sustainability, ensuring controlled blockchain growth.

### Minning
By adding fees to the transactions, we don't need to mine them anymore. So the proof of work (PoW) won't affect the fee calculation.

### Fee calculation

Hathor’s fees will be calculated similarly to Bitcoin, depending only on transaction size (in bytes) and a network-defined fee rate (in HTR per byte).

Proof-of-Work difficulty (transaction weight) will not influence the fee calculation.

Let's assume:
1. HTR is always used as the fee unit, regardless of the tokens involved in the transaction.
2. Even if fee-based tokens are used alone, or with a deposit-based token, a single fee in HTR will be charged, regarding the transaction size.
3. Interacting with nano contracts may introduce an additional fee depending on the execution complexity.

Below is a pseudo code that explain how fee should be calculated, using a fee rate per byte:

*(Note: Excluded "hash" size from calculation, hence 114 reduced to 84 bytes.)*

```python
def calculate_tx_fee(tx, fee_rate_per_byte):
    # Fixed size fields (always present)
    fixed_size = 84  # version (2) + timestamp (4) + nonce (4) + weight (8) + parents (64)

    # Tokens array size
    tokens_size = 32 * len(tx.tokens)

    # Inputs size calculation
    inputs_size = 0
    for input in tx.inputs:
        inputs_size += 35 + len(input.data)  
        # 32 bytes (tx_id) + 1 byte (index) + 2 bytes (data length prefix) + len(data)

    # Outputs size calculation
    outputs_size = 0
    for output in tx.outputs:
        outputs_size += 7 + len(output.script)
        # 4 bytes (value) + 1 byte (token_data) + 2 bytes (script length prefix) + len(script)

    # Total transaction size
    tx_size = fixed_size + tokens_size + inputs_size + outputs_size

    # Fee calculation
    fee = tx_size * fee_rate_per_byte

    return fee + tx.priority_tip # plus any nano fee that should match

```


### **Example Calculation:**

**Hypothetical Transaction:**
- 1 token UID
- 2 inputs (with 108 bytes of data each, common signature size)
- 2 outputs (with 25 bytes of script each, common P2PKH script)

Calculate size:

| Field                | Calculation                 | Size     |
|----------------------|-----------------------------|----------|
| Fixed-size fields    |                             | **84** bytes |
| Tokens               | 1 token × 32 bytes          | **32** bytes |
| Inputs               | 2 × (35 + 108)              | **286** bytes |
| Outputs              | 2 × (7 + 25)                | **64** bytes |
| **Total Tx Size**    | 84 + 32 + 286 + 64          | **466** bytes |

## Wallet UI

We should provide to the users the option to select between the two models (deposit and fee) when minting.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
TODO

# Drawbacks of the Hybrid Fee Model
[drawbacks]: #drawbacks

While the hybrid fee model balances incentives and network sustainability, it also introduces potential drawbacks that must be considered. Below are key concerns and challenges:

### Complexity in Multi-Token Transactions

**Issue:** Transactions involving multiple tokens introduce a more complex fee structure, requiring additional computation to determine the appropriate charge.

**Impact:**
  - Increased processing overhead for wallets.
  - Potential user confusion when sending transactions with different token types.

**Potential Mitigation:**
  - Improve **wallet UX** to clearly display estimated fees before confirming transactions.

# Prior art

### How Bitcoin deals with fee

Bitcoin handles transaction fees through a system based on competition for limited block space. Each transaction has a weight measured in vBytes, and users set their fees by offering satoshis per vByte, which influences the confirmation speed. During periods of high demand, fees increase due to competition, while in lower traffic periods, they decrease. 

Wallets and services use fee estimators to suggest optimal fees based on the mempool. The protocol does not enforce a minimum fee, but nodes can set a "relay fee" to prevent spam from very low-value transactions.

### How Ethereum deals with fee

Ethereum handles transaction fees through the concept of Gas, where each operation consumes a specific amount of computational resources. Before EIP-1559, users set the Gas Price, and those who paid more had priority. With EIP-1559, a dynamically adjusted Base Fee was introduced based on demand, along with an optional Priority Tip to incentivize validators. 

The Base Fee is burned, reducing the Ether supply, while the Priority Tip goes to validators. This model improves fee predictability, making costs more stable and transparent for users.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What will happen to the fees? Will they be collected by the miners?
- How should melt operations be handled for fee-based custom tokens?
- Should storage fees be one-time or recurring (rent model)?
- How fee adjustments will be governed?
- Removing the transaction fee from minning, how will it affect the tx-mining-service?

# Future possibilities
[future-possibilities]: #future-possibilities

- Custom Fee Markets (letting users optimize fees).
- Dynamic Fee Adjustment Mechanism, where fees scale based on network congestion.

# References

- https://github.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md
- https://github.com/HathorNetwork/rfcs/blob/master/text/0004-tokens.md
- https://github.com/HathorNetwork/rfcs/blob/master/text/0015-anatomy-of-tx.md
- https://bitcoin.org/bitcoin.pdf
- https://eips.ethereum.org/EIPS/eip-1559
- https://ethereum.org/en/developers/docs/gas/
- https://developer.bitcoin.org/devguide/transactions.html#transaction-fees-and-change

