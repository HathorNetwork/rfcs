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

Dozer suggested an alternative to the HTR deposit requirement when minting tokens. The idea is to remove the 1% deposit requirement and, instead, implement a transaction fee for every transfer of the minted tokens. [RFC](https://github.com/Dozer-Protocol/hathor-rfcs/blob/new-token-economics/projects/new-token-economics/token-economics.md)

This change would reduce the upfront cost of minting tokens, making it more accessible to users who may not have sufficient HTR at the time of minting. The network would still benefit from a fee mechanism that contributes to miners’ incentives and overall network security.

An explicit use case is meme coins creation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When minting tokens, creators can choose between two transaction models: Deposit-Based and Fee-Based. Each model offers different benefits, depending on how the tokens will be used.

## Deposit-Based Model

In this [model](https://github.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md), creators must deposit a percentage (P%) of the minted tokens’ value in HTR. For example, if P% is set at 1% and a user mints 100 tokens, they must deposit 1 HTR. This deposit acts as a reserve and is fully refundable if the tokens are later melted.

## Fee-Based Model

The fee-based model does not require an upfront deposit. Instead, each transfer of the minted tokens incurs a transaction fee, which is deducted automatically. The exact fee rate will be defined at a later stage.

By selecting the appropriate model, token creators can optimize their minting strategy based on their specific needs and usage scenarios. A key use case is the creation of memecoins.

### How Bitcoin deals with fee

Bitcoin handles transaction fees through a system based on competition for limited block space. Each transaction has a weight measured in vBytes, and users set their fees by offering satoshis per vByte, which influences the confirmation speed. During periods of high demand, fees increase due to competition, while in lower traffic periods, they decrease. 

Wallets and services use fee estimators to suggest optimal fees based on the mempool. The protocol does not enforce a minimum fee, but nodes can set a "relay fee" to prevent spam from very low-value transactions.

### How Etherium deals with fee

Ethereum handles transaction fees through the concept of Gas, where each operation consumes a specific amount of computational resources. Before EIP-1559, users set the Gas Price, and those who paid more had priority. With EIP-1559, a dynamically adjusted Base Fee was introduced based on demand, along with an optional Priority Tip to incentivize validators. 

The Base Fee is burned, reducing the Ether supply, while the Priority Tip goes to validators. This model improves fee predictability, making costs more stable and transparent for users.

### An hybrid approach

A third option for blockchain fees is a hybrid model that combines minimum fees with payments based on actual resource usage, along with policies to prevent spam on the network. This model aims to balance user simplicity, protection against spam attacks, and long-term sustainability.

The adaptive minimum fee consists of a fixed base fee that all users pay per transaction, ensuring a minimum cost for network usage and preventing abuse. Additionally, there is a variable component that adjusts the fee based on resource consumption, such as transaction size or script complexity.

To address the spam problem, some networks have tested models with proof of work (PoW) embedded in transactions. This mechanism requires users to perform a small computational task before sending transactions, making mass spam transactions more difficult. Another approach is the use of dynamically adjusted microfees to maintain network balance.

This model offers several advantages. First, it simplifies the user experience by providing a predictable fee without major fluctuations. Additionally, it enables a filtering mechanism to prevent excessively large or complex transactions, protecting the network from spam. Finally, its structure promotes sustainability, ensuring controlled blockchain growth.

### General Fee calculation rule

1. HTR is always used as the fee unit, regardless of the tokens involved in the transaction.
2. If only fee-based tokens are used, a single fee in HTR will be charged, regardless of the number of tokens.
3. If a transaction includes both deposit-based and fee-based tokens, only the fee-based tokens will incur a fee.
4. Interacting with nano contracts may introduce an additional fee depending on the execution complexity.

### Example Scenarios

####  Token A x Token A (fee in HTR)

Description: Transfer of the same token (A) between two parties.


Fee Calculation:

- Since Token A is fee-based, the transaction requires a fee in HTR.
```
- Base Fee: 0.001 HTR
- Total Fee: 0.001 HTR (regardless of the amount transferred)
```
----
#### Token A x Token B x Token C (only 1 fee, in HTR)

Description: Multi-token transfer involving three fee-based tokens.

Fee Calculation:
- Only one HTR fee is charged, regardless of the number of tokens.
```
- Base Fee: 0.001 HTR
- Complexity Fee (extra tokens): 0.0005 HTR per additional token.
- Total Fee: 0.002 HTR (0.001 + 0.0005 + 0.0005)
```

#### Token A x HTR (fee in HTR)

Description: Transfer of a fee-based token and HTR in the same transaction.

Fee Calculation:
- Since a fee-based token is involved, an HTR fee will be charged.
```
- Base Fee: 0.001 HTR
- Total Fee: 0.001 HTR
```

#### Token D (deposit-based) x Token B (fee-based)
Description: Transfer of a deposit-based token (D) and a fee-based token (B).

Fee Calculation:
- The deposit-based token (D) does not pay a fee.
- The fee-based token (B) pays the standard fee.

```
- Base Fee: 0.001 HTR
- Total Fee: 0.001 HTR (no extra charge for Token D)
```

#### Transaction Interacting with a Nano Contract
Description: A transfer involving a nano contract, which may require additional execution on the network.

Fee Calculation:
- The standard transaction base fee still applies.
- Since nano contract execution involves additional computation, an extra fee may be charged based on complexity.
- Execution Fee (depends on contract complexity): Example, 0.002 HTR.

```	
- Base Fee: 0.001 HTR
- Total Fee: 0.003 HTR (0.001 base + 0.002 contract execution)
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
TODO

# Drawbacks of the Hybrid Fee Model
[drawbacks]: #drawbacks

While the hybrid fee model balances incentives and network sustainability, it also introduces potential drawbacks that must be considered. Below are key concerns and challenges:

### 1. Token Inflation and Supply Growth

**Issue:** Since there is no burning mechanism tied to fee-based token transfers, the total supply of tokens may continuously increase without a natural deflationary counterbalance.

**Impact:** Over time, token supply inflation can reduce the value of individual tokens, especially if demand does not grow proportionally.

**Potential Mitigation:**
  - Introduce a **burn mechanism** where a percentage of the transaction fee is permanently removed from circulation.
  - Implement an **optional melt requirement** where token issuers must periodically remove supply from circulation.

---

### 2. Potential for Low-Cost Network Congestion

**Issue:** If fees are too low, bad actors could create a high volume of low-value transactions, leading to congestion.

**Impact:**
  - Increased load on network validators and miners.
  - Slower transaction processing times during peak usage.

**Potential Mitigation:**
  - Introduce **dynamic fee adjustments** where fees scale based on network congestion.
  - Set **minimum transaction fees** to prevent microtransactions from overloading the system.

---

### 3. Complexity in Multi-Token Transactions

**Issue:** Transactions involving multiple tokens introduce a more complex fee structure, requiring additional computation to determine the appropriate charge.

**Impact:**
  - Increased processing overhead for nodes.
  - Potential user confusion when sending transactions with different token types.

**Potential Mitigation:**
  - Improve **wallet UX** to clearly display estimated fees before confirming transactions.
  - Optimize **fee calculation logic** to minimize the processing impact.

---

### 4. Miner Incentive Model May Need Adjustments

**Issue:** Since miners are currently incentivized through TX mining and HTR block rewards, a shift to fee-based transactions could impact their revenue model.

**Impact:**
  - If transaction volume is low, fee revenue might not be sufficient to replace TX mining incentives.
  - Long-term miner participation may decline if fees do not adequately compensate for network maintenance.

**Potential Mitigation:**
  - Transition **gradually** from TX mining to a full fee-based model.
  - Allow **fee-burning mechanisms** to ensure that fees remain valuable to miners.

---

### 5. Interactions with Smart Contracts and Additional Execution Costs

**Issue:** Nano contracts and other execution-based transactions introduce **variable computational costs** that are not reflected in a simple per-transaction fee model.

**Impact:**
  - Fees may be **too low to cover execution costs**, leading to **free rider issues**.
  - Overuse of nano contracts for cheap computation could **strain network resources**.

**Potential Mitigation:**
  - Charge **additional execution fees** based on contract complexity.
  - Implement **gas-like metering** for smart contract interactions.



# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How should melt operations be handled for fee-based custom tokens?
- Should storage fees be one-time or recurring (rent model)?
- Should there be discounts for batch transactions?
- Should fee adjustments be governed by community proposals?
- Should TX mining remain if transaction fees are introduced?

# Future possibilities
[future-possibilities]: #future-possibilities

- L2 Solutions (e.g., batch-processing for lower fees).
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

