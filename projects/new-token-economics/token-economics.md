# RFC: New Token Creation Economics for Hathor Blockchain

- Feature Name: new_token_economics
- Start Date: 2024-07-17
- RFC PR: -
- Hathor Issue: (leave this empty)
- Author: bulldozer@dozer.finance

## Summary

This RFC proposes a new mechanism for token creation and transaction fees in the Hathor blockchain. The key changes are:

1. Remove the HTR deposit requirement for creating new tokens.
2. Introduce a flat fee for transactions involving custom tokens.
3. Allow token creators to choose between HTR or a custom token for fee payment.

## Motivation

The current token deposit system presents barriers for small projects or those requiring large initial token supplies. Thiggs new approach aims to:

1. Lower the entry barrier for token creation.
2. Provide flexibility for various use cases (e.g., fixed supply tokens, memecoins with large supplies).
3. Distribute the cost of maintaining the token among all token holders.
4. Create long-term demand for HTR through transaction fees.

## Guide-level explanation

### Token Creation

Users can create new tokens with a specified supply without depositing HTR. This allows for more flexibility in token design and reduces initial costs.

### Transaction Fees

For each transaction involving a custom token, a flat fee will be charged. The token creator will configure which token is used for fee payment:

1. If HTR is chosen: 0.01 HTR will be burned per transaction.
2. If a custom token is chosen: 1 custom token will be burned per transaction. This custom token must be created using the HTR deposit model.

### Fee Configuration

When creating a new token, the creator will specify:
1. The token to be used for transaction fees (HTR or a custom token).
2. If a custom token is chosen, they must provide proof of its creation using the HTR deposit model.

## Reference-level explanation

### Token Creation Process

1. User initiates token creation transaction.
2. User specifies total supply and fee configuration.
3. No HTR deposit is required.
4. The system validates the fee configuration (HTR or valid custom token).

### Transaction Validation

For each transaction involving a custom token:

1. Validate the presence of the required fee.
2. If HTR is the fee token:
   ```python
   if fee_token == HTR:
       assert fee_amount >= 0.01
       burn(0.01, HTR)
   ```
3. If a custom token is the fee token:
   ```python
   if fee_token != HTR:
       assert fee_amount >= 1
       assert fee_token.created_with_deposit
       burn(1, fee_token)
   ```

## Drawbacks

1. Potential for spam tokens due to lower creation costs.
2. Complexity in managing different fee tokens.
3. Possible reduction in immediate HTR demand compared to the deposit system.

## Rationale and alternatives

This approach balances the need for network sustainability with lower barriers to entry. Alternatives considered:

1. Keeping the deposit system but with lower percentages.
2. Implementing a sliding scale for deposits based on token supply.

The proposed system was chosen for its flexibility and long-term sustainability model.

## Unresolved questions

1. How to handle upgrades for existing tokens?
2. Should there be a mechanism to change fee configurations post-creation?
3. How to ensure the value stability of custom fee tokens?

## Future possibilities

1. Implement percentage-based fees paid to specific addresses for project revenue.
2. Dynamic fee adjustment based on network usage.
3. Integration with governance mechanisms for community-driven fee adjustments.