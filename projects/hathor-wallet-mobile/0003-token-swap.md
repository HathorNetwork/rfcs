# Summary
[summary]: #summary

Token swap with Dozer protocol on the mobile wallet.

# Motivation
[motivation]: #motivation

Trading tokens is probably the main activity users do on DeFi nowadays. And Dozer Finance is the main venue to perform that on Hathor Network.

We want to encourage users to discover new tokens and swap them, so adding this to the mobile wallet, the main wallet used, is a big push in this direction.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## External services

#### Dozer protocol

The Dozer protocol for token swap is implemented as a nano contract on Hathor. 

> [!TODO]
> Describe methods we will call from dozer

#### Allowed Tokens

While the Dozer contract may allow many tokens for swaps we will only have a pre-approved set of tokens shown on the wallet.

The list and its managing methods will be implemented as a nano contract which the mobile wallet will read from.

The contract will have the methods:
- `get_allowed_tokens(self, ctx: Context)`
	- view method that returns `List[TokenUid]`
- `add_token(self, ctx: Context, token: SignedData[TokenUid])`
	- Add a token to the allowed list
	- Should only be called by the contract admin
- `remove_token(self, ctx: Context, token: SignedData[TokenUid])`
	- Remove a token from the allowed list
	- Should only be called by the contract admin

To minimize pressure on the fullnode, once the `get_allowed_tokens` method is called the wallet will cache the result as to not call the method repeatedly.

#### Feature Flag

Unleash allows a feature flag to have multiple variants, which we can use here to enable the token swap feature and also configure it once enabled. This means we can set the token list contract id on the feature flag itself.

Not having the contract id hard-coded or configured by network (mainnet, testnet, etc.) helps if in the future we decide to upgrade or change the contract, which will not affect any wallets that do not update.
With a feature flag we can rollout this change for all users, even if the wallet is not upgraded as long as the new contract maintains the same view method.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Loading the Token Swap

During the wallet initialization the feature flags will be fetched, if the token swap is enabled it will come with the token list contract id.
Once the contract id is known we can call the fullnode's API in the background to get the allowed tokens for swapping, this list should be cached locally.

A token swap button on the main screen will be available as soon as the wallet loads (if the feature flag is enabled), but if the user clicks this button before the allowed tokens are fetched there will be a loading screen until we know the allowed token list.

#### Choosing tokens for the swap

The empty token swap screen will prompt the user to choose tokens for the swap, once the tokens are chosen we will start a cancellable async task to fetch the exchange rate for the swap by calling the Dozer method \[\[**TODO**\]\].

The fetching of the exchange rate has to be a cancellable async task because the user may change the tokens during the request so we will need to stop the current request and start a second one with the new tokens chosen.

When the exchange rate is known we will be able to calculate the desired values for the swap.
These values will be locked in when the user clicks the review button.
