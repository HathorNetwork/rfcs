# Summary

Create a transaction template api on the wallet-lib that can be used to build transactions by following user defined instructions and implement an endpoint on wallet-headless to allow user build a transaction by offering a transaction template as a JSON object.

# Motivation

Some clients require special additions to how the transactions are created, for instance when sending authority to a certain addresses after the transaction, allowing the authority address to be from outside the wallet, adding a special output in the transaction, etc.

Each of these use cases would require us to write new methods or extend existing ones in the wallet-lib, which translates to a reasonable cost of development and risk to introduce bugs.

A compelling solution for this problem is to create an abstraction layer of transaction creation by using transaction description.
This way we detach the transaction building logic from the specific implementation details, allowing for greater flexibility and extensibility.
This approach minimizes the risks associated with frequent code changes while giving users more control over transaction structure, though it may introduce new responsibilities for users in ensuring correct template design.

# Guide-level explanation

We can describe a transaction by using a system of *instructions template* set that represents a collection of basic transaction components like *input*, *output* and common operations over inputs or outputs by the means of *actions*.

A user must be able to control instructions to design a transaction by using a *builder* which should be able to build a *transaction template* and use an *interpreter* to execute its instructions to generate a final *transaction*.

For this, we can segregate the mentioned system in three areas:

- User land
  - public set of interfaces, types and classes that will be used to create templates.
- Interpreter
  - public set of classes used by wallets to build transactions from a template.
- Lib land
  - protected internal methods and utilities used to create templates and transaction.

On *user land* we provides:

- Template instructions schemas
- Transaction template builder
- Transaction template parser

On *interpreter* side we provides:

- Wallet facade interpreter
- Wallet service interpreter

On *lib land* we provides:

- All the existing methods for transaction preparation, creation and send transaction, including helpers and utils to create transaction components
- New methods to assist the interpreters, if needed

```mermaid
flowchart TD;
%% Nodes
    USER(fa:fa-user User)
    INS("fa:fa-list Template instructions")
    TMPL("fa:fa-code Template instance")
    INT("fa:fa-cubes Interpreter")
    WALLET("fa:fa-wallet Wallet")
    TX(fa:fa-shapes Transaction)

%% Connections
    USER -- Creates --> INS
    INS --> TMPL
    INT -- Read instructions --> TMPL
    USER --> WALLET
    WALLET <-- Calls --> INT
    INT == Builds ==> TX

    style INT color:#FFFFFF, fill:#AA00FF, stroke:#AA00FF
    style INS color:#FFFFFF, stroke:#00C853, fill:#00C853
    style WALLET color:#FFFFFF, stroke:#00C853, fill:#00C853
    style TMPL color:#FFFFFF, stroke:#00C853, fill:#00C853
    style USER color:#FFFFFF, stroke:#2962FF, fill:#2962FF
    style TX color:#FFFFFF, stroke:#014743, fill:#014743
   ```

## User land

### Instruction template interfaces

**Input:**

- [UtxoSelectInstruction](#utxoselectinstruction): query token UTXOs to add as inputs.
- [AuthoritySelectInstruction](#authorityselectinstruction): query authority UTXOs to add as inputs.
- [RawInputInstruction](#rawinputinstruction): can determine a known UTXO at the risk to be already spent.

**Output:**

- [TokenOutputInstruction](#tokenoutputinstruction): Creates a token output.
- [AuthorityOutputInstruction](#authorityoutputinstruction): Creates one or more authority outputs.
- [DataOutputInstruction](#dataoutputinstruction): Creates a data output.
- [RawOutputInstruction](#rawoutputinstruction): Creates an output from the raw values.

**Action:**

- [ShuffleInstruction](#shuffleinstruction): can determine a shuffle over a target array, either an input or output
- [ChangeInstruction](#changeinstruction): can determine a change output generation
- [CompleteTxInstruction](#completetxinstruction): complete the transaction balance.
- [ConfigInstruction](#configinstruction): Change a transaction data that is not related to the inputs and outputs.
- [SetVarInstruction](#setvarinstruction): Set a variable in the template context.

#### Template variables

Some arguments of the template can be expressed as a contextualized variable for the build runtime.
This means that you can set a variable during the build execution and refer to that variable on subsequent instructions.

To set a variable this can be done via the [SetVarInstruction](#setvarinstruction) where the variable value and name will be chosen.
Other instructions will be able to refer to this variable if they have an property with one of the template variable types.

A template reference string will be a string with brackets around the variable name, for instance `"{addr}"` will reference a variable called `addr`.
Any argument that can be a template variable will accept the template reference or its usual type.

```ts
const TEMPLATE_REFERENCE_RE = /\{[\w\d]+\}/;

const TemplateRef = z.string().regex(TEMPLATE_REFERENCE_RE);

const NumberThatCanBeAvar = TemplateRef.or(z.coerce.number());
```

#### `UtxoSelectInstruction`

Query token UTXOs to add as inputs.

```ts
const UtxoSelectInstruction = z.object({
  type: z.literal('input/utxo'),
  position: z.number().default(-1),
  fill: TemplateRef.or(z.coerce.bigint()),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}|00$/).default('00')),
  address: TemplateRef.or(z.string().optional()),
  autoChange: z.boolean().default(true),
  changeAddress: TemplateRef.or(z.string().optional()),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
- `fill`: amount of tokens to select
  - It may require multiple selection of UTXO to match the amount
  - It doesn't imply the generation of a change output
  - The amount selected may surpass the `fill`, so a change output may be required.
- `token`: which token to select, defaults to native token, in practice `00` (HTR). User must set the token UID here.
- `address`: find only UTXOs of this address to fill the amount
- `autoChange`: whether to automatically add any surplus (amount selected minus `fill`) in a change output, defaults to `true`
  - If `true` and a change is generated, then it will add the output change at the final position of outputs array
- `changeAddress`: If a change is generated with `autoChange` it will be sent to this address.

#### `AuthoritySelectInstruction`

Query authority UTXOs to add as inputs.

```ts
const AuthoritySelectInstruction = z.object({
  type: z.literal('input/authority'),
  position: z.number().default(-1),
  authority: z.enum(['mint', 'melt']),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}$/)),
  count: TemplateRef.or(z.coerce.bigint()),
  address: TemplateRef.or(z.string().optional()),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
- `authority`: `mint` or `melt`
- `token`: which token to select. User must set the token UID here.
- `amount`: amount of authorities to select
- `address`: find only UTXOs of this address to fill the amount

#### `RawInputInstruction`

Add a known UTXO as input on the transaction.

```ts
const RawInputInstruction = z.object({
  type: z.literal('input/raw'),
  position: z.number().default(-1),
  index: TemplateRef.or(z.coerce.number()),
  txId: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}$/)),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
- `txId`: Transaction ID os the UTXO.
- `index`: output index of the UTXO.

#### `TokenOutputInstruction`

Add an output to the transaction

```ts
const TokenOutputInstruction = z.object({
  type: z.literal('output/token'),
  position: z.number().default(-1),
  amount: TemplateRef.or(z.coerce.bigint()),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}|00$/).default('00')),
  address: TemplateRef.or(z.string()),
  timelock: TemplateRef.or(z.coerce.number()).optional(),
  checkAddress: z.boolean().optional(),
  createdToken: z.boolean().default(false),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
- `amount`: amount of tokens
- `token`: which token to select, defaults to native token, in practice `00` (HTR). User must set the token UID here.
- `address`: create an output script for the address.
- `timelock`: UNIX timestamp, the generated output may not be spent before this date and time.
- `checkAddress`: whether to check that the address is from the wallet, defaults to `false`
  - If `true` and address is not from the wallet, then the interpreter should fail
- `createdToken`: If this is true we will reference the token being created in this transaction
  - This means using `tokenData = 1` even if the token array is empty.
  - We will ignore the `token` argument even if one is present.

#### `AuthorityOutputInstruction`

Add an authority output to the transaction

```ts
const AuthorityOutputInstruction = z.object({
  type: z.literal('output/authority'),
  position: z.number().default(-1),
  count: TemplateRef.or(z.coerce.number()),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}$/)),
  authority: z.enum(['mint', 'melt']),
  address: TemplateRef.or(z.string()),
  timelock: TemplateRef.or(z.coerce.number()).optional(),
  checkAddress: z.boolean().optional(),
  createdToken: z.boolean().default(false),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
- `count`: amount of authority outputs to create.
- `token`: which token to select. User must set the token UID here.
- `address`: create an output script for the address
- `timelock`: UNIX timestamp, the generated output may not be spent before this date and time.
- `checkAddress`: whether to check that the address is from the wallet, defaults to `false`
  - If `true` and address is not from the wallet, then the interpreter should fail
- `createdToken`: If this is true we will reference the token being created in this transaction
  - This means using `tokenData = 1` even if the token array is empty.
  - We will ignore the `token` argument even if one is present.


#### `DataOutputInstruction`

```ts
const DataOutputInstruction = z.object({
  type: z.literal('output/data'),
  position: z.number().default(-1),
  data: TemplateRef.or(z.string()),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}|00$/).default('00')),
  createdToken: z.boolean().default(false),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
- `data`: UTF-8 encoded string of data
- `token`: which token to select. User must set the token UID here.
  - Defaults to HTR.
- `createdToken`: If this is true we will reference the token being created in this transaction
  - This means using `tokenData = 1` even if the token array is empty.
  - We will ignore the `token` argument even if one is present.

#### `RawOutputInstruction`

```ts
const RawOutputInstruction = z.object({
  type: z.literal('output/raw'),
  position: z.number().default(-1),
  amount: TemplateRef.or(z.coerce.bigint()),
  script: TemplateRef.or(z.string()),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}|00$/).default('00')),
  timelock: TemplateRef.or(z.coerce.number()).optional(),
  authority: z.enum(['mint', 'melt']).optional(),
  createdToken: z.boolean().default(false),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
- `amount`: amount of tokens
- `script`: base64 encoded script
- `token`: which token to select, defaults to native token, in practice `00` (HTR). User must set the token UID here.
- `authority`: `mint` or `melt`, if present will create the desired authority output
  - If the required input is not present, then it will generate an invalid transaction
- `timelock`: UNIX timestamp, the generated output may not be spent before this date and time.
- `createdToken`: If this is true we will reference the token being created in this transaction
  - This means using `tokenData = 1` even if the token array is empty.
  - We will ignore the `token` argument even if one is present.

#### `ShuffleInstruction`

```ts
const ShuffleInstruction = z.object({
  type: z.literal('action/shuffle'),
  target: z.enum(['inputs', 'outputs', 'all']),
});
```

It will apply an array shuffle action over the selected target.

#### `ChangeInstruction`

```ts
const ChangeInstruction = z.object({
  type: z.literal('action/change'),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}$/)).optional(),
  address: TemplateRef.or(s.string()).optional(),
  timelock: TemplateRef.or(z.coerce.number()).optional(),
});
```

This action creates change output to match the outstanding balance of the transaction.
If `token` is present we will only generate change for this specific token, if not we will generate change outputs for all tokens.
`address` and `timelock` will control the generated change outputs.

#### `CompleteTxInstruction`

```ts
const CompleteTxInstruction = z.object({
  type: z.literal('action/complete'),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}$/)).optional(),
  address: TemplateRef.or(s.string().optional()),
  changeAddress: TemplateRef.or(s.string().optional()),
  timelock: TemplateRef.or(z.coerce.number()).optional(),
});
```

This action will try to complete a transaction, meaning that if inputs are required it will try to select inputs, if outputs are required it will add change outputs to match the balance.
If `token` is present we will only try to complete this specific token, if not we will complete all tokens.
`address` will control the generated inputs, while `changeAddress` and `timelock` will control the generated change outputs.

#### `ConfigInstruction`

```ts
const ConfigInstruction = z.object({
  type: z.literal('action/config'),
  version: TemplateRef.or(z.number()).optional(),
  signalBits: TemplateRef.or(z.number()).optional(),
  tokenName: TemplateRef.or(z.string()).optional(),
  tokenSymbol: TemplateRef.or(z.string()).optional(),
});
```

This action can be used to configure some transaction data that does not affect the inputs or outputs.

#### `SetVarInstruction`

```ts
const SetVarGetWalletAddressOpts = z.object({
  unused: z.boolean().optional(),
  withBalance: z.number().optional(),
  authority: z.enum(['mint', 'melt']).optional(),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}|00$/).default('00')),
});

const SetVarGetWalletBalanceOpts = z.object({
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}|00$/).default('00')),
});

const SetVarOptions = z.union([SetVarGetWalletAddressOpts, SetVarGetWalletBalanceOpts]);

const SetVarInstruction = z.object({
  type: z.literal('action/setvar'),
  name: z.string().regex(/[\d\w]+/),
  value: z.any().optional(),
  action: z.enum(['get_wallet_address', 'get_wallet_balance']),
  options: SetVarOptions.optional(),
});
```

This action can be used to create a reusable variable that other instructions can reference.
The variable will be referenced by the `name` the user provides and the value will be determined by the `value` or `action`.

If `value` is provided the variable will be stored with it, but if `action` is provided the `options` will be used to determine the actual value to store.

- `get_wallet_address` will get an address from the wallet
  - If no option is provided a simple unused address will be fetched.
  - If `option` is provided it will be used to determine the address.
    - `unused` should be used to filter if we want an address with at least 1 tx in its history of not.
    - `withBalance`
      - should be used to find an address with at least this amount of tokens.
    - `withAuthority`: `mint` or `melt`
      - should be used to find an address with this authority
      - `withBalance` is now the number of this authorities the wallet should have.
    - `token`: token UID
      - should be used to determine which token the balance and authority is refering to, defaults to HTR.
- `get_wallet_balance` will get the wallet balance for a token
  - If no option is provided the HTR balance will be returned.
  - If `option` is provided it will be used to fetch the balance for a specific token.
    - `token` will determine which token to get the balance, defaults to HTR.

To use a stored variable the template should have a string with brackets with the variable name inside, for instance `{addr}` for the `addr` variable.

```json
[
  {
    "type": "action/setvar",
    "name": "changeAddr",
    "action": "get_wallet_address"
  },
  {
    "type": "action/setvar",
    "name": "dstAddr",
    "action": "get_wallet_address"
  },
  {
    "type": "action/setvar",
    "name": "TKA",
    "value": "00002ba722721e482cf3e32310e1ecf7d589744f9c5043b1b88cb4403ae6bfba"
  },
  {
    "type": "output/token",
    "amount": 100,
    "token": "{TKA}",
    "address": "{dstAddr}"
  },
  {
    "type": "input/utxo",
    "fill": 100,
    "token": "{TKA}",
    "autoChange": false
  },
  {
    "type": "action/change",
    "token": "{TKA}",
    "address": "{changeAddr}",
  },
]
```

In this example we send 100 tokens from our wallet to our wallet, we ensure that the destination address is different from the change address even if both are from the same wallet.
We also use a specific token in all instances, which is easier to notice since we don't have to compare the 64 character string on all instructions to know that we are using the same token.

### Transaction template

The transaction template itself is a list of instructions so the scheme for the transaction template can be made with:

```ts
const TxTemplateInstruction = z.discriminatedUnion('type', [
  RawInputInstruction,
  UtxoSelectInstruction,
  AuthoritySelectInstruction,
  RawOutputInstruction,
  DataOutputInstruction,
  TokenOutputInstruction,
  AuthorityOutputInstruction,
  ShuffleInstruction,
  ChangeInstruction,
  CompleteTxInstruction,
  ConfigInstruction,
  SetVarInstruction,
]);
type TxTemplateInstructionType = z.infer<typeof TxTemplateInstruction>;

const TransactionTemplate = z.array(TxTemplateInstruction);
type TransactionTemplateType = z.infer<typeof TransactionTemplate>;
```

With this we can parse and validate transaction templates from user input.

### Transaction template builder

A builder class must assists the user to build a `TransactionTemplate` instance.

```ts
class TransactionTemplateBuilder {
  build(): TransactionTemplate {}
  addInstruction(
    instruction: TxTemplateInstructionType
  ): TransactionTemplateBuilder {}
  addRawInput(ins: Omit<RawInputInstruction, "type">) {}
  addUtxoSelect(ins: Omit<UtxoSelectInstruction, "type">) {}
  addAuthoritySelect(ins: Omit<AuthoritySelectInstruction, "type">) {}
  addRawOutput(ins: Omit<RawOutputInstruction, "type">) {}
  addDataOutput(ins: Omit<DataOutputInstruction, "type">) {}
  addTokenOutput(ins: Omit<TokenOutputInstruction, "type">) {}
  addAuthorityOutput(ins: Omit<AuthorityOutputInstruction, "type">) {}
  addShuffleAction(ins: Omit<ShuffleInstruction, "type">) {}
  addChangeAction(ins: Omit<ChangeInstruction, "type">) {}
  addCompleteAction(ins: Omit<CompleteTxInstruction, "type">) {}
  addConfigAction(ins: Omit<ConfigInstruction, "type">) {}
  addSetVarAction(ins: Omit<SetVarInstruction, "type">) {}
}
```

Each `add` method will add an instruction of the same name, the methods will parse user and validate each instruction.

### Usage example

The following template will mint 1.00 TST, generate another mint authority to the wallet and it will have a 'foobar' data output as the first output.
The mint deposit inputs are manually added to the transaction and all outputs are shuffled except for the data output at position 0.

```json
[
  { "type": "action/setvar", "name": "TKA", "value": "000cafe...cafe123" },
  { "type": "action/setvar", "name": "addr", "action": "get_wallet_address" },
  { "type": "input/utxo", "fill": 2 },
  { "type": "input/authority", "authority": "mint", "token": "{TKA}" },
  { "type": "output/token", "amount": 100, "token": "{TKA}", "address": "{addr}" },
  { "type": "output/authority" "token": "{TKA}", "authority": "mint", "address": "{addr}" },
  { "type": "action/shuffle", "target": "outputs" },
  { "type": "output/data", "data": "foobar", "position": 0 }
]
```

This same transaction template instructions set could be built as:

```ts
// Token UID for token TST
let TST = '000cafe...cafe123';

const templ = new TransactionTemplateBuilder()
    .addSetVarAction({name: 'TKA', value: TST})
    .addSetVarAction({name: 'addr', action: 'get_wallet_address'})
    .addUtxoSelect({ fill: 2 })
    .addAuthoritySelect({ authority: 'mint', token: '{TKA}' })
    .addTokenOutput({ address: '{addr}', amount: 100, token: '{TKA}' })
    .addAuthorityOutput({ authority: 'mint', address: '{addr}', token: '{TKA}' })
    .addShuffleAction({ target: 'outputs' })
    .addDataOutput({ data: 'foobar', position: 0 })
    .build();
```

## Interpreters

A class which responsibility is to read the sequence of instructions to create a transaction and expose methods to get information from the user's wallet.

```ts
interface TxTemplateInstructionInterpreter {
  build(tt: TransactionTemplateType): Transaction;
  getAddress(): string;
  getChangeAddress(): string;
  getUtxos(amount: OutputValueType, options: IGetUtxosOptions): Promise<IGetUtxoResponse>;
  getAuthorities(amount: OutputValueType, options: IGetUtxosOptions): Promise<Utxo[]>;
  getTx(txId: string): Promise<IHistoryTx>;
  getNetwork(): Network;
}
```

- build: Will create a `TxTemplateContext` instance and execute each instruction with the context then return the resulting `Transaction` instance.
- getAddress: get a valid address from the wallet.
- getChangeAddress: get a change address from the wallet.
- getTx: get a transaction (may not be from the wallet).
- getNetwork: get the currently connected network.

We have two implementations of interpreters:

- WalletTemplateInterpreter
- WalletServiceTemplateInterpreter

```ts
class TxTemplateContext {
  version: number;
  signalBits: number;
  inputs: Input[];
  outputs: Output[];
  tokens: string[];
  balance: TxBalance;
  tokenName?: string;
  tokenSymbol?: string;
  vars: Record<TemplateVarName, TemplateVarValue>;

  constructor(...): {...}

  // Manage transaction tokens
  addToken() {}
  // Manage transaction balance
  addInput() {}
  addOutput() {}
}
```

## Lib land

Beyond the wallet API we can use any utils or helper available to support the interpreter in its mission.

For now we don't need to create anything. However, it worths to mention the `IHathorWallet` doesn't conveys all methods available on both wallets.
Just to reference one, we can take the `getUtxos` which is implemented in both wallets but is not present at `IHathorWallet`.
Not to mention the lack of type inference at wallet facade which makes the development experience poorer.
