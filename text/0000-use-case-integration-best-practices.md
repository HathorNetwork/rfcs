- Feature Name: use_case_integration_best_practices
- Status: Draft
- Start Date: 2022-09-20
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

This document presents security best practices for use cases that integrate with the Hathor Network, in order to improve the reliability of their operation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are multiple ways that a use case can integrate with Hathor, here is a common architecture compliant to our recommendations

- The use case has a software that runs operations offchain and is used by its customer.
- This software connects to a headless wallet which communicates with a full node. The full node is connected to the Hathor's p2p network.
- The headless wallet will handle all events from the full node to keep its balance and transaction history updated. The headless software have APIs to manage a wallet, e.g. get addresses, create and push transactions to the network, create new tokens, and some other features described in the project [repository](https://github.com/HathorNetwork/hathor-wallet-headless).

<pre>
                       ┌─────────────┐      ┌──────────┐                     ┌─────────────┐        ┌─────────────┐
                       │             │      │          │◀────────────────────┤             │        │             │
                       │  Hathor     │      │ Headless │                     │  Use Case   │        │  Customer   │
                       │ Full Node 1 │◀────▶│  Wallet  │                     │  Software   │◀──────▶│             │
                       │             │      │          ├────────────────────▶│             │        │             │
                       └────┬────────┘      └──────────┘                     └─────────────┘        └─────────────┘
                            │
┌────────────┐              │
│            │◀─────────────┘
│  Rest of   │
│  Network   │
│            │◀─────────────┐
└────────────┘              │
                       ┌────┴────────┐
                       │             │               
                       │  Hathor     │
                       │ Full Node 2 │
                       │             │
                       └─────────────┘

</pre>

## Run more than one node

We strongly recommend use cases to run two or more full nodes as a protection to direct attacks to their full nodes.

Use case's full nodes **should not** be connected among them. This is important to mitigate some attack vectors. Remember that the transactions will be propagated by the p2p network and all use case's full nodes will receive the transactions eventually during normal network activity.

### Validate new transactions on more than one full node before accepting them

Let's assume an exchange wants to run nodes to identify deposits in the Hathor network, so a recommended approach for the integration would be to run at least two full nodes (node1 and node2), which are not connected between them (node1 must have node2 in the blacklist and vice versa). In that architecture, if any deposit is identified in node1, then the exchange must check that it's also a valid transaction in node2 and in one of the public nodes. In this approach, if an attacker successfully compromises one of your full nodes, your validation would fail and the deposit will not be accepted.

### Validate all your full nodes have the same best block

Use cases should regularly check whether the best block is the same on all their full nodes. If full node have different best blocks, the validation must be done again some seconds later because this might happen depending on the network block propagation time. If the differente continue, the nodes might be under attack and the use case should consider blocking deposits.


## Peer-id

The peer-id is a unique identifier of your full node in Hathor's p2p network. You must keep your peer-id secret to prevent attackers from directly targeting your full nodes. Do not tell anyone your peer-ids, and do not publish them on public channels. If you think your peer-id has been exposed, you should generate a new peer-id and replace the exposed ones.

## How to validate a new transaction

The transactions in the Hathor network have many fields that must be checked to guarantee that a transaction is valid for your use case. For more details about the fields of a transaction, check the [Transaction Anatomy RFC](https://github.com/HathorNetwork/rfcs/blob/master/text/0015-anatomy-of-tx.md).

- [Version](#version)
- [Voided state](#voided-state)
- [Outputs](#outputs)
- [Number of confirmations](#number-of-confirmations)

### Version

Version identifies the type of vertex. The possible choices are:

- Block: 0
- Regular Transaction: 1
- Token Creation Transaction: 2
- Merged Mined Block: 3

Depending on your use case you must accept one or more types.

### Voided state

A voided transaction is cancelled and should **never** be accepted. You must validate that the transaction is not voided asserting that `is_voided == false`.

### Outputs

Hathor supports multi-token transactions. You must confirm that your outputs are correctly placed as follows:

- Token id is correct. If you accept only HTR token, token id must be `"00"`.
- Token data is equal to `0`, if it's HTR token.
- Value matches the expected value. Note that it is an integer and `12.34` is represented by `1234`.
- Timelock must be `null`; otherwise your funds might be locked.

### Number of confirmations

Some use cases might handle transactions with huge amounts, so it's essential to wait for some blocks to confirm the transaction before accepting it as a valid one. The more blocks confirm a transaction, the more guarantee there is that this transaction won't become voided in the future. As a reference, Bitcoin's use cases usually require six confirmations before accepting a new deposit.

## Be alert for weird behavior

### Check if an unusual amount of deposits or withdrawals are being made

Many use cases have withdrawals/deposits in their set of features through blockchain transactions. It's important to check for unusual levels of deposits and withdrawals because this could be caused by an attack.

For that situation, it's crucial to have an easy way to block specific accounts that have unusual behavior and might be part of an attack. You might also consider to limit the number of operations a user can do in a time window.

This validation and user throttling must be done in the use-case software application.

### Check if one of your full nodes gets out-of-sync

Always check for weird behavior in the synchronization among full nodes. We recommend use cases to regularly validate that all their full nodes are in sync among them and in sync with at least one public node as well.

This validation is important to guarantee the node is not isolated from the rest of the network with a fork of the blockchain.

Besides that, it's also important to validate that the timestamp of the best block of the node is recent, which means that the node's blockchain is not halted in the past.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Run a full node with a blacklist of peer-ids

Following the recommended architecture, you will need to run more than one full node and they shouldn't be connected among them. To achieve that, each of your nodes must have a blacklist of peer-ids, containing the ids of the other nodes.

For example, you run node1 (peerid1), node2 (peerid2), and node3 (peerid3). The node1 should have peerid2 and peerid3 in the blacklist, node2 should have peerid1 and peerid3 in the blacklist, and node3 should have peerid1 and peerid2 in the blacklist.

To create this blacklist in the full node there are some possible approaches:

### CLI command

If you use the full node parameters directly in the command line, then you should add:

```
--peer-id-blacklist peerid1 peerid2
```

### Environment variables

If you use the full node parameters using env vars, then you should add:

```
export HATHOR_PEER_ID_BLACKLIST=[peerid1, peerid2]
```

### API

There is also an API to add a peer-id to the blacklist while the node is running, however this is not recommended because if you restart your node you will lose this blacklist.

```
POST to /v1a/p2p/netfilter

{
    "chain": {
        "name": "post_peerid",
    },
    "target": {
        "type": "NetfilterReject",
        "target_params": {}
    },
    "match": {
        "type": "NetfilterMatchPeerId",
        "match_params": {
            "peer_id": "peerid1"
        }
    }
}
```