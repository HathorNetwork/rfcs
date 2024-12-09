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
  address: TemplateRef.or(z.string()).optional(),
  autoChange: z.boolean().default(false),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
  - the request will fail if position doesn't exists
- `fill`: amount of tokens to select
  - It may require multiple selection of UTXO to match the amount
  - It doesn't imply the generation of a change output
  - The amount selected may surpass the `fill`, so a change output may be required.
- `token`: which token to select, defaults to native token, in practice `00` (HTR). User must set the token UID here.
- `address`: find only UTXOs of this address to fill the amount
- `autoChange`: whether to automatically add any surplus (amount selected minus `fill`) in a change output, defaults to `true`
  - If `true` and a change is generated, then it will add the output change at the final position of outputs array

#### `AuthoritySelectInstruction`

Query authority UTXOs to add as inputs.

```ts
const AuthoritySelectInstruction = z.object({
  type: z.literal('input/utxo'),
  position: z.number().default(-1),
  authority: z.enum(['mint', 'melt']),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}$/)),
  amount: TemplateRef.or(z.coerce.bigint()),
  address: TemplateRef.or(z.string()).optional(),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
  - the request will fail if position doesn't exists
- `authority`: `mint` or `melt`
- `token`: which token to select. User must set the token UID here.
- `amount`: amount of authorities to select
- `address`: find only UTXOs of this address to fill the amount

#### `RawInputInstruction`

Add a known UTXO as input on the transaction.

```ts
const RawInputInstruction = z.object({
  type: z.literal('input/utxo'),
  position: z.number().default(-1),
  index: TemplateRef.or(z.coerce.number()),
  txId: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}$/)),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
  - the request will fail if position doesn't exists
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
  address: TemplateRef.or(z.string()).optional(),
  timelock: TemplateRef.or(z.coerce.number()).optional(),
  checkAddress: z.boolean().optional(),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
  - the request will fail if position doesn't exists
- `amount`: amount of tokens
- `token`: which token to select, defaults to native token, in practice `00` (HTR). User must set the token UID here.
- `address`: create an output script for the address, if not present will get one from the wallet
  - It will get a not used address from the wallet
- `timelock`: UNIX timestamp, the generated output may not be spent before this date and time.
- `checkAddress`: whether to check that the address is from the wallet, defaults to `false`
  - If `true` and address is not from the wallet, then the interpreter should fail

>[!NOTE]
> Decision: Should we keep the default behavior on `address` or make it more strict? Like, if there isn't any `address` value, then fail short.
>
> To counter the behavior of a stricter `address` we can add a flag to let the user choose the desired behavior, either get a new address or get an address by index.
>
> Or we can make `address` a poli typed property which can accept `number` for indexes, the string `new` or a general `string` to denote an address.

#### `AuthorityOutputInstruction`

Add an authority output to the transaction

```ts
const AuthorityOutputInstruction = z.object({
  type: z.literal('output/authority'),
  position: z.number().default(-1),
  count: TemplateRef.or(z.coerce.number()),
  token: TemplateRef.or(z.string().regex(/^[a-fA-F0-9]{64}$/)),
  authority: z.enum(['mint', 'melt']),
  address: TemplateRef.or(z.string()).optional(),
  timelock: TemplateRef.or(z.coerce.number()).optional(),
  checkAddress: z.boolean().optional(),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
  - the request will fail if position doesn't exists
- `count`: amount of authority outputs to create.
- `token`: which token to select, defaults to native token. User must set the token UID here.
- `address`: create an output script for the address, if not present will get one from the wallet
  - It will get a not used address from the wallet.
- `timelock`: UNIX timestamp, the generated output may not be spent before this date and time.
- `checkAddress`: whether to check that the address is from the wallet, defaults to `false`
  - If `true` and address is not from the wallet, then the interpreter should fail


#### `DataOutputInstruction`

```ts
const DataOutputInstruction = z.object({
  type: z.literal('output/data'),
  position: z.number().default(-1),
  data: TemplateRef.or(z.string()),
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
  - the request will fail if position doesn't exists
- `data`: UTF-8 encoded string of data

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
});
```

- `position`: which index to insert the input, default to `-1`
  - can be `-1` to insert at the end
  - the request will fail if position doesn't exists
- `amount`: amount of tokens
- `script`: base64 encoded script
- `token`: which token to select, defaults to native token, in practice `00` (HTR). User must set the token UID here.
- `authority`: `mint` or `melt`, if present will create the desired authority output
  - If the required input is not present, then it will generate an invalid transaction
- `timelock`: UNIX timestamp, the generated output may not be spent before this date and time.

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

If `token` is present we will only generate change for this specific token, if not we will generate change outputs for all tokens.
`address` and `timelock` will control the generated change outputs.

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
    "type": "output/change",
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
  addInstruction(
    instruction: TxTemplateInstructionType
  ): TransactionTemplateBuilder {}
  addUtxoInput(
    position: number,
    fill: number,
    token?: string,
    authority?: `mint` | `melt`,
    address?: string,
    autoChange?: boolean,
  ): TransactionTemplateBuilder {}
  addRawInput(
    position: number,
    txId: string,
    index: number,
  ): TransactionTemplateBuilder {}
  addTokenOutput(
    position: number,
    amount: number,
    token?: string,
    authority?: `mint` | `melt`,
    address?: string,
    checkAddress?: boolean,
  ): TransactionTemplateBuilder {}
  addDataOutput(
    position: number,
    data: string,
  ): TransactionTemplateBuilder {}
  addRawOutput(
    position: number,
    amount: number,
    script: string,
    token?: string,
    authority?: `mint` | `melt`,
  ): TransactionTemplateBuilder {}
  addShuffleAction(
    target: 'inputs'|'outputs'|'all',
  ): TransactionTemplateBuilder {}
  addFillChangeAction(
    target: 'inputs'|'outputs'|'all',
  ): TransactionTemplateBuilder {}
  build(): TransactionTemplateType {}
}
```

- `addInstruction`: add a general instruction as an object literal
- `addUtxoInput`: add the specific UTXO input instruction
- `addRawInput`: add the specific Raw input instruction
- `addTokenOutput`: add the specific Token output instruction
- `addDataOutput`: add the specific Data output instruction
- `addRawOutput`: add the specific Raw output instruction
- `addShuffleAction`: add the specific Shuffle action instruction
- `addFillChangeAction`: add the specific FillChange action instruction
- `build`: build and returns a `TransactionTemplate` instance

### Usage example

The following template will mint 0.01 TST, generate another mint authority to the wallet and it will have a 'foobar' data output as the first output.
The mint deposit inputs are manually added to the transaction and all outputs are shuffled except for the data output at position 0.

```json
[
  { "type": "input/utxo", "position": -1, "fill": 2, "token": "00", "autoChange": false },
  { "type": "input/utxo", "position": -1, "fill": 1, "token": "TST_UID", "authority": "mint", "autoChange": false },
  { "type": "output/token", "position": -1, "token": "TST_UID", "amount": 1 },
  { "type": "output/token", "position": -1, "token": "TST_UID", "authority": "mint", "amount": 1 },
  { "type": "action/change" },
  { "type": "action/shuffle", "target": "outputs" },
  { "type": "output/data", "position": 0, "data": "foobar" },
]
```

This same transaction template instructions set could be built as:

```ts
// Token UID for token TST
let TST = '000cafe...cafe123';

const templ = new TransactionTemplateBuilder()
  .addUtxoInput(-1, 2)
  .addUtxoInput(-1, 2, TST, "mint")
  .addTokenOutput(-1, 1, TST)
  .addTokenOutput(-1, 1, TST, "mint")
  .addShuffleAction("outputs")
  .addDataOutput("foobar")
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

>[!NOTE]
>Decision Call: Should we use this implementation as an opportunity to improve the lib in the topics mentioned?
