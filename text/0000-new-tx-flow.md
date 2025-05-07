### Anatomy of a transaction

Each transaction has two set of fields: (i) the DAG fields and (ii) the Funds fields.

```
 ------------------------------------- 
| Transaction
 ------------------------------------- 
| DAG Fields (unsigned):
| - parents: list[VertexId]
| - weight: float[4 bytes]
| - nonce: integer[16 bytes]
| - timestamp: integer[4 bytes]
|
| Fund fields (signed):
| - version: integer
| - signal bits: integer
| - inputs: list[TxInput]
| - outputs: list[TxOutput]
| - tokens: list[VertexId]
 ------------------------------------- 
```

In the blockchain network, all transactions undergo a comprehensive verification process that includes the verification of each input's digital signature. The digital signature serves two crucial purposes: (i) ensuring that only the token owner can transfer their tokens, and (ii) providing immutability to the new transaction's fund fields. Immutability is vital as parties creating a transaction must verify the correctness of origins (inputs) and destinations (outputs) before signing. This safeguards against unauthorized changes to the destination without access to the owner's private key.

Notice that only the Funds fields are signed.


### Steps

1. Prepare an unsigned partial transaction, where all DAG fields are empty.
2. Sign all inputs, generating a signed partial transaction.
3. Fill out the DAG fields (parents, weight, nonce, and timestamp).
4. Broadcast to network.

It's crucial to emphasize that, after step 2, the fund fields cannot be modified anymore, which means that the inputs and outputs cannot be modified in any way. For clarity, no inputs or outputs can be added, removed, or changed. Any change would invalidate the digital signature verification.

#### Code Example

```js
# 1. Prepare an unsigned partial transaction, where all DAG fields are empty.
unsigned_partial_tx = ...

# 2. Sign all inputs, generating a signed partial transaction.
signed_partial_tx = sign_inputs(unsigned_partial_tx)

# 3. Fill out the DAG fields (parents, weight, nonce, and timestamp).
signed_tx = SendTransaction(..., {push_tx: false})

# 4. Broadcast to network.
push_tx(signed_tx)
```

### Q&A

1) What is the nonce field?

In the Hathor Network, transaction must be mined. In other words, each transaction has a mining difficulty (`weight`) and the `nonce` field must be filled out to solve the transaction.

Hathor Labs offers a free service, called tx-mining-service, that handles all DAG fields including the mining itself. In other words, this service fills out all DAG fields (`parents`, `timestamp`, `weight`, and `nonce`).
