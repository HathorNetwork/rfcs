- Feature Name: evm_compatible_bridge
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary
[summary]: #summary

This document will describe the smart contract that will manage the brige on the EVM chain.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Bridge

To define which operations need to be implemented we first need to understand how the contract should interact with the bridge.

### Cross tokens Hathor -> EVM

This operation can only be initiated by the federation.

First an actor of the federation will propose the action, this should include the tx id on Hathor that initiated the cross (so we can ignore duplicates).
This action will emit an event for a new proposal, which the federation can then vote to approve.

Once enough votes are cast to accept the proposal, we can begin the operation.
If after 100 blocks the proposal was not accepted by the federation it will be considered as rejected.

If the token is native to Hathor:

1. We need to find the equivalent token on the EVM side.
  - if no such token exists yet, create it.
2. Then we will mint the desired amount of tokens on the destination address.
  - If we have configured the bridge to take fees, we will mint the fee to an address controlled by the federation.
3. Emit a `Cross` event to confirm the proposal was accepted and fulfilled.

If the token is native to the EVM compatible chain:

1. We need to unlock the amount of tokens requested.
2. Get the interface of the token contract (using ERC-1820) or assume its an ERC-20 token.
3. Use the interface to send the tokens from the contract's "lock" address to the requested destination.
  - The amount will be available since we have locked this amount when crossing to Hathor.
  - If we have configured the bridge to take fees, we will send the fee to an address controlled by the federation.
4. Emit a `Cross` event to confirm the proposal was accepted and fulfilled.

### Cross tokens EVM -> Hathor

This operation can be initiated by the user.

The caller will provide the token identifier, amount and destination address on Hathor.

If the token is native to Hathor it means the token is managed by our contract:

1. We will send the amount of tokens from the user address to a "crossing" state
  - The tokens are still connected to the address and the proposal.
  - They will not count to the caller balance.
  - If the proposal is rejected the caller can redeem the tokens.
2. We will emit a proposal event so the federation can vote to approve this crossing.
  - If after 100 blocks it has not been approved it will be considered as rejected.
3. Once approved we will destroy the amount of tokens in the "crossing" state
  - If we have configured the bridge to take fees, we will send the fee to an address controlled by the federation.
4. Emit a `Cross` event to confirm the proposal was accepted and fulfilled.
5. The federation will be in charge of sending the tokens to the destination on Hathor.

If the token is native to the EVM compatible chain:

1. We need to check if an equivalent Hathor token already exists.
  - If not we need to halt this operation.
  - To create the the equivalent token a separate method needs to be called to emit an event so the federation can create the token on Hathor and save the equivalency state on-chain.
2. Get the interface of the token contract (using the ERC-1820 client) or assume its an ERC-20 token.
3. We will send the amount requested to the contract's "lock" address.
  - We need to save the proposal with metadata so the user can request to redeem the tokens if the proposal is rejected.
  - If we have configured the bridge to take fees, we will send the fee to an address controlled by the federation.
4. Emit an event to send the proposal to the federation.
5. The federation will be in charge of minting the tokens to the destination on Hathor.
6. Once the federation has minted the tokens on Hathor it should call a method to confirm the fulfillment by emitting the `Cross` event.

Obs.: Any token on the "lock" address can only be moved through this contract crossing methods.

## Contracts

To support the features defined in the bridge definition above we need at least 2 contracts, one for the federation and one for the bridge.

### Bridge contract

The bridge contract will require support for the standards:

- [ERC-1155: Multi-token standard](https://ethereum.org/pt/developers/docs/standards/tokens/erc-1155/)
  - Why: The contract will be required to mint, destroy and manage multiple tokens (when native to Hathor)
- [ERC-1820: Pseudo-introspection Registry Contract](https://eips.ethereum.org/EIPS/eip-1820) (client only)
  - Why: The contract will be required to communicate with other contracts, and having a client to this standard is a good way to check which interface we can use to communicate with the other contract.
  - We can also register our contract as ERC-1155 compatible.

It will also be required to provide:

- Contract suspension (i.e. pausable contract)
  - In case of a security breach or legal obstruction we may have to halt operations.
- User operations
  - Allow users to propose to send tokens across the bridge

#### Bridge contract: Suspension

A pausable contract ([example implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/Pausable.sol)) is a contract that can suspend some activities until an admin can "unpause" it.
The methods affected by the bridge pause will be the ERC-1155 and the methods to propose a token cross.

#### Bridge contract: Storage

_Token equivalency_

Other than the storage required to implement the ERC-1155 token standard we require additional storage to track which tokens are allowed in the bridge and to map the equivalency of Hathor and EVM tokens.

For tokens native to the EVM compatible chain we require an identification called `nativeTokenId` which we will map to a local id.
This local id will be mapped to a Hathor token uid.
The same will be done in reverse, a Hathor native token when crossing to the EVM chain will mint an equivalent token in our ERC-1155 contract.
A token created in this contract will have a `uint256` id.

```solidity
struct nativeTokenId {
  address contract;
  uint256 tokenId;
  isERC20 bool;
}
// This nonce will begin at 1 and will increase with each new native token supported.
uint256 localIdNonce;

// This is a mapping of each nativeTokenId to a local id.
// Tokens created in the bridge contract (with erc-1155) will also be mapped here.
// A local token id of `0` means the native token of the chain (e.g. in ethereum it would be ETH)
mapping (nativeTokenId => uint256) mappedNativeTokens;

// This maps a local token id to a hathor token uid
// This can be used to know which token in Hathor is equivalent to our native token.
mapping (uint256 => bytes32) nativeTokenIdToHathorUid;

// This maps a Hathor token uid to a local token id.
mapping (bytes32 => uint256) hathorUidToLocalTokenId;
```
Obs: if we want to implement support only for ERC-20 we can forget the local id since the contract address can be used as a token id.
Obs2: We should have getter method to retrieve equivalent tokens in both directions.

To check if the token is native to Hathor or the EVM we can check the contract address of the token, since all Hathor native tokens will create an EVM token on the bridge address.

### Federation contract

A good example implementation can be found [here](https://github.com/onepercentio/tokenbridge/blob/master/bridge/contracts/Federation.sol).
This contract has a very good example of a federation with all needed functionalities, it also includes an owner account for the federation that can change the members and the bridge being managed.

The main method of the federation is `voteTransaction` which is the only method only the federation can call.
It will add votes on the transaction until it passes the required amount of votes, then it will execute the bridge method to accept the transaction.
Any votes cast after this was accepted will not count since the transaction has already been processed.

The federation contract should also be "pausable", where we suspend the `voteTransaction` method but keep other methods, e.g. add and remove members, change the managed bridge, etc.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

_Why use events for communication?_

An event (also called a log entry) is a cheaper way to store data on-chain.
The price to allocate a 4 byte word on storage is 20.000 gas.
In contrast an example of log gas for 2 topics and 200 bytes of data would cost 3.325 gas, 50 times more data for almost 10 times less gas.
The method to calculate log gas can be found [here](https://github.com/ethereum/go-ethereum/blob/8a24b563312a0ab0a808770e464c5598ab7e35ea/core/vm/gas_table.go#L220).

Cost is not the only benefit, responding to events is easier to implement than having getter methods for all types of required data.

# Prior art
[prior-art]: #prior-art

Although [this bridge](https://github.com/onepercentio/tokenbridge) implements a bridge between 2 EVM compatible chains, the concepts implemented there are a good reference for security and best practices.
