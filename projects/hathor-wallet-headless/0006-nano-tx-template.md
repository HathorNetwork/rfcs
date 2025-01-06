# Summary
[summary]: #summary

Nano contract transaction support in the tx template instruction set.

# Motivation
[motivation]: #motivation

Being able to create nano contracts using the transaction template module would make developing new applications with nano contracts easier and more developer friendly.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A nano contract transaction differs from a regular transaction due to the fields:

- `version`: Nano contract tx version is 4.
- `id`: ID of the nano contract being invoked.
- `method`: Method being invoked.
- `args`: Arguments being passed to the method.
- `pubkey`: Caller pubkey (used to identify caller).
- `signature`: Signed with the private key of `pubkey` (used to authenticate caller).

The inputs, outputs, tokens and other fields will work the same way as the regular tx.
This means that we can add these fields to the context via a specialized instruction for nano contract and keep the usual template workflow.

## Template instructions

```ts
const NanoConfigInstruction = z.object({
  type: z.literal('action/nano'),
  ncId: TemplateRef.or(Sha256),
  method: TemplateRef.or(z.string()),
  caller: TemplateRef.or(PubKeySchema),
  args: z.any().array().default([]),
});
```

> [!NOTE]
> Since we do not yet know the blueprint method arg structure we cannot effectively parse the input.
> We will accept any array as input but will parse the args during build time.

The caller pubkey may not be known while building the template but we can get any wallet pubkey using the `get_wallet_pubkey` setvar command.

```ts
// call method schema for config/setvar
const SetVarGetWalletPubkeyOpts = z.object({
  method: z.literal('get_wallet_pubkey'),
  index: z.number().optional(),
});
```

Example:

```json
[
  {
    "type": "action/setvar",
    "name": "callerPubkey",
    "call": { "method": "get_wallet_pubkey", "index": 10 },
  },
  {
    "type": "action/nano",
    "caller": "{callerPubkey}",
    "...": "...",
  }
]
```

## Differences with current builder

### Actions

The current nano transaction builder relies on "actions" to manage inputs and outputs or the transaction.
It allows 2 types of action:

- Deposit: Selects utxos to add as inputs and may add a change output.
- Withdrawal: Creates an output with the amount of tokens.

These 2 actions can be directly translated to the `input/utxo` and `output/token` instructions so we will not be creating new instructions for these.

### Individual methods for each data

The current nano transaction builder has a method for each individual information.

Example:

```ts
const builder = new NanoContractTransactionBuilder()
  .setMethod('paycoffee')
  .setWallet(wallet)
  .setBlueprintId(blueprintId)
  .setNcId(ncId)
  .setCaller(Buffer.from(pubkeyStr, 'hex'))
  .setActions([{
    type: 'deposit',
    token: '00',
    amount: 100,
  }])
  .setArgs(['bob', 'alice', 100n]);

const tx = await builder.build()
```

While the template builder for the same transaction would be:

```ts
const template = new TransactionTemplateBuilder()
  .addSetVarAction({ name: 'caller', call: { method: 'get_wallet_pubkey' } })
  .addNanoAction({
    ncId: ncId,
    method: 'paycoffee',
    caller: '{caller}',
    args: ['bob', 'alice', 100]
  })
  .addUtxoSelect({ fill: 100 })
  .build();
```
