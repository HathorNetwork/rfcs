- Feature Name: evm_compatible_bridge
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary
[summary]: #summary

A bridge of tokens between Hathor and an EVM compatible blockchain.
The project will be divided in 3 parts:

- The EVM smart contract.
- The Hathor multisig wallet.
- The service that will connect events from the smart contract to the multisig wallet.

Each part will work to enable moving tokens between chains.

# Motivation
[motivation]: #motivation

Being able to move tokens between Hathor and other chains will enable using other technologies with Hathor native assets.
We will also be able to take advantage of Hathor technology with assets from other chains, e.g. moving stable coins faster and without fees.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

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
On the EVM side, a [ERC-1155](https://ethereum.org/pt/developers/docs/standards/tokens/erc-1155/) contract will handle the creation of multiple tokens in the same contract, each identified by an id (`uint256`).
So we can map a Hathor token to a token id with `mapping(uint256 => byte32)`.

The only special case would be the Hathor native token (i.e. HTR) that has an id of `00` instead of 32 bytes, this should be handled separately,
for instance by creating a token with id `0` when deploying the contract, so that the id `0` always represents HTR.

#### EVM native token

A token created on the EVM chain does not have a universal identifier, it depends on which ERC the contract of the token is compliant,
[ERC-20](https://ethereum.org/pt/developers/docs/standards/tokens/erc-20/) and [ERC-777](https://ethereum.org/pt/developers/docs/standards/tokens/erc-777/) tokens are identified by the contract address and [ERC-721](https://ethereum.org/pt/developers/docs/standards/tokens/erc-721/) and [ERC-1155](https://ethereum.org/pt/developers/docs/standards/tokens/erc-1155/) use contract address plus token id (usually `uint256`).
We should be able to support all of these standards by creating a local token id (separate from the ids of the tokens managed in this contract) and map it to a struct with the necessary data.
This id can be mapped to the Hathor token uid created as its equivalent.

With this we can get the equivalent Hathor token from the id of a EVM native token.

## Bridge service

This service will be the main actor in the bridge, it will listen for events on the EVM bridge and listen for transactions on Hathor and act accordingly.

For increased security we will have a certain number of copies of the service running (i.e. M) each with their own private key,
And any operation on both sides of the bridge will require a minimum number of participants to agree to the transaction (i.e. N).

On the EVM side this can be achieved with a federation style voting system, under which some operations can be proposed by any participant of the federation but will require a minimum number of voters on this proposal so it can actually be executed.
A proposal can be unlocking tokens, minting or destroying tokens, or even changing contract configurations like removing a participant or changing the minimum number.

On Hathor this is more strict, since the best way to ensure the minimum number agree is to use a MultiSig wallet, which requires N out of M participants to sign a transaction before it is even recorded on the ledger.
The admins of this wallet cannot simply remove a participant from the MultiSig since it would change the actual wallet so any changes on the participants or the number of minimum signatures would require a redeploy of all instances of the service.

The EVM provides events (or logs) which can be used to communicate a new proposal and request for signatures, but for the MultiSig wallet on Hathor the communication needs to be handled in a different way.

## Hathor MultiSig

Hathor MultiSig standard relies on a pay-to-script-hash (P2SH) model, this means that for any data to be written on-chain (e.g. transactions, token create/mint/melt operations) the signatures for this "write" must already be collected.
This makes is why we need a secondary communication channel for proposals since they were not yet accepted.

To handle operations on the MultiSig wallet each participant will run a copy of a headless wallet initialized with its seed and MultiSig configuration.
They will coordinate with the other service instances using the secondary communication channel and make operations on the wallet using the headless REST API.

For the service to be notified that a transaction was sent to the "federation" MultiSig wallet we will use the websocket extension of the headless which enabled real time notification of a received transaction.

# Future possibilities
[future-possibilities]: #future-possibilities

Although Hathor's MultiSig has all the secutiry features required to make the bridge functional we still need to rely on external components,
i.e. a communication channel for actors and a storage for state management (e.g. token equivalency and transaction proposals).
This could be changed to use a nano-contract, where it can handle events, listing tx proposals and federation with a changing number of participants with 0 downtime.
Using nano-contracts would also enable very low cost deployment and infrastructure of bridges to new EVM compatible chains and allow for security updates to take effect on all bridges.
