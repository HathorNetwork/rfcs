- Feature Name: evm_compatible_bridge
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary

This document will describe the smart contract that will manage the brige on the EVM chain.

# Guide-level explanation

## Contracts

To support the features defined in the [design](./design.md#interactions) we need multiple contracts:

- Federation contract.
  - This contract will manage the federation and the voting process.
- Bridge contract.
  - This contract will manage the crossing process.
- Side token factory.
- Allowed tokens registry.

### Bridge contract

The bridge contract will require support to receive [ERC-777](https://eips.ethereum.org/EIPS/eip-777) tokens and since this standard is backwards compatible with [ERC-20](https://eips.ethereum.org/EIPS/eip-20) we will also support it.

Since we only support tokens from the ERC-777 and ERC-20 interfaces we need a client for the ERC-1820 interface to check that a token is compatible with our bridge before starting any process to support it.

The bridge contract will also be required to have some user and admin methods.

#### User methods

- `getTokenUid` and `getSideToken`
  - These methods will be used to get the Hathor token uid and the side token address for a given token.
- `receiveTokens` and `tokensReceived`
  - These methods are required for the contract to receive tokens.
- `sendTokens`
  - The user will provide the Hathor destination address along with the tokens to cross.
  - The method will be responsible to accept the tokens on the bridge, validate the Hathor address provided and calling `crossTokens` to initiate the process.
- `crossTokens`
  - This method will be private and called when the user sends tokens to the bridge.
  - The bridge contract will check if the token is supported and if it is not, it will fail the transaction.
  - The bridge will register this request with the federation contract.
  - The bridge contract will emit an event to log the request.
  - The amount of tokens must be valid according to the granularity rules.

#### Admin methods

This contract is "ownable" and the admin will be able to make some operations, for instance:

- Contract suspension (i.e. pausable contract)
  - In case of a security breach or legal obstruction we may have to halt operations.
  - Example implementation [here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/Pausable.sol)
- Burn and mint tokens
  - The admin should be able to manage tokens created by the bridge.
  - The admin can use this to refund tokens or make manual corrections.
- Configuration methods
  - The admin should be able to change the configuration of the contract, this includes changing the fee percentage, changing the federation contract address, etc.
- Set token id
  - The admin should be able to correct and change the mapping of equivalent tokens.
  - This is to correct any mistakes or to add support for new tokens.

#### Bridge contract storage

##### _Token equivalency_

The contract will require some data structures to be stored on the persistent storage mainly to track the mapping of Hathor tokens to EVM tokens and the crossing operations.

The ERC-777 and ERC-20 tokens are identified by the contract address and the Hathor token by its 32 bytes token uid.
There should be a map going both ways to map a contract address to a token uid and a map from token uid to contract address.

```solidity
// This maps a local token address to a hathor token uid
// This can be used to know which token in Hathor is equivalent to our native token.
mapping (address => bytes32) localTokenToHathorUid;

// This maps a Hathor token uid to a local token address.
mapping (bytes32 => address) hathorUidToLocalToken;
```

The only special case is Hathor native token (HTR) which has a token uid of `00` which is not 32 bytes long.
To use the same method we will map HTR to a 32 bytes sequence of zeroes, i.e. `00000000000000000000000000000000`.

### Federation contract

A good example implementation can be found [here](https://github.com/onepercentio/tokenbridge/blob/master/bridge/contracts/Federation.sol).
This contract has a very good example of a federation with all needed functionalities, it also includes an owner account for the federation that can change the members and the bridge being managed.

The main method of the federation is `voteProposal` which will be used to vote on a request.
Requests can be used to unlock tokens to a user, burn or mint tokens.
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

# Prior art

[This bridge](https://github.com/onepercentio/tokenbridge) implements a bridge between 2 EVM compatible chains, the concepts implemented there are a good reference for security and best practices.
