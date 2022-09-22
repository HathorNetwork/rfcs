- Feature Name: use_case_integration_best_practices
- Status: Draft
- Start Date: 2022-09-20
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

This document presents best practices for use cases that integrate with the Hathor Network, in order to improve the reliability of their operation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are multiple ways that a use case can integrate with Hathor, here is a basic architecture of a common integration.

- The use case has a software that runs operations offchain and is used by its customer.
- This software connects with a headless wallet, that executes operations in the Hathor network.
- The headless wallet is connected to a full node, that belongs to the Hathor peer-to-peer network.

<pre>
┌─────────────┐      ┌──────────┐                     ┌─────────────┐        ┌─────────────┐
│             │      │          │◀────────────────────│             │        │             │
│  Hathor     │      │ Headless │                     │  Use Case   │        │  Customer   │
│  Full Node  │◀────▶│  Wallet  │                     │  Software   │◀──────▶│             │
│             │      │          │────────────────────▶│             │        │             │
└─────────────┘      └──────────┘                     └─────────────┘        └─────────────┘
</pre>

## Run more than one node

Even though the architecture described above works fine for a simple use case, we strongly recommend any use case to run more than one node, in order to increase the reliability of the operation.

During normal network activity, any transaction will be propagated through all peers and they will get to the same consensus. However, there are some situations where this might not be true, e.g. if an attack is happening to a single peer. To improve the reliability of the operation at any moment, it's important that all use cases run more than one full node and they **can't be connected among them**, only to the rest of the network.

Let's assume an exchange wants to run nodes to identify deposits in the Hathor network, so a recommended approach for the integration would be to run at least two full nodes (node1 and node2), which are not connected between them (node1 must have node2 in the blacklist and vice versa). In that architecture, if any deposit is identified in node1, then the exchange must check that it's also a valid transaction in node2 and in one of the public nodes. With this approach, if one of the nodes is compromised and with a different consensus/state, an alert must be raised, so we must investigate what happened.

### Validate new transactions on more than one full node before accepting them

When running more than one node, any new transaction must be accepted only after validating that it's a valid transaction in all the nodes.

### Validate all your full nodes have the same best block

It's also important to regularly check if all nodes have the same best block. If they have different best blocks, this might be a clue that one of the nodes could be a target of an attack because is running a separate fork of the blockchain.

### Recommended architecture

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

## Peer-id

The peer-id is a unique identifier of your full node in Hathor's p2p network. Even though it does not contain any private key information, keeping it secret is a security measure that prevents attackers from targeting directly your full nodes.

Because of that, it's important to never share it in public chats, only in private groups with the Hathor team. If you think your peer-id has been exposed, you should generate a new peer-id and replace the exposed ones.

## Be alert for weird behavior

### Correctly validate transactions

The transactions in the Hathor network have many fields that must be checked to guarantee that a transaction is valid for your use case. For more details about the fields of a transaction, check the [Transaction Anatomy RFC](https://github.com/HathorNetwork/rfcs/blob/master/text/0015-anatomy-of-tx.md).

#### Version

Transactions have `version=1`, blocks have versions `0` or `3`, and token creation transactions have `version=2`.

#### Voided

A voided transaction is **not** a valid transaction. You must only accept transactions that have `is_voided=false`.

#### Outputs

The outputs contain the fields used to identify that the funds belong to your wallet.

##### Token

The output has the token information, so you can check if it has the expected tokens. For example, exchanges expect the deposits to be of HTR, so `token = '00'` and `token_data=0`.

##### Value

The value of the output is an integer, so `1000` means `10.00`.

##### Address

The output address must be one of your wallet's addresses.

##### Timelock

The output must have a timelock, and if it's bigger than the current timestamp, you won't be able to spend this output until this moment is reached. Because of that, it's essential to validate that `timelock=null` before validating this transaction.

#### Number of confirmations

Some use cases might handle transactions with huge amounts, so it's essential to wait for some blocks to confirm the transaction before accepting it as a valid one. The more blocks confirm a transaction, the more guarantee there is that this transaction won't become invalid in the future.

### Check if an unusual amount of deposits or withdrawals are being made

Many use cases have withdrawals/deposits in their set of features through blockchain transactions. It's important to be alerted in case any unusual amount of deposits/withdrawals are being made because this could be caused by an attack.

For that situation, it's crucial to have an easy way to block specific accounts that have unusual behavior and might be part of an attack.

### Check if one of your full nodes gets out-of-sync

One other relevant aspect of full node to always check for weird behavior is the sync among them. We recommend use cases to regularly validate that all their full nodes are in sync among them and in sync with at least one public node as well.

This validation is important to guarantee the node is not isolated from the rest of the network with a fork of the blockchain.

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