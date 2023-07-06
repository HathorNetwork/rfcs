- Feature Name: evm_compatible_bridge
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary

A bridge of tokens between Hathor and an EVM compatible blockchain.
The project will be divided in 3 parts:

- The EVM smart contract.
- The Hathor multisig wallet and coordinator service.
- The federation service that will connect events from the smart contract to the multisig wallet.

Each part will work to enable moving tokens between chains.

# Motivation

Being able to move tokens between Hathor and other chains will enable using other technologies with Hathor native assets.
We will also be able to take advantage of Hathor technology with assets from other chains, e.g. moving stable coins faster and without fees.

# Guide-level explanation

The main feature of a bridge is moving tokens between chains, first we will describe how a token is "moved" between chains.

## Moving tokens between chains

Weather a chain uses account or UTXO, an owned token is an entry on the chain ledger, this means we cannot actually send the token to the other chain.
To move a token _ATk_ from chain A to chain B we will "lock" the token on chain A and mint the same amount of a token _bATk_ token on chain B.
This way, every _bATk_ in existence has a _ATk_ locked on the original chain so that we can equate the value of 1 _bATk_ to 1 _ATk_.

The value of _bATk_ can only be guaranteed if we have an operation to move the tokens back to their original chain, effectively melting or destroying a certain amount of _bATk_ and "unlocking" the same amount of _ATk_.

### State and token equivalency

A token has a native chain which holds the worth of the token and we create a equivalent token on the secondary chain.
To know which token is equivalent to which token on the other chain we need a state to read from, this state should be updated when we create a equivalent token the first time.

The safest approach would be to save everything on-chain and since Hathor does not currently have a way to keep a structured state on-chain we need to use the EVM compatible chain as a storage.

#### Hathor native tokens

A Hathor custom token has a name and symbol, but is identified by its uid, a 32 byte string (in solidity it would be a `byte32`).
On the EVM side, an [ERC-777](https://eips.ethereum.org/EIPS/eip-777) contract will be created for each Hathor token, each identified by a contract address (i.e. `address`).
So we can map a Hathor token to a token id with `mapping(address => byte32)`.

The only special case would be the Hathor native token (i.e. HTR) that has an id of `00` instead of 32 bytes, to handle this we will use the token uid of `00000000000000000000000000000000` (32 zeroes) when refering to HTR.

#### EVM native token

A token created on the EVM chain does not have a universal identifier, it depends on which ERC the contract of the token is compliant,
[ERC-20](https://ethereum.org/pt/developers/docs/standards/tokens/erc-20/) and [ERC-777](https://ethereum.org/pt/developers/docs/standards/tokens/erc-777/) tokens are identified by the contract address and [ERC-721](https://ethereum.org/pt/developers/docs/standards/tokens/erc-721/) and [ERC-1155](https://ethereum.org/pt/developers/docs/standards/tokens/erc-1155/) use contract address plus token id (usually `uint256`).
We will require support for ERC-777 and ERC-20 tokens, which means we can use the contract address as the token id.
The token contract address can be mapped to the Hathor token uid created as its equivalent.

With this we can get the equivalent Hathor token from the id of a EVM native token.
The only special case is the EVM native token which does not have an address, for these we can use a wrapped native token, e.g. WETH or wrapper ether which is an ERC-20 compatible contract ([reference](https://etherscan.io/token/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2)).

## Interactions

### For tokens native to Hathor

#### Hathor Native: Hathor -> EVM

When a user wants to send a token native to Hathor over to the EVM chain, they will send the tokens to a Hathor multisig wallet with a special format, which will include a data output with the destination address.

```mermaid
sequenceDiagram
  box Hathor
  actor userH as User wallet on Hathor
  participant ms as MultiSig Wallet in Hathor
  end
  participant cood as Coordinator service
  participant fed as Federation
  box EVM
  participant evm as EVM smart contract
  actor userE as User address on EVM
  end

  userH->>ms: Send tokens
  ms--)+cood: Find a new transaction
  Note over cood: Save the request<br/>Wait 5 blocks for confirmation
  cood-->-cood: Make it available for polling

  fed->>cood: Poll for new requests
  cood-->>+fed: Return the list of new requests
  fed-->>-fed: Check if the request is valid
  fed->>+evm: Vote to create/Mint tokens
  Note over evm: Once enough votes are cast<br/>the tokens are created<br/>If 100 blocks pass and<br/>the votes are not enough<br/>the request is considered<br/>as rejected
  evm->>userE: Mint tokens
  evm--)-fed: Notify the request was completed with an event
  fed->>cood: Mark request as fulfilled
  Note over ms: Original tokens are locked on the wallet (with fees)
  Note over userE: Equivalent tokens are with the user (minus fees)
```

In the event of a failure, the request will be marked as rejected, the user then will have to contact the administrator to request a refund of tokens.
The sequence diagram of this interation would be as follow:

```mermaid
sequenceDiagram
  actor user
  actor admin
  actor userH as User wallet on Hathor
  participant ms as MultiSig Wallet in Hathor
  participant cood as Coordinator service
  participant fed as Federation
  participant evm as EVM smart contract

  user->>admin: request a refund and provide<br/> data to identify the transaction
  admin->>cood: Request a refund for the provided transaction
  cood->>cood: Check if the request was fulfilled on database
  cood->>evm: Call method to check if the request was fulfilled
  evm->>cood: Return data about the request
  Note over cood: If the request was fulfilled, the refund is not possible<br/>The service should return this as an error.
  cood->>+cood: Make a refund request available for polling
  fed->>cood: Pool for requests
  cood-->>fed: Return the refund request
  fed->>cood: Sign transaction and send signature
  cood->>ms: Once enough signatures are collected, push transaction
  ms->>userH: Send tokens
  cood->>-cood: Mark refund request as completed
  Note over userH: Tokens are back on the wallet<br/>We have validated that no tokens were sent to the user on the other chain<br/>So the refund is complete
```

#### Hathor Native: EVM -> Hathor

When a user wants to cross tokens native to Hathor back to Hathor, they will call a method on the EVM smart contract, which will burn the tokens and the federation will send them to the user address.

```mermaid
sequenceDiagram
  box EVM
  actor userE as User address on EVM
  participant evm as EVM smart contract
  end
  participant fed as Federation
  participant cood as Coordinator service
  box Hathor
  participant ms as MultiSig Wallet in Hathor
  actor userH as User wallet on Hathor
  actor admin as Admin address on Hathor
  end

  userE->>+evm: Call method to cross the tokens
  evm-->>evm: Burn tokens from the user account
  evm--)-fed: emit event with the request to cross tokens
  Note over fed: If the tokens are crossing back to Hathor<br/>we can be sure that there are enough tokens<br/>on the wallet to fulfill the request
  fed->>+cood: Notify a crossing of tokens from the EVM event
  Note over cood: Save the request<br/>Wait 5 blocks for confirmation
  cood-->>-cood: Make it available for polling

  fed->>cood: Poll for new requests
  cood-->>fed: Return the list of new requests
  fed-->>fed: Check if the request is valid
  fed->>+cood: Send signatures for the request
  cood->>ms: Once enough signatures are collected, push transaction
  ms->>userH: Send tokens (minus fees)
  ms->>admin: Send fees
  ms-->>cood: Return the transaction id
  cood->>-cood: Mark refund request as completed
  Note over userH: Tokens are with the user (minus fees)
  Note over admin: Fees are sitting on the admin address
```

If the request fails, the user will have to contact the administrator to request a refund of tokens.
Since the tokens are native to Hathor, the best course would be to have an admin method to manually burn the tokens on the EVM and use the already existing refund strategy to send the tokens back to the user on Hathor.

### For tokens native to the EVM chain

#### EVM Native: EVM -> Hathor

When a user wants to cross tokens native to the EVM chain to Hathor, they will call a method on the EVM smart contract, which will lock the tokens and the federation will mint the equivalent tokens on Hathor and send them to the user address.

```mermaid
sequenceDiagram
  box EVM
  actor admin as Admin address on EVM
  actor userE as User address on EVM
  participant evm as EVM smart contract
  end
  participant fed as Federation
  participant cood as Coordinator service
  box Hathor
  participant ms as MultiSig Wallet in Hathor
  actor userH as User wallet on Hathor
  end

  userE->>+evm: Call method to cross the tokens
  evm-->>evm: Check token is supported, if not, fail transaction
  evm-->>admin: Send fees to the admin address
  evm-->>evm: Send tokens from user account to the bridge account<br/>Tokens on the bridge account are considered "locked"
  evm--)-fed: emit event with the request to cross tokens<br/>This will include the token uid<br/>of the equivalent token on Hathor
  fed->>+cood: Notify a crossing of tokens from the EVM event
  Note over cood: Save the request<br/>Wait 5 blocks for confirmation
  cood-->>-cood: Make it available for polling

  fed->>cood: Poll for new requests
  cood-->>fed: Return the list of new requests
  fed-->>fed: Check if the request is valid
  fed->>+cood: Send signatures for the request
  cood->>ms: Once enough signatures are collected, push transaction
  ms->>userH: Mint tokens to user address
  ms-->>cood: Return the transaction id
  cood->>-cood: Mark cross request as completed
  Note over userH: Equivalent tokens are with the user
  Note over evm: Original tokens are locked on the bridge account
  Note over admin: Fees are sitting on the admin account
```

If the request fails, the user will have to contact the administrator to request a refund of tokens.
The admin will then use an admin method on the EVM contract to unlock the tokens and send them back to the user address.
Any refunds should check with the coordinator service first since he can check that the original request was fulfilled, rejected or on-going.

Adding support for tokens can only be done by the admin so the user will have to contact the admin and wait for the token to be added to the bridge allowed tokens.

#### EVM Native: Hathor -> EVM

When crossing an EVM native token back to the EVM chain, the user will send the tokens to the MultiSig wallet on Hathor and the coordinator service will begin the process to melt the tokens, once the tokens are melted the federation will unlock the tokens to the user address on the EVM chain, this operation will send the tokens from the bridge contract to the user address.

```mermaid
sequenceDiagram
  box Hathor
  actor userH as User wallet on Hathor
  participant ms as MultiSig Wallet in Hathor
  end
  participant cood as Coordinator service
  participant fed as Federation
  box EVM
  participant evm as EVM smart contract
  actor userE as User address on EVM
  end

  userH->>ms: Send tokens
  ms--)+cood: Find a new transaction
  Note over cood: Save the request<br/>Wait 5 blocks for confirmation<br/>We will start the process to<br/>melt the tokens on Hathor
  cood-->-cood: Make melt operation available for polling<br/>The operation will melt the amount of tokens minus fees

  fed->>cood: Poll for new requests
  cood-->>fed: Return the list of new requests
  fed-->>fed: Check if the request is valid
  fed->>+cood: Send signatures for the melt request
  cood->>ms: Once enough signatures are collected, push transaction

  Note over cood: We will start the process to<br/>send the tokens to the user on the EVM chain

  cood->>-cood: Make request available for polling
  fed->>cood: Poll for new requests
  cood-->>fed: Return the list of new requests
  fed-->>fed: Check if the request is valid
  fed->>+evm: Vote to create/Mint tokens
  Note over evm: Once enough votes are cast<br/>the tokens are created<br/>If 100 blocks pass and<br/>the votes are not enough<br/>the request is considered<br/>as rejected
  evm->>userE: Send tokens.
  evm--)-fed: Notify the request was completed with an event
  fed->>cood: Mark request as fulfilled
  Note over ms: Original tokens are locked on the wallet (with fees)
  Note over userE: Equivalent tokens are with the user (minus fees)
```

## Federation service

This service will be the main actor in the bridge, it will listen for events on the EVM bridge and on Hathor to create the transactions and calls necessary to cross tokens from one side of the bridge to another.

For increased security we will have a certain number of copies of the service running (i.e. M) each with their own private key,
And any operation on both sides of the bridge will require a minimum number of participants to agree to the transaction (i.e. N).

On the EVM side this can be achieved with a federation style voting system, under which some operations can be proposed by any participant of the federation but will require a minimum number of voters on this proposal so it can actually be executed.

On Hathor this is more strict, since the best way to ensure the minimum number of participants agree is to use a MultiSig wallet, which requires N out of M participants to sign a transaction before it is even recorded on the ledger.
The admins of this wallet cannot simply remove a participant from the MultiSig since it would change the actual wallet so any changes on the participants or the number of minimum signatures would require a redeploy of all instances of the service.

The EVM provides events (or logs) which can be used to communicate a new proposal and request for signatures, but for the MultiSig wallet on Hathor the communication needs to be handled in a different way.
To achieve this we introduce a coordinator service, which will listen for transactions in the MultiSig wallet and coordinate which utxos will be spent for each transaction.
This coordinator service will also provide some APIs so admins can issue commands to the federation and so admins can check the state of the bridge.

## Hathor MultiSig

Hathor MultiSig standard relies on a pay-to-script-hash (P2SH) model, this means that for any data to be written on-chain (e.g. transactions, token create/mint/melt operations) the signatures for this "write" must already be collected.

To handle operations on the MultiSig wallet each participant will run a copy of a headless wallet initialized with its seed and MultiSig configuration.

For the coordinator service to be notified that a transaction was sent to the "federation" MultiSig wallet we will use the queue plugin of the headless which enables real time notification of a received transaction (the queue can be SQS or rabbitMQ depending on where the service will be deployed).
The coordinator is not a participant of the federation so his headless wallet will be initialized with the MultiSig configuration but without a seed, this does not mean it is a readonly wallet since we will use it to send transactions with the collected signatures.

The federation will have to inspect any transaction proposed by the coordinator service and check if it is valid, to achieve this we need an API on the headless to that can return the transaction information with metadata (balance to our wallet, decoded data outputs, etc.).

## Bridge crossing fees

The fee percentage will be configured on the EVM contract and will be applied to all tokens crossing the bridge, the fee tokens will be kept on the chain their native of.

For clarification, we will list the operations and where the fees are kept:

- Hathor -> EVM (Hathor native)
  - The fee is deducted from the amount of tokens being crossed and kept on the MultiSig wallet in Hathor.
  - The admin can request the withdrawal from the wallet by using the coordinator service, which will check that we are not withdrawing more than the collected fees.
  - For EVM native tokens, we will only melt the amount of tokens minus fees, so the fees will automatically be kept on the MultiSig wallet.
- Hathor -> EVM (EVM native)
  - All tokens are melted on Hathor.
  - The tokens unlocked will be sent minus fees to the destination address.
  - The fees are sent to the admin account.
- EVM -> Hathor (Hathor native)
  - All tokens are destroyed on EVM and the transaction to unlock the tokens on Hathor will send the tokens minus fees to the destination and the fees to the admin address
- EVM -> Hathor (EVM native)
  - The fee is deducted from the amount of tokens being crossed and they will be sent to the admin account.
  - The admin account can be a MultSig contract that can receive ERC-777 and ERC-20 tokens but require acknowledgement from multiple accounts to spend these tokens.

When crossing from Hathor and from the EVM chain the same fee will be applied and the services should read the fee from the contract to ensure they are using the correct updated value.
The admin address on the EVM should be configured on the contract and the admin address on Hathor should be configured on the federation service.

## Add support for a new tokens

The admins should create the equivalent token and use the EVM contract to add support for this token in the bridge.

### Adding support for Hathor tokens

The admin will call a method on the contract passing the Hathor token data (token_uid, symbol and name), the bridge will create a side token, which is a ERC-777 token but the bridge will have authority to mint and destroy this side token.
After the side token is created the bridge contract will associate the contract address with the token uid, making the token supported by the bridge.

With the token supported the admin should make a call to the allowed tokens contract to make it possible to use the token on the bridge.

### Adding support for EVM native tokens

To cross a EVM native token we need to create the equivalent token on Hathor first before allowing it to be crossed.

1. Create a token on Hathor with the same name and symbol.
1. Melt the tokens created on the transaction above.
1. Create more authority outputs
1. Send all authorities to the federation MultiSig address.
1. Call a method on the bridge contract to map the EVM token to the created token uid.
1. Call a method on the allowed tokens contract to allow the token to be used on the bridge.

# Alternative solutions

## Granularity

Tokens defined in the ERC-777 have a granularity of 1e18, this means that the smallest amount of tokens that can be transferred is 1e18.
This is not compatible with the granularity of Hathor tokens, which means we cannot cross 0.0001 tokens to Hathor where the granularity is 1e2.

This limitation means that the bridge can only support sending multiple of 0.01 of any token since it is the higher granularity.

The tokens native to Hathor that cross to the EVM chain can be transactioned in smaller amounts (e.g. 0.0000001 or smaller), but to cross the token back to Hathor we need to send at least 0.01 tokens.

Some other token standards have variable granularity, but if we adhere to the solution that we will only accept multiples of the higher granularity we can support any token standard (when it comes to granularity).

# Future possibilities

Although Hathor's MultiSig has all the secutiry features required to make the bridge functional we still need to rely on external components,
i.e. a communication channel for actors and a storage for state management (e.g. token equivalency and transaction proposals).
This could be changed to use a nano-contract, where it can handle events, listing tx proposals and federation with a changing number of participants with 0 downtime.
Using nano-contracts would also enable very low cost deployment and infrastructure of bridges to new EVM compatible chains and allow for security updates to take effect on all bridges.

# Required features

- We require APIs on the headless wallet to create, mint and melt tokens.
- We require an API on the headless to return the transaction information with metadata (balance to our wallet, decoded data outputs, etc.).
- We require a way to start a MultiSig wallet in the lib without a seed or private key.
  - It should not be a read-only instance since it can send transactions (from gathered signatures)
