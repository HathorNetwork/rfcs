- Feature Name: nano_contracts
- Start Date: 2021-06-14
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

Nano Contracts enable users to create distributed programs to execute complex operations. It is a simplified version of Ethereum's Smart Contracts.

# Motivation
[motivation]: #motivation

Nano Contracts enable the community to create decentralized applications where the rules are immutable and users can interact with them.

It is expected that this new feature will enable other use cases to migrate to Hathor.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Hathor currently implements the UTXO model which means each transaction has one or more outputs and each output can be spent just once. Any attempt to spent an output more than once is considered a double spending attempt and the consensus algorithm will ensure that at most one of the attemps will be executed (and all the others will be voided).

Each output controls some funds (e.g. 1 HTR) and has a script with the rules to spend it (i.e. transfer the funds). A very common script is the Pay-to-Pubkey-Hash (P2PH) which means the ouput can only be spent by the owner of the private key associated with it. Another common script is the Pay-to-Script-Hash (P2SH) which enables multisig among other rules. The multisig rule requires M out of N valid signatures to spend the output.

It is called the UTXO model because the funds are "stored" in Unspent Transaction Outputs that can be used only once. So, one can say that a transaction spends some utxos and create new ones. In other words, the transaction spent the inputs (the utxos of other transactions) and create new outputs that will be used in the future.

Wallets are managers of private keys and utxos. When Alice sends 10 HTR to Bob, Alice's wallet creates a transactions that spends one of Alice's utxos and create a new output that can be spent only by Bob. Let's say Alice has a total of 14 HTR split in two utxos (one with 6 HTR and the other with 8 HTR), so Alice's wallet must create a transaction with two inputs (one for each utxo) and two outputs. The first output is for Bob with 10 HTR and the second output is for herself with a change of 4 HTR. After the transaction is confirmed by the network, Alice will have spent her two utxos that will never be used again, and she will have created two new utxos. Transaction 3 in the image below is Alice's transaction.

<pre>
     ┌───────────────────────────────────┐           ┌───────────────────────────────────┐
     │           Transaction 1           │           │           Transaction 3           │
     └┬────────────────┬────────────────┬┘           └┬────────────────┬────────────────┬┘
      │┌──────────────┐│┌──────────────┐│             │┌──────────────┐│┌──────────────┐│
◀─────┼┼─  Input 1    │││   Output 1  ◀┼┼─────────────┼┼─  Input 1    │││   Output 1   ││
      │└──────────────┘││    6 HTR     ││             │├──────────────┤││    10 HTR    ││
      │                │└──────────────┘│      ┌──────┼┼─  Input 2    ││└──────────────┘│
      │                │┌──────────────┐│      │      │└──────────────┘│┌──────────────┐│
      │                ││   Output 2   ││      │      │                ││   Output 2   ││
      │                ││    1 HTR     ││      │      │                ││     4 HTR    ││
      │                │└──────────────┘│      │      │                │└──────────────┘│
      │                │┌──────────────┐│      │      └────────────────┴────────────────┘
      │                ││   Output 3   ││      │
      │                ││    2 HTR     ││      │
      │                │└──────────────┘│      │
      └────────────────┴────────────────┘      │
                                               │
     ┌───────────────────────────────────┐     │
     │           Transaction 2           │     │
     └┬────────────────┬────────────────┬┘     │
      │┌──────────────┐│┌──────────────┐│      │
◀─────┼┼─  Input 1    │││   Output 1   ││      │
      │└──────────────┘││    4 HTR     ││      │
      │                │└──────────────┘│      │
      │                │┌──────────────┐│      │
      │                ││   Output 2  ◀┼┼──────┘
      │                ││    8 HTR     ││
      │                │└──────────────┘│
      │                │┌──────────────┐│
      │                ││   Output 3   ││
      │                ││    8 HTR     ││
      │                │└──────────────┘│
      └────────────────┴────────────────┘
</pre>

In the image above, we say that __Transaction 3__ is spending outputs from __Transaction 1__ and __Transaction 2__. More specifically, we say that __Input 1__ of __Transaction 3__ is spending the first output of __Transaction 1__, while __Input 2__ of __Transaction 3__ is spending the second output of __Transaction 2__.

Nano Contracts empowers transactions with even more complex rules. These other rules may need extra pieces of information besides the amount of funds.

## Features

Nano Contracts have the following highlighted features:

1. Methods that will be used to validate a transaction.
2. Custody of tokens.
3. Send and receive tokens from UTXOs.
4. Execute methods in other Nano Contracts.
5. Get information from the real world through Oracles.

## Nano Contracts

Nano Contracts are special vertices in the DAG. Each Nano Contract has a state and a list of methods. All states have a special variable that stores the amount of tokens controlled by the Nano Contract.

The public methods must raise an exception if the transaction is invalid. Otherwise, the transaction is valid and the method might have different types of return. The transaction will be executed only, and only if, all methods are executed without any exception. Notice that a transaction might have both regular UTXOs and calls to methods.

Methods will be executed by the Hathor Virtual Machine (HVM) which has only operations that run in constant time and no loops are allowed.

The state is made of variables that are accessed and modified by the methods. It means that the order of execution of the calls to the methods matter and might affect the final state of the contract.

In order to deposit tokens into a Nano Contract, one must call a method in an output. The method will decide whether it accepts or not the deposit. If it is accepted, the output's amount will be deposited in the Nano Contract.

In order to withdraw tokens from a Nano Contract, one must have one input calling a method that will return a `TxInput` object with the amount of tokens authorized for withdrawal.

Notice that one transaction can have multiple deposits and withdrawals at once. Besides that, all transactions still have to have their balances verified, i.e., the sum of the inputs must equal the sum of the outputs (except when authorities are used).

## Examples of Use

### HathorSwap

(todo)

### Bettin between two people

Betting between two people is simple.

```python
class SimpleBet(NanoContract):
    def __init__(self):
		self.bets = {}

	def create_bet(self, tx) -> None:
		self.assertTwoDeposits(tx)

	def deposit(self, txin: TxInput) -> bool:
		pass

	def withdraw(self, result) -> bool:
		checksig(self.oracle_pkh, result)
		:
		(todo)
		:

```

```
TxOutput[0].script =
    checksig(oracle, results)
	if results == '1x4':
		return checksig(userA, tx)
	else:
		return checksig(userB, tx)
```

### Group Betting

Betting using data from the real world requires an Oracle. Imagine a predict-a-score game where any person can place a bet and the funds will be split among the winners. In this case, each participants will place a transfer to the Nano Contract. At the end of the game, an Oracle will provide the game results to the Nano Contracts and the winners will be able to collect their prize. Such Nano Contract would have the following features:

```python
class Bet(NanoContract):
    def __init__(self, oracle_pkh: PubKeyHash):
	    self.bets: Dict[str, List[Tuple[Address, Amount]]]
		self.total: Amount = 0
		self.oracle_pkh: PubKeyHash = oracle_pkh

	def make_a_bet(self, address: Address, amount: Amount, score: str):
		if timestamp > '2021-01-01 17:00':
			fail('cannot place bets after 2021-01-01 17:00')
		self.bets[score].append((address, amount))
		self.total += amount

	def resolve(self, result):
		assert checksig(self.oracle_pkh)
		winners = sum(amount for _, amount in self.bets.items())
		for address, amount in self.bets[result]:
			send_funds(address, self.total * amount / winners)
```

The example above needs a `for` loop which runs in O(n) and gets slower and slower as more and more bets are placed.

## Hathor Virtual Machine

- All instructions must be O(1) and no loops are allowed.
	- Example: `x in mylist` is not allowed because it is O(n).
- Maximum number of instructions per method?
- Should we charge a fee per instruction?
- Enumerate variables and associate each one with a register.
    - Maximum number of registers?
- Flags
    - N: Negative
	- Z: Zero
	- C: Carry or unsigned overflow
	- V: Signed overflow

### Instruction Set

Maybe we should get inspiration in [Python's bytecode](https://docs.python.org/3/library/dis.html#bytecodes).

1. `JMP`: Unconditional jump (always relative).
2. `JLE`: Jump if less than or equal. (`(Z == 1) || (N != V)`)
3. `JLT`: Jump if less than. (`N != V`)
4. `JGE`: Jump if greater than or equal. (`N == V`)
5. `JGT`: Jump if greater than. (`(Z == 0) && (N == V)`)
6. `JNE`: Jump if not equal. (`Z == 0`)
7. `JEQ`: Jump if equal. (`Z == 1`)
8. `CMP`: Compare two values and update the flags.

### Examples

- `a += 1` will be compiled to `INC reg[A]`, where `regA` is the register associated with variable `a`.
- `a = a + b` will be compiled to `ADD reg[A] reg[A] reg[B]`.

```
if a > 5:      |   CMP reg[A] const[5]
               |   JLE mark1
    a += 1     |   INC reg[A]
               |   JMP mark2
else:          | mark1:
    a -= 1     |   DEC reg[A]
               | mark2:
```

The assembler will translate the assembly code into object files.

## Matching for Transactions

- Regexp for transactions
  - Index: `txin[0].token_idx == 0`
  - Any:   `txin[?].token_idx == 1`
  - All:   `txin[*].token_idx == 0`
  - `txout[0].amount < 500`
  - `txout[?].token_idx == 1`
  - `tokens[1] == "HTR"`
  - `txin.length == 2`
  - `txout.length == 3`
  - `tokens.length == 1`

For example, `type == Transaction && tokens.length == 0 && txin.length == 1 && txout.length <= 2 && txin[*].authority == False && txout[*].authority == False`.

### Consensus & Order of Execution

Order of execution might be: first by the height of the block and then by the position of the transaction in the DAG. The position of the transaction might be ambiguous about a DAG is only partially ordered so it might require further investigation.

## Overall Costs & Fees

As Nano Contracts will not be turing complete and the method will be fast to process, its economics might be similar to custom tokens. In other words, no fees will be chaged to execute methods in Nano Contracts and the transaction's PoW will be the spam control mechanism.

## Generation of random values

Even though it might be useful, it is far from simple because all full-nodes must generate exactly the same value to be able to verify the transaction. As full-nodes must generate the same value, it might be a predictable value which is a deal breaker for many uses.

One idea might be to use the hash of the first block to confirm the transaction as a random seed (e.g. `sha256d(block.hash)`). If the hashrate is big enough and decentralized, it might be accept as random enough. This would prevent the full-node from executing some transactions before it gets confirmed by a block and requires further analysis of impact.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

# Drawbacks
[drawbacks]: #drawbacks

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Nano Contracts with UTXO only

In the UTXO model, the P2SH Script is a clever approach to create Nano Contracts because they have an address associated to each script. The methods of the Nano Contracts would be leafs in a Merkle Tree and the hash of the root will be used to generate the P2SH Address. The Nano Contracts would have to be published off-chain and users would interact with a Nano Contract spending a utxo and putting the desired method (and merkle path) in the input.

These are some disadvantages to this approach that made me abadon it: (i) each transaction will replicate the code to be executed, requiring more storage and bandwidth, (ii) if many users would like to interact with the same Nano Contract at the same time, their transactions might be in conflict and some of them will have to regenerate their transactions, (iii) the funds of a Nano Contract will be split among many utxos and the wallet will have to select the right utxos when building a new transaction, and (iv) there's no way to store extra pieces of information.

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For protocol, network, algorithms and other changes that directly affect the
  code: Does this feature exist in other blockchains and what experience have
  their community had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other blockchains, provide readers of your RFC with a fuller
picture. If there is no prior art, that is fine - your ideas are interesting to
us whether they are brand new or if it is an adaptation from other blockchains.

Note that while precedent set by other blockchains is some motivation, it does
not on its own motivate an RFC. Please also take into consideration that Hathor
sometimes intentionally diverges from common blockchain features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be
and how it would affect the network and project as a whole in a holistic way.
Try to use this section as a tool to more fully consider all possible
interactions with the project and network in your proposal. Also consider how
this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
