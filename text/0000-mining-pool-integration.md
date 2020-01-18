- Feature Name: mining_pool_integration
- Start Date: 2020-01-18
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

This document describes the integration of a mining pool with Hathor, both mining only HTR and mining HTR+BTC using merged mining.

# Services
[services]: #services

These are the services that will communicate and make things happen:

1. **Mining Pool Service**, which is the backend software used by the mining pool.
2. **HTR Wallet**, used to generate addresses and send/receive transactions for Hathor.
2. **BTC Wallet**, used to generate addresses and send/receive transactions for Bitcoin.
3. **HTR Full-node**, which connects to the Hathor's p2p network.
3. **BTC Full-node**, which connects to the Bitcoin's p2p network.

These services communicate directly accessing their HTTP APIs.

# High Level Design
[high-level-design]: #high-level-design

The high level design is almost the same for both mining HTR only and mining HTR+BTC. So, it just says _Wallet_ and _Full-node_, without any specification of HTR or BTC.

Initially, the _Mining Pool_ generates a new job and distributes it to the miners. To generate the job, the _Mining Pool_ gets an address from the _Wallet_ and gets a block template from the _Full-node_.

When a miner submits a solution to the _Mining Pool_, this solution may either have found a block or not. If it does, the block is submitted to the _Full-node_.

The _Mining Pool_ tracks all submitted blocks through the _Full-node_. When a block is mature, the _Mining Pool_ rewards are distributed to the miners (notice that it is not a payout). A block is mature when its reward can be spent.

When a miner requests a payout, the _Mining Pool_ adds it to a list of pending payouts, so it can execute multiple payouts in a single transaction.

When executing the payouts, the _Mining Pool_ generates a transaction using the _Wallet_ and propagates it to the network using the _Full-node_.

The _Mining Pool_ tracks all payout transactions using the _Full-node_. When a transaction is confirmed, the _Mining Pool_ marks the payout as confirmed.

The following diagram summarizes the communication between the services. The arrows indicate how the data flows (and not who initiated the request). Notice that the diagram is a simplified view of the flow and some steps have an external trigger. For instance, the "Get block status" is executed many times until the block is mature; the "Send payout tx" is executed only when the _Mining Pool_ is executing the payouts; and the "Get tx list" is executed many times to check whether the payout transaction is confirmed.

<pre>
                                     ┌─────────────────┐                             ┌───────────────┐
┌───────────┐                        │                 │◀───2. Block template────────│               │
│           │◀──3. Send job──────────│                 │                             │               │
│   Miner   │                        │   Mining Pool   │────5. Propagate block──────▶│   Full-node   │
│           │───4. Submit solution──▶│                 │                             │               │
└───────────┘                        │                 │◀───6. Block status──────────│               │
                                     └─────────────────┘                             └───────────────┘
                                            ▲   │  ▲
                                            │   │  │
                                            │   │  │
                                            │   │  │                               ┌────────────┐
                                            │   │  └───1. Wallet address───────────│            │
                                            │   │                                  │            │
                                            │   └──────7. Send payout tx──────────▶│   Wallet   │
                                            │                                      │            │
                                            └──────────8. Tx list──────────────────│            │
                                                                                   └────────────┘
</pre>

# Low Level Design
[low-level-design]: #low-level-design

This document only details the communication between (i) the _Mining Pool_ and the _HTR Full-node_ and (ii) the _Mining Pool_ and the _HTR Wallet_.

## Mining Pool <> HTR Full-node

Getting a template and submitting a block is different for mining HTR only and mining HTR+BTC.

Checking a block's status is the same for both mining types. You can use `/v1a/transaction?id=BLOCK_HASH`. It returns all pieces of information about either a block or a transaction.

__Note__ Merge Request 359 includes the block's height in the `/v1a/transaction` API. Check it out [here](https://gitlab.com/HathorNetwork/hathor-python/merge_requests/359/diffs#3069a669581264882591ff171b48e2db0efc3766_222_224).

__TODO__ The number of confirmation blocks for both blocks and transactions must be included into the `v1a/transaction` API.

### HTR Only

For HTR only, you can use the `/v1a/mining/?address=YOUR_ADDRESS` API. The GET method returns a block template to be mined, while the POST method submits a new block. Check out their docs [here](https://docs.hathor.network/#operation/mining_get) and [here](https://docs.hathor.network/#operation/mining_post), respectively.

### HTR+BTC

For HTR+BTC, you can use the `createauxblock` and `submitauxblock` APIs. These APIs are under development, and their returns will be fully compatible with the well-known APIs from other coins that accept merged mining. This document will be updated as soon as they are available.

## Mining Pool <> HTR Wallet

You can use the `hathor-wallet-headless` to create a headless wallet that is controlled through its APIs. Check out the repository [here](https://github.com/HathorNetwork/hathor-wallet-headless).

To generate an address, you can use the `/address` API. If called multiple times, it will return the same address unless you mark the address as used or a transaction arrives to the current address. It will return error if you try to generate more than 20 empty addresses, i.e. addresses without any transaction.

To get the transaction list, you can use `/tx-list` API. The list is sorted by timestamp, from the most recent to the last, and it includes only basic information about each transaction.

To get all pieces of information about a transaction, you should directly request from the _Full-node_ using the `/v1a/transaction` API. Check out its documentation [here](https://docs.hathor.network/#operation/transaction).

To send a transaction, you can use the `/simple-send-tx` API. It let you specify the destination address and amount to be sent. Then, it automatically select the inputs and send the transaction.

The API's full documentation is available in the repository.

# Tips & Tricks for Testing
[tips-and-tricks-for-testing]: #tips-and-tricks-for-testing

## Running your full-node

When developing the integration, you should test either using Hathor's testnet or running a solo full-node. Both choices are similar to safely test payouts. You should **never** test your integration using our mainnet because you can lose your real tokens.

You should run your full-node with the parameters `--status 8000` and `--wallet-index`. They will enable the full-node's API that you will use in your integration. This API is also used by `hathor-wallet-headless`.

### Hathor's testnet

When running the full-node, include the `--testnet` parameter. So, it will connect to Hathor's testnet where the mining difficulty is smaller.

To connect to the Hathor's testnet: `hathor-cli run_node --status 8000 --wallet-index --testnet`.

### Running a solo full-node

In this case, your full-node will not connect to any network. So, you will be the only one running miners. You can quickly run a CPU miner using the full-node's built-in CPU miner.

To run a solo full-node: `hathor-cli run_node --status 8000 --wallet-index --skip-bootstrap --allow-mining-without-peers`.

To run the full-node's built-in CPU miner: `hathor-cli run_miner http://localhost:8000/v1a/mining/`.

__TODO__ Add `--skip-bootstrap` to `hathor-core`.

## Converting weight to difficulty

To convert block's weight in the equivalent difficulty, you can use the formula: `diff = 2^(weight - 32)`. It works for any value of `weight`, even if it is below 32.

To convert difficulty to weight, you can use the formula: `weight = log2(diff) + 32`. It also works for any value of `diff`.

In Hathor's full node, we have it implemented and you can do the following:

```
from hathor.merged_mining.coordinator import diff_from_weight
diff = diff_from_weight(weight)
```

# Resources
[resources]: #resources

- Hathor Full-node: https://github.com/HathorNetwork/hathor-core/
- Hathor Wallet Headless: https://github.com/HathorNetwork/hathor-wallet-headless/
- Hathor Wallet Library: https://github.com/HathorNetwork/hathor-wallet-lib/
- Hathor API Documentation: https://docs.hathor.network/
- Hathor Merged Mining with Bitcoin RFC: https://gitlab.com/HathorNetwork/rfcs/blob/master/text/0006-merged-mining-with-bitcoin.md
