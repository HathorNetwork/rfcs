- Feature Name: Feature Activation Explorer UIs
- Start Date: 2023-07-03
- Author: Gabriel Levcovitz <<gabriel@hathor.network>>

# Summary
[summary]: #summary

Given the [Feature Activation for Blocks RFC](./0001-feature-activation-for-blocks.md), a mechanism is provided to deploy new features in the Hathor network conditionally on some criteria. It defined two new Explorer user interfaces: a new page containing a table of all features in the Feature Activation process and their metadata, and a new panel in the Block interface displaying the signal bits and respective features for that block. This RFC details those interfaces.

# Motivation
[motivation]: #motivation

There should be an easy way for users to follow the Feature Activation process and each feature's current state. Using the Explorer is ideal for this.

# Guide-level explanation
[Guide-level explanation]: #guide-level-explanation

This section defines what the interfaces will contain and provides a simple mockup for them.

## New Feature Activation interface

There must be a new option in the Tools dropdown, at the Explorer header. When clicked, it will open a new page specific for Feature Activation information. It will include a table as described in [this section](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md#explorer-user-interface) of the original RFC, containing all Criteria for each feature, and also it's current state and acceptance percentage.

Also, there must be an indication on the block used to retrieve the "current" information, that is, the best block when the request was made. Its height should be displayed and there should be a link to its page (this is not shown in the mockup below).

### Mockup

![feature_activation.png](0003-images%2Ffeature_activation.png)

## New panel in Block interface

There must be a new panel in the Block interface containing the Feature Activation information for that block, that is, a table with the block's signal value for each feature, the corresponding bit and feature name, and the feature state at that point.

### Mockup

![block.png](0003-images%2Fblock.png)

# Reference-level explanation
[Reference-level explanation]: #reference-level-explanation

## New Feature Activation interface

### Backend

The API described in [this section](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md#rest-api) of the original RFC is enough to power this interface and is already implemented.

### Frontend

A new page will be created, analogous to the [Tokens Explorer interface](https://explorer.hathor.network/tokens), leveraging its table component.

## New panel in Block interface

### Backend

The [existing](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md#feature-service) `FeatureService.get_bits_description()` method will be used to power this panel, although some manipulation will be necessary to convert it to this schema:

```json
{
  "signal_bits": [
    { "bit": 0, "signal": 1, "feature": "MY_NEW_FEATURE_1", "state": "STARTED" },
    { "bit": 1, "signal": 0, "feature": null, "state": null },
    { "bit": 2, "signal": 1, "feature": "MY_NEW_FEATURE_2", "state": "MUST_SIGNAL" },
    { "bit": 3, "signal": 0, "feature": "MY_NEW_FEATURE_3", "state": "STARTED" }
  ]
}
```

A new endpoint must be provided to return this information for a specific block.

### Frontend

Should be straightforward to implement, creating a new panel analogous to the existing ones, like the Funds neighbors, and also leveraging the Tokens table component. The request to the endpoint should be made lazily, when the user clicks on "Click to show".
