# Summary
[summary]: #summary

Create a transaction template api on the wallet-lib that can be used to build transactions by following user defined instructions and implement an endpoint on wallet-headless to allow user build a transaction by offering a transaction template as a JSON object.

# Motivation
[motivation]: #motivation

Some clients require special additions to how the transactions are created, for instance when sending authority to a certain addresses after the transaction, allowing the authority address to be from outside the wallet, adding a special output in the transaction, etc.

Each of these use cases would require us to write new methods or extend existing ones in the wallet-lib, which translates to a reasonable cost of development and risk to introduce bugs.

A compelling solution for this problem is to create an abstraction layer of transaction creation by using transaction description. This way we detach the transaction building logic from the specific implementation details, allowing for greater flexibility and extensibility. This approach minimizes the risks associated with frequent code changes while giving users more control over transaction structure, though it may introduce new responsibilities for users in ensuring correct template design.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We can describe a transaction by using a system of *instructions template* set that represents a collection of basic transaction components like *input*, *output* and common operations over inputs or outputs by the means of *actions*.

A user must be able to control instructions to design a transaction by using a *client* which should be able to build a *transaction template* and executes its interpretation to generate a final *transaction*.

For this, we can segregate the mentioned system in three areas:
- User land
- Interpreter
- Lib land

On *user land* we provides:
- Instruction template interfaces
- Instruction classes
- Transaction template builder
- Transaction template client

On *interpreter* side we provides:
- Wallet facade interpreter
- Wallet service interpreter

On *lib land* we provides:
- All the existing methods for transaction preparation, creation and send transaction, including helpers and utils to create transaction components
- New methods to assist the interpreters, if need

To limit the scope of this work, we are going to focus on transactions of version `0x01` which means that we will not create tokens or NFTs.

## User land

### Instruction template interfaces
[instruction-template-interfaces]: #instruction-template-interfaces

**Basic:**
- [BaseTemplateInstruction](#basetemplateinstruction): determine a instruction template type
- InputOutputTemplateInstruction: can control an insertion position into the array
- ActionTemplateInstruction: can determine over which component to act

These basic template instructions are for *internal* use and define **required fields** to declare a *template instruction literal* object by the user.

The following interfaces are describing each instruction type by its fields and possible values. A user can define a literal object of these types to design a final transaction.

**Input:**
- [UtxoInputTemplateInstruction](#utxoinputtemplateinstruction): can control any param to query a UTXO input
- RawInputTemplateInstruction: can determine a known UTXO at the risk to be already spent

**Output:**
- [TokenOutputTemplateInstruction](#utxoinputtemplateinstruction): can control a token output generation
- DataOutputTemplateInstruction: can control data output generation
- RawOutputTemplateInstruction: can determine an output

**Action:**
- [ShuffleActionTemplateInstruction](#shuffleactiontemplateinstruction): can determine a shuffle over a target array, either an input or output
- FillChangeActionTemplateInstruction: can determine a change output generation

#### `BaseTemplateInstruction`
[basetemplateinstruction]: #basetemplateinstruction

```ts
interface BaseTemplateInstruction {
  readonly type: AllInstructionTypes|string;
}
```

- `type`: select a type from all possible types among inputs, outputs and actions

#### `InputOutputTemplateInstruction`

```ts
interface InputOutputTemplateInstruction {
  /**
   * A positive integer
   * It allows -1 to denote the last position in the array
   */
  position: number;
}
```

- `position`: which index to insert the input, default to `-1`
  - [NOTE-1]: can be `-1` to insert at the end
  - [NOTE-2]: the request will fail if position doesn't exists

#### `ActionTemplateInstruction`

```ts
interface ActionTemplateInstruction {
  target: 'inputs'|'outputs'|'all';
}
```

- `target`: choose a target to execute an action over

#### `UtxoInputTemplateInstruction`
[utxoinputtemplateinstruction]: #utxoinputtemplateinstruction

```ts
interface UtxoInputTemplateInstruction extends InputOutputTemplateInstruction, BaseTemplateInstruction {
  fill: number;
  token?: string;
  authority?: `mint` | `melt`;
  address?: string;
  autochange?: boolean;
}
```

- `fill`: amount of tokens to select
	- [NOTE-1]: It may require multiple selection of UTXO to match the amount
	- [NOTE-2]: It doesn't imply the generation of a change output
- `token`: which token to select, defaults to native token, in practice `00` (HTR)
- `authority`: `mint` or `melt`
	- [NOTE-3]: It may miss any UTXO and result in an empty array
- `address`: find only UTXOs of this address to fill the amount
	- [NOTE-4]: It may miss any UTXOs and result in an empty array
- `autochange`: whether to add a change output if required, defaults to `true`
	- [NOTE-5]: If `true` and a change is generated, then it will add the output change at the final position of outputs array

#### `RawInputTemplateInstruction`

```ts
interface RawInputTemplateInstruction extends InputOutputTemplateInstruction, BaseTemplateInstruction {
  tx_id: string;
  index: number;
}
```

- `tx_id`: transaction ID in which the input belongs to
	- [NOTE-1]: It may miss the transaction
- `index`: output index in which the input is located
	- [NOTE-2]: It may miss the index

>[!NOTE]
>Decision: A miss here means an input will not be added to the list of inputs. It can drive the final transaction to be invalid, but the template itself will be generated nevertheless.

#### `TokenOutputTemplateInstruction`
[tokenoutputtemplateinstruction]: #tokenoutputtemplateinstruction

```ts
interface TokenOutputTemplateInstruction extends InputOutputTemplateInstruction, BaseTemplateInstruction {
  amount: number;
  token?: string;
  address?: string;
  authority?: `mint` | `melt`;
  check_address?: boolean;
}
```

- `amount`: amount of tokens
- `token`: which token to select, defaults to native token, in practice `00` (HTR)
- `address`: create an output script for the address, if not present will get one from the wallet
	- [NOTE-1]: It will get a not used address from the wallet
- `authority`: `mint` or `melt`, if present will create the desired authority output
	- [NOTE-2]: If the required input is not present, then it will generate an invalid transaction
- `check_address`: whether to check that the address is from the wallet, defaults to `false`
	- [NOTE-3]: If `true` and address is not from the wallet, then the interpreter should fail short

>[!NOTE]
>Decision Call: Should we keep the default behavior or `address` or make it more strict? Like, if there isn't any `address` value, then fail short.
>
>To counter the behavior of a stricter `address` we can add a flag to let the user choose the desired behavior, either get a new address or get an address by index.
>
>Or we can make `address` a poli typed property which can accept `number` for indexes, the string `new` or a general `string` to denote an address.

#### `DataOutputTemplateInstruction`

```ts
interface DataOutputTemplateInstruction extends InputOutputTemplateInstruction, BaseTemplateInstruction {
  data: string;
}
```

- `data`: UTF-8 encoded string of data
#### `RawOutputTemplateInstruction`

```ts
interface RawOutputTemplateInstruction extends InputOutputTemplateInstruction, BaseTemplateInstruction {
  amount: number;
  script: string;
  token?: string;
  authority?: `mint` | `melt`;
}
```

- `amount`: amount of tokens
- `script`: base64 encoded script
- `token`: which token to select, defaults to native token, in practice `00` (HTR)
- `authority`: `mint` or `melt`, if present will create the desired authority output
	- [NOTE-1]: If the required input is not present, then it will generate an invalid transaction

#### `ShuffleActionTemplateInstruction`
[shuffleactiontemplateinstruction]: #shuffleactiontemplateinstruction

```ts
interface RawOutputTemplateInstruction extends ActionTemplateInstruction, BaseTemplateInstruction {
}
```

It will apply an array shuffle action over the selected target.

#### `FillChangeActionTemplateInstruction`

```ts
interface RawOutputTemplateInstruction extends ActionTemplateInstruction, BaseTemplateInstruction {
}
```

The `target` doesn't have any effect. It will will check the transaction balance and calculate any change output, if any it will generate a change output and add it to the final position in outputs array.

>[!NOTE]
>Decision Call: We can add `position` property to allow user determine the position of the change output if it is generated.

### Instruction classes

Each template instruction interface must have its own class implementation. A class instance is useful to both help user to form an instruction and help the interpreter to form the transaction.

- UtxoInputInstruction
- RawInputInstruction
- TokenOutputInstruction
- DataOutputInstruction
- RawOutputInstruction
- ShuffleActionInstruction
- FillChangeActionInstruction

Nevertheless a user can still form an instruction using a literal object.

### Transaction template client

A client gives a user the ability to execute a transaction template object to make a transaction, and gives the user the responsibility to select the wallet, either Wallet Facade of Wallet Service.

```ts
class TransactionTemplate {
  constructor (instructions: BaseTemplateInstruction[]): TransactionTemplate {...}
  execute(wallet: IHathorWallet, options?: TransactionTemplateOptions): Transaction {...}
}
```

- `constructor`: gives user the ability to instantiate a `TransactionTemplate` from an array of instructions, and it may be an object literal while calling the constructor
- `execute`: triggers the interpretation of the set of instructions using the appropriate interpreter implementation, either the one for Wallet Facade or the one for Wallet Service, and must result in a `Transaction`
	- [NOTE-1]: It will generate the transaction and send it.

```ts
interface TransactionTemplateOptions {
  useWalletService: boolean;
}
```

- `useWalletService`: determine the use of Wallet Service, defaults to `false`, which means "use the Wallet Facade interpreter implementation".

>[!NOTE]
>In case of a mismatch in the passed wallet to `execute` signature and the value of `useWalletService`, it must throw an error. 

>[!NOTE]
>Decision Call: Should the `execute` method build **and** send the transaction? As an alternative we can have `execute` to build a transaction, a `send` method only to send the transaction built and `executeAndSend` to do it in sequence. 

### Transaction template builder

A builder class must assists the user to build a `TransactionTemplate` instance.

The `TransactionTemplate` class can be initiated from an object (the JSON template specified in this design) or it can be initialized empty and we add each input/output/action instruction with methods of the class.

```ts
class TransactionTemplateBuilder {
  addInstruction(
    instruction: BaseTemplateInstruction
  ): TransactionTemplateBuilder {}
  addInputOrOutput(
    io: InputOutputTemplateInstruction
   ): TransactionTemplateBuilder {}
  addAction(
    action: ActionTemplateInstruction
  ): TransactionTemplateBuilder {}
  addUtxoInput(
    position: number,
    fill: number,
    token?: string,
    authority?: `mint` | `melt`,
    address?: string,
    autochange?: boolean,
  ): TransactionTemplateBuilder {}
  addRawInput(
    position: number,
    tx_id: string,
    index: number,
  ): TransactionTemplateBuilder {}
  addTokenOutput(
    position: number,
    amount: number,
    token?: string,
    authority?: `mint` | `melt`,
    address?: string,
    check_address?: boolean,
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
  build(): TransactionTemplate {}
  export(): BaseTemplateInstruction[] {}
}
```

- `addInstruction`: add a general instruction as an object literal
- `addInputOrOutput`: add a general input or output as an object literal
- `addAction`: add a general action as an object literal
- `addUtxoInput`: add the specific UTXO input instruction
- `addRawInput`: add the specific Raw input instruction
- `addTokenOutput`: add the specific Token output instruction
- `addDataOutput`: add the specific Data output instruction
- `addRawOutput`: add the specific Raw output instruction
- `addShuffleAction`: add the specific Shuffle action instruction
- `addFillChangeAction`: add the specific FillChange action instruction
- `build`: build and returns a `TransactionTemplate` instance
- `export`: returns an array of instructions added to the builder in order of addition

### Usage example

The following template will mint 0.01 TST, generate another mint authority to the wallet and it will have a 'foobar' data output as the first output.
The mint deposit inputs are manually added to the transaction and all outputs are shuffled except for the data output at position 0.

```json
[
	{ "type": "input/utxo", "position": -1, "fill": 2, "token": "00" },
	{ "type": "input/utxo", "position": -1, "fill": 1, "token": "TST", "authority": "mint" },
    { "type": "output/token", "position": -1, "token": "TST", "amount": 1 },
    { "type": "output/token", "position": -1, "token": "TST", "authority": "mint", "amount": 1 },
    { "type": "action/shuffle", "target": "outputs" },
    { "type": "output/data", "position": 0, "data": "foobar" },
]
```

This same transaction template instructions set could be built as:

```ts
const ttInstructions = new TransactionTemplateBuilder()
  .addUtxoInput(-1, 2)
  .addUtxoInput(-1, 2, "TST", "mint")
  .addTokenOutput(-1, 1, "TST")
  .addTokenOutput(-1, 1, "TST", "mint")
  .addShuffleAction("outputs")
  .addDataOutput("foobar")
  .export();
```

We can get a transaction template instance from the instructions set by the `build` or by calling the transaction template constructor directly with the transaction template instructions:

```ts
const tt = new TransactionTemplate(ttInstructions);
```

Then we can get the designed transaction by calling the `execute`:

```ts
const transaction = tt.execute(myWalletFacade);
```

## Interpreters

A class which responsibility is to read the sequence of instructions to create a transaction using a determined type of wallet.

```ts
interface TemplateInstructionInterpreter {
  createTx(tt: TransactionTemplate): Transaction;
}
```

We have two instances of interpreters:
- WalletFacadeTemplateInterpreter
- ServiceTemplateInterpreter

## Lib land

Beyond the wallet API we can use any utils or helper available to support the interpreter in its mission.

For now we don't need to create anything. However, it worths to mention the `IHathorWallet` doesn't conveys all methods available on both wallets. Just to reference one, we can take the `getUtxos` which is implemented in both wallets but is not present at `IHathorWallet`. Not to mention the lack of type inference at wallet facade which makes the development experience poorer.

>[!NOTE]
>Decision Call: Should we use this implementation as an opportunity to improve the lib in the topics mentioned?

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The sequence of operations for the template is very straightforward, we choose the inputs, then we form the output array and once the transaction is formed we can perform actions on it to finalize the transaction instance.

## UTXO Input Instruction

If the instruction type is `input/utxo` we need to use the wallet's API to query for UTXOs, however token and authority requires a specialized usage of the APIs for one or the other, therefore we can optimize implementation for either:
- Token selection, or
- Authority selection

### Token selection

After query for UTXOs then we add them to the inputs array at the defined position.

	The addition must happen as a merge of arrays.

If any change is required and `autochange` is `true` or missing we add them to the outputs.

	We first create a token ouput instruction and then add it to the last position of outputs array.

#### Sample

The following instruction will find unlocked UTXOs that completes at least 0.10 TST tokens from address `Wtest` and add them to the inputs, also add change output to outputs if necessary.

```json
{
  "type": "input/utxo",
  "token": "TST",
  "fill": 10,
  "address": "Wtest",
}
```

This translates to the following call to `getUtxos`:

```ts
wallet.getUtxos({
  tokenId: "TST",
  addresses: ["Wtest"],
  totalAmount: 10,
});
```

### Authority selection

After query for UTXOs then we add them to the inputs array at the defined position.

	The addition must happen as a merge of arrays.

There is no output change for authority.

#### Sample

The following instruction will find 1 TST Mint authority UTXO and add them to the inputs.

```json
{
  "type": "input/utxo",
  "token": "TST",
  "fill": 1,
  "authority": "mint",
}
```

This translates to the following call to `getUtxos`:

```ts
wallet.getUtxos({
  tokenId: "TST",
  addresses: ["Wtest"],
  authority: "mint",
  count: 1,
});
```

## Shuffle Action

1. Select an array and create a new array of the same size
2. For each position generate a random index in the range of the array size that not repeat
3. Iterate over the new array and replace each element elements by the element of the selected array that represents the index value

## Fill Change Action

1. Iterate over all UTXOs inputs totaling the amount of each token
2. Iterate over all input instructions totaling the fill value of each token
3. Operate the difference between them
4. Create an output change if any difference is found
5. Position the output change at the end of outputs
