# Summary
[summary]: #summary

Create a transaction template that can be used to form a transaction by following the instructions.

# Motivation
[motivation]: #motivation

Some clients require special additions to how the transactions are created (e.g. sending authority to certain addresses after the transaction, allowing the authority address to be from outside the wallet, adding a special output in the transaction, etc.).
Each of these would be added as an option on the wallet facade and allowing the wallet-lib to create the desired transaction, but each option requires discussion, development time and a new release from both the headless and wallet-lib.

A solution to this is a more flexible API that can receive a transaction template and follow its intructions to form a transaction, if the template is well contructed it can solve all special cases without additional development.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The transaction template will only contemplate version 0x01 transactions which means that we will not create tokens or NFTs with the template for now.
The transaction template will be a set of instructions on how to create a transaction and how to modify it.

The "user controlled" fields of a transaction are inputs and outputs, which means we can focus on instructions to select inputs and generate outputs.

```ts
interface TxTemplate {
  inputs?: InputTemplateInstruction[];
  outputs?: OutputTemplateInstruction[];
  actions?: ActionTemplateInstruction[];
}
```

## User instructions

### Input instructions

Each input has a `type` argument which will define the other arguments.
The `type` can be 'utxo', 'raw' or 'action' (defaults to 'utxo' if not present).

The `utxo` inputs will use the storage's `selectUtxos` to find the desired inputs.
- `token`: which token to select, defaults to HTR.
- `fill`: amount of tokens to select, may require multiple inputs to match.
- `authority`: 'mint' or 'melt', if present will find the desired authority utxos.
- `address`: find only utxos of this address.
- `autochange`: boolean, whether to add a change output if required, defaults to true.
- `position`: which index to insert the input.
  - can be `-1` to insert at the end, which is the default behavior.
  - if the position does not exist, the request will fail.

The `raw` inputs will have `tx_id` and `index` as arguments.

The `action` input instruction can only be 'shuffle' (i.e. `{"type": "action", "action": "shuffle"}`) which will simply shuffle the input array.

### Output instructions

Each output instruction will create an output and add it to the array.
Each output has a `type` argument which will define the other arguments.
The `type` can be 'token', 'data', 'raw' or 'action' (defaults to 'token' if not present).

`token` instruction arguments:
- `token`: token uid, defaults to HTR.
- `address`: create an output script for this address, if not present will get one from the wallet.
- `amount`: amount of tokens.
- `authority`: 'mint' or 'melt', if present will create the desired authority output.
- `checkAddress`: 'true' or 'false', whether to check that the address is from the wallet (defaults to false).
- `position`: which index to insert the output.
	- can be `-1` to insert at the end, which is the default behavior.
	- if the position does not exist, the request will fail.

`data` instruction arguments:
- `data`: utf-8 encoded string of the data.
- `position`: which index to insert the output.
	- can be `-1` to insert at the end, which is the default behavior.
	- if the position does not exist, the request will fail.

`raw` instruction arguments:
- `token`: token uid, defaults to HTR.
- `authority`: 'mint' or 'melt', if present will create the desired authority output.
- `script`: base64 encoded script.
- `amount`: number of tokens in this output.
- `position`: which index to insert the output.
	- can be `-1` to insert at the end, which is the default behavior.
	- if the position does not exist, the request will fail.

`action` instruction arguments:
- `action`: which action to execute.
	- 'shuffle': will shuffle the current output array.

### Action instructions

Like the outputs we will have a `type` key that defines the action, `type` can be `shuffle`, `add-output`, `add-input` or `fill-change`.

`shuffle` action arguments:
- `target`: 'outputs', 'inputs' or 'all', defaults to 'all'.

`add-output` action arguments:
- `output`: an output instruction (cannot be an `action`)

`add-input` action arguments:
- `input`: an input instruction (cannot be an `action`)

`fill-change` does not have arguments, as the name implies it will check the transaction balance and find any change outputs necessary.

### Example

The following template will mint 0.01 TST, generate another mint authority to the wallet and it will have a 'foobar' data output as the first output.
The mint deposit inputs are manually added to the transaction and all outputs are shuffled except for the data output at position 0.

```json
{
  "inputs": [
    { "fill": 2, "token": "00" },
    { "fill": 1, "token": "TST", "authority": "mint" }
  ],
  "outputs": [
    { "type": "token", "token": "TST", "amount": 1 },
    { "type": "token", "token": "TST", "authority": "mint", "amount": 1 },
    { "type": "action", "action": "shuffle" },
    { "type": "data", "data": "foobar", "position": 0 },
  ],
}
```
