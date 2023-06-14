- Feature Name: evm_compatible_bridge
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary

This document will describe the smart contract that will manage the brige on the EVM chain.

# Guide-level explanation

## Contracts

To support the features defined in the [design](./design.md#interactions) we need at least 2 contracts, one for the federation and one for the bridge.

### Bridge contract

The bridge contract will require support for the [ERC-1155 Multi-token standard](https://ethereum.org/pt/developers/docs/standards/tokens/erc-1155/) because the contract will be required to mint, destroy and manage multiple tokens (that were crossed from Hathor).
We will also require the contract to be able to receive tokens from the [ERC-777 token standard](https://eips.ethereum.org/EIPS/eip-777) because users will send tokens to our bridge so we can cross them to Hathor. This standard is compatible with [ERC-20](https://eips.ethereum.org/EIPS/eip-20) so we will also support it.

Since we only support tokens from the ERC-777 and ERC-20 interfaces we need a client for the ERC-1820 interface to check that a token is compatible with our bridge before starting any process to support it.

The bridge contract will also be required to have some user and admin methods.

#### User methods

- `getTokenId`
  - All tokens supported by the bridge will have a unique id, this includes tokens native to the EVM chain and tokens managed by the contract.
  - The user will provide a token address and optionally an integer which will be the ERC-1155 token id (for tokens managed by the bridge).
  - For the chain native token, the token id will always be `0` so this call is unnecessary.
- `crossTokens`
  - This method will be called by the user to send tokens to the bridge.
  - The bridge contract will check if the token is supported and if it is not, it will fail the transaction.
  - The user will send tokens to the bridge contract address.
  - The bridge will register this request with the federation contract.
  - The bridge contract will emit an event to log the request.
  - The amount of tokens must be valid according to the granularity rules.
- `requestSupportForToken`
  - The user will call this method to request support for a token (native to the EVM chain).
  - This method will register a unique id and register this request with the federation contract.
  - The bridge contract will emit an event to log the request.
- All features supoprted by the ERC-1155 token standard.

#### Admin methods

This contract is "ownable" and the admin will be able to make some operations, for instance:

- Contract suspension (i.e. pausable contract)
  - In case of a security breach or legal obstruction we may have to halt operations.
  - Example implementation [here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/Pausable.sol)
- Burn and mint tokens
  - The admin should be able to manage tokens in the bridge.
  - The admin can use this to refund tokens or make manual corrections.
- Configuration methods
  - The admin should be able to change the configuration of the contract, this includes changing the fee percentage, changing the federation contract address, etc.
- Set token id
  - The admin should be able to correct and change the mapping of equivalent tokens.
  - This is to correct any mistakes or to manually support new tokens.

#### Bridge contract storage

##### _Token equivalency_

Other than the storage required to implement the ERC-1155 token standard we require additional storage to track which tokens are allowed in the bridge and to map the equivalency of Hathor and EVM tokens.

For tokens native to the EVM compatible chain we require an identification called `nativeTokenId` which we will map to a local id.
This local id will be mapped to a Hathor token uid.
The same will be done in reverse, a Hathor native token when crossing to the EVM chain will mint an equivalent token in our ERC-1155 contract.
A token created in this contract will have a `uint256` id.

```solidity
struct nativeTokenId {
  address contract;
  uint256 tokenId; // Should be 0 for native tokens
}
// This nonce will begin at 2 and will increase with each new native token supported.
// 0 is reserved for the native token of the chain.
// 1 is reserved for the native token of Hathor.
// Tokens created in the bridge contract will also increase this nonce.
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
// The Hathor native token will not be mapped in the mappings above since its uid is 00 and not a 32 byte integer.
// To use Hathor native token (HTR) we can use the local id 1 which is reserved for this.
```

To check if the token is native to Hathor or the EVM we can check the contract address of the token, since all Hathor native tokens will create an EVM token on the bridge address.

#### Federation methods

Some methods can only be called by the federation contract (or admin).
These methods will be used to manage tokens, include support for new tokens, etc.

The federation will also provide a method so the bridge contract can notify when a user requests support for a token or a crossing of a token to Hathor.

### Federation contract

A good example implementation can be found [here](https://github.com/onepercentio/tokenbridge/blob/master/bridge/contracts/Federation.sol).
This contract has a very good example of a federation with all needed functionalities, it also includes an owner account for the federation that can change the members and the bridge being managed.

The main method of the federation is `voteProposal` which will be used to vote on a request.
Requests can be used to unlock tokens to a user, to support a new token, burn or mint tokens.
Each request will be saved with an id so it can be voted on and later checked if it was accepted or not, the id will be a hash of the request data.

The federation contract will gather votes for requests and when a request has reached the amount of votes required it will execute the request calling the necessary methods on the bridge contract.
The request will be marked as executed and no more votes will be accepted.

The federation contract should also be "pausable", where we suspend the `voteProposal` method but keep admin methods, e.g. add and remove members, change the managed bridge, etc.

# Rationale and alternatives

_Why use events for communication?_

An event (also called a log entry) is a cheaper way to store data on-chain.
The price to allocate a 4 byte word on storage is 20.000 gas.
In contrast an example of log gas for 2 topics and 200 bytes of data would cost 3.325 gas, 50 times more data for almost 10 times less gas.
The method to calculate log gas can be found [here](https://github.com/ethereum/go-ethereum/blob/8a24b563312a0ab0a808770e464c5598ab7e35ea/core/vm/gas_table.go#L220).

Cost is not the only benefit, responding to events is easier to implement than having getter methods for all types of required data.

_Why use ERC-1155?_

An alternative would be to use a SideTokenFactory contract (example [here](https://github.com/onepercentio/tokenbridge/blob/master/bridge/contracts/SideTokenFactory.sol)) but I believe we should strive to use the most common standards.

# Prior art

Although [this bridge](https://github.com/onepercentio/tokenbridge) implements a bridge between 2 EVM compatible chains, the concepts implemented there are a good reference for security and best practices.

# Future possibilities

- Add support for ERC-721 and ERC-1155 tokens.
