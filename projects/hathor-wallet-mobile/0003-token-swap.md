# Summary
[summary]: #summary

Token swap with Dozer protocol on the mobile wallet.

# Motivation
[motivation]: #motivation

Trading tokens is probably the main activity users do on DeFi nowadays. And Dozer Finance is the main venue to perform that on Hathor Network.

We want to encourage users to discover new tokens and swap them, so adding this to the mobile wallet, the main wallet used, is a big push in this direction.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Dozer token swap protocol

The Dozer protocol for token swap is implemented as a nano contract on Hathor. 
The methods being called for a token swap depend on the user input and the exchange expected path returned by the contract.

Once both tokens are chosen the user will enter either the input token value or expected output token value, depending on which input area the user chooses the methods called from the contract will be different but should serve the same overall purpose.

With the tokens and value we can call a method on the contract to calculate the swap information, which returns a tuple with:
- `path`: Comma-separated string of pool keys to traverse in the format `{tokenA}/{tokenB}/{fee}`
- `amounts`: List of expected amounts at each step
- `amount_out`/`amount_in`: Final expected output amount or expected deposit
- `price_impact`: Overall price impact

The swap information can be used so the user can review the swap and proceed to call the actual swap method.

There are 2 swap methods depending on the number of itens on the calculated `path` if there is only 1 swap or multiple swaps.
Both methods work similarly by choosing actions to deposit the input tokens and withdraw the output tokens.

### Slippage

The actual exchange rate of a token swap is defined during the swap, so when choosing the input token value you must withdraw less than the expected output amount to account for slippage, if you try to withdraw more than the actual amount the swap will fail, if you withdraw less than the actual amount the exceeding amount will be left as balance for the user on the contract.

The same should apply when the user chooses the output token value, he should deposit more than the expected value to account for slippage and any outstanding amount will be left as balance on the contract.

This means that the accepted slippage will be defined by the user's wallet and the swap will automatically fail if slippage is more than expected by the user.

## Allowed Tokens

While the Dozer contract may allow many tokens for swaps we will only have a pre-approved set of tokens shown on the wallet.

The list of allowed tokens will be served via CDN to the wallet, the document will contain the list of tokens allowed in each network, each token will also have a link to the icon (for the UI) that will also be available via CDN.

Failure to resolve the list of tokens will block the swap screen from fully loading.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Token Swap UI

The token swap button on the main screen will be loaded depending on a feature flag, once clicked the allowed token list will be fetched during an initial load step before the user can interact with the swap form.

The screen should track if the user last changed the input token or the output token, this information is important to determine which method to call from the contract and how the swap will be made.

### Choosing tokens for the swap

Once the user selects both tokens and writes a value on either input or output token area the wallet will start a cancellable async task to fetch the swap information from the contract.

The fetching of the swap information has to be a cancellable async task because the user may change the tokens during the request so we will need to stop the current request and start a second one with the new tokens chosen.

When the exchange rate is known we will be able to calculate the desired values for the swap.
These values will be locked in when the user clicks the review button.

## Dozer token swap protocol methods

### User defined the input tokens

We will call the following method to receive the swap information.

`find_best_swap_path(self, amount_in: Amount, token_in: TokenUid, token_out: TokenUid, max_hops: int)`
- `amount_in`: The amount of input tokens
- `token_in`: The input token uid
- `token_out`: The output token uid
- `max_hops`: Maximum number of hops (1-3, but algorithm handles any number)

The path will define the next method to call.
if only 1 swap is detected on the path `swap_exact_tokens_for_tokens(self, ctx: Context, fee: Amount)` should be called, the input and output token and amounts should be defined in the actions.

If more than 1 swap is detected on the path `swap_exact_tokens_for_tokens_through_path(self, ctx: Context, path_str: str)` should be called, the input and output token and amounts should be defined in the actions.

### User defined the output tokens

We will call the following method to receive the swap information.

`find_best_swap_path_exact_output(self, amount_in: Amount, token_in: TokenUid, token_out: TokenUid, max_hops: int)`
- `amount_in`: The amount of input tokens
- `token_in`: The input token uid
- `token_out`: The output token uid
- `max_hops`: Maximum number of hops (1-3, but algorithm handles any number)

The path will define the next method to call.
if only 1 swap is detected on the path `swap_tokens_for_exact_tokens(self, ctx: Context, fee: Amount)` should be called, the input and output token and amounts should be defined in the actions.

If more than 1 swap is detected on the path `swap_tokens_for_exact_tokens_through_path(self, ctx: Context, path_str: str)` should be called, the input and output token and amounts should be defined in the actions.