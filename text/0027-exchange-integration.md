- Feature Name: exchange_integration_guide
- Status: Draft
- Start Date: 2020-03-30
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>, Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

This document presents different ways to integrate Hathor with an exchange.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are multiple ways to integrate Hathor with an exchange. So, let's explore what are the choices and their pros and cons. First of all, here is the basic architecture of the integration.

<pre>
┌─────────────┐      ┌──────────┐                     ┌─────────────┐        ┌─────────────┐
│             │      │          │◀────────────────────│             │        │             │
│  Hathor     │      │  Wallet  │                     │  Exchange   │        │  Customer   │
│  Full Node  │◀────▶│          │                     │  Software   │◀──────▶│             │
│             │      │          │────────────────────▶│             │        │             │
└─────────────┘      └──────────┘                     └─────────────┘        └─────────────┘
</pre>

## Environments

We strongly recommend the exchange to have three separate and isolated environments: (i) testing, (ii) staging, and (iii) production. Each environment should have its own full-node and wallet services.

The testing environment should be connected to the testnet and will be used for any development.

The staging environment should be connected to the mainnet and will be used for running QA tests before any major update. It is important for testing new releases of Hathor's full-node and wallet before rolling it out to all users.

Finally, the production environment should be connected to the mainnet and will be used by most users. The staging and the production wallet services usually share the same seed.

## Exchange's own wallet **vs** Hathor's wallet

One of the first decisions is what wallet will be used in the integration: either Hathor's wallet or the exchange's own wallet.

Some exchanges have developed their own wallet with proprietary technology and rather use it. In this case, the exchange must develop an integration between Hathor's full-node and its wallet. The details of such integration will not be covered in this document but please send us a message and we will help you all the way.

The simplest way is to use [Hathor's headless wallet](https://github.com/HathorNetwork/hathor-wallet-headless/), which is a full featured wallet that is easily controlled by an HTTP API. This wallet handles all the communication with the full-node and the exchange just has to use the wallet's API. Check out the API documentation [here](https://wallet-headless.docs.hathor.network/).

## Address Management

Each address of the wallet is associated with a derivation path and an index. We highly recommend the exchanges to store the address and the derivation path for debugging purposes. For more information, see the [BIP-0044](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki).

### GAP Limit

When the wallet is started, it loads the transactions from the full-node starting at index 0 and increasing one by one. When the wallet finds `GAP_LIMIT` empty addresses, it stops the scan. The default value for the gap limit is 20 but you might need to adjust it depending on your integration.

If the `GAP_LIMIT` is not properly set, the wallet might happen to stop the scan and do not find one or more transactions. Let's say `GAP_LIMIT = 200`, and let's enumerate the users by the index of their addresses. So, user 0 is the user whose address has index 0, user 15 is the user whose address has index 15, and so on. Let's say user 50 has made a deposit but no users between 51 and 250 has made any deposit. In this case, even if a deposit is made to address 251, the wallet won't find it because it will stop scanning at index 250.

We recommend a minimum value of 5,000 for exchanges. One should never just put a high number as `GAP_LIMIT` because it slows down the wallet initialization and also increases memory consumption.

For more information, click [here](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki#Address_gap_limit).

### Number of addresses per user

Exchanges might support either only one HTR address per user or many addresses per user. Both cases can be implemented using Hathor's headless wallets, and the exchange is responsible for associating which addresses belongs to each user.

## Multiple wallets

Some exchanges have two or more wallets to increase safety. Running multiple wallets is as simple as starting multiple wallet services.

For example, one exchange might use one wallet for deposits and another one for withdraws. In this case, all funds sent to the deposits wallets will eventually be transferred to the withdrawal wallet.

## User deposits

The detection of deposits can be implemented using a polling strategy.

1. The user will see the address and send HTR to it;
2. There will be a polling to check any new transactions for the deposit wallet;
3. When a new transaction with an output for the user's address arrives, a new deposit is created with a pending status;
4. A new polling will start to check the height of the network to validate the minimum number of confirmations required to confirm the deposit;
5. Once the deposit is confirmed, the HTR of this transaction is transferred to the withdrawal wallet and the HTR is credited to the user's exchange account.

### User withdrawals

1. The user selects a destination address and amount.
2. After approved, the exchange sends a request to the wallet to transfer the funds.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Running a full-node

We recommend you to run the full-node from our [Docker Hub image](https://hub.docker.com/r/hathornetwork/hathor-core). Check out a `docker-compose.yml` example [here](https://github.com/HathorNetwork/hathor-core/issues/183).

Each full-node has a peer-id that uniquily identifies it in the p2p network. The exchanges should always create fixed peer-ids for its full-nodes (and never use random peer-ids).

Full-nodes in production environments should always run with RocksDB storage and should follow the minimum requirements of CPU, RAM, and storage.

The exchange can easily integrate the full-node in its monitoring and alerting systems using the parameter `--prometheus`.

### Full node custom configuration file

There are two configuration parameters that should be changed in the full node to support the `GAP_LIMIT` increase. To create a custom mainnet configuration file you must create a new configuration file (e.g. `myconf.py`):

```
from hathor.conf import mainnet as base # if you are using the mainnet
# from hathor.conf import testnet as base # if you are using the testnet
SETTINGS = base.SETTINGS._replace(
    # Maximum number of subscribed addresses per websocket connection
    WS_MAX_SUBS_ADDRS_CONN=200000,
    # Maximum number of subscribed addresses that do not have any outputs (also per websocket connection)
    WS_MAX_SUBS_ADDRS_EMPTY=40000
)
```

If this file is not in the path where python would search for modules, `PYTHONPATH` env var should be updated to include this path. Besides that, `HATHOR_CONFIG_FILE` environment variable must be set to load this configuration file.

Here is the docker cmdline to use a custom configuration file, where `myconf.py` is stored in the container folder `/hathor/data`:

```
docker run --env PYTHONPATH=/hathor/data --env HATHOR_CONFIG_FILE=myconf …
```

## Running a Hathor's headless wallet

We recommend you to run the wallet service from our [Docker Hub image](https://hub.docker.com/r/hathornetwork/hathor-wallet-headless). Check out a `docker-compose.yml` example [here](https://github.com/HathorNetwork/hathor-wallet-headless/issues/66).

You can also manually install it from its [GitHub repository](https://github.com/HathorNetwork/hathor-wallet-headless).

The documentation of the wallet's API is available [here](https://wallet-headless.docs.hathor.network/).

We recommend you to use a different API Key for each wallet. The API keys can be prefixed with the environment of each wallet. For example, a wallet in the testing environment should be prefixed with `testing-`, while a wallet in the production environment should be prefixed with `production-`. It prevents operational errors such as sending the request to the wrong wallet.

We also recommend you to enable the `confirmFirstAddress` configuration. This requires all requests to come with the `X-First-Address: {first-address}` header. It is also important to prevent operational errors.

### Examples

1. `src/config.js`

```
module.exports = {
  http_bind_address: 'localhost',
  http_port: 8002,
  http_api_key: 'production-YOUR_API_KEY',
  network: 'mainnet',
  server: 'URL_TO_YOUR_FULLNODE',
  // Wallet Config
  seeds: {
    deposit: 'PUT_YOUR_SEED_HERE',
  },
  gapLimit: 5000,
  confirmFirstAddress: true,
};
```

2. Run two headless daemons (one for deposit and one for address) and start the wallet.
```
$ curl -X POST --data "wallet-id=idDeposit" --data "seedKey=deposit" http://{headless-wallet}/start
{
  "success":true
}
```

3. Get an address for each user

```
$ curl -H "X-Wallet-Id: idDeposit" http://{headless-wallet}/wallet/address?mark_as_used=true
{
  "address": "WPHkmTUAaLJyjxQbpaPnpupmB4KqaQsHkt"
}
```

You must always confirm that the returned address has not been assigned to another user. Assigning the same address to two different users is a **critical** problem.

This request returns a new address until the GAP limit is reached. After that, it returns **always** the same address.

The recommended process to generate new addresses is:

a) Request a new address from the wallet using the query param `mark_as_used=True`.
b) Confirm that this address has never been assigned to a user. The exchange's database is usually queried for this.
c) Assign the address to a user.

4. Polling to check new transactions on the deposit wallet

The transaction history has transactions that were sent to this wallet and from this wallet, thus to guarantee that a new transaction is a new deposit it's important to see the address in the outputs and check if it's from one of the users.

This API used with the `limit` query parameter returns the last N transactions from this wallet, ordered by decreasing timestamp.

```
$ curl -H "X-Wallet-Id: idDeposit" http://{headless-wallet}/wallet/tx-history?limit=10
[{
  "tx_id": "0064584152bfd4a3ab6fb23ac35cec50c69b4ea2b66b63969e2b471f4d84b1f8",
  "version": 1,
  "weight": 8.000001,
  "timestamp": 1608218417,
  "is_voided": false,
  "inputs": [
    {
      "value": 12300,
      "token_data": 0,
      "script": "dqkUA3DKxP9CT64k9YyE2pB0NvuFtjeIrA==",
      "decoded": {
        "type": "P2PKH",
        "address": "WNzE2S6XskM6ynYWbKJHzHtQAw2GziaAaH",
        "timelock": null
      },
      "token": "00",
      "tx_id": "009f5274c1a5b5c5db5b8d1f0b63382d9b7efdaf91727297c12890a812fb38e7",
      "index": 0
    },
    {
      "value": 99998887700,
      "token_data": 0,
      "script": "dqkUA3DKxP9CT64k9YyE2pB0NvuFtjeIrA==",
      "decoded": {
        "type": "P2PKH",
        "address": "WNzE2S6XskM6ynYWbKJHzHtQAw2GziaAaH",
        "timelock": null
      },
      "token": "00",
      "tx_id": "009f5274c1a5b5c5db5b8d1f0b63382d9b7efdaf91727297c12890a812fb38e7",
      "index": 1
    }
  ],
  "outputs": [
    {
      "value": 100000,
      "token_data": 0,
      "script": "dqkUTP0s6rZkuW0RyFZ3F0jxhUJLSGWIrA==",
      "decoded": {
        "type": "P2PKH",
        "address": "WVh7XSpJgKv6ZGWQEB5uEsJgBJBfYe8ENn",
        "timelock": null
      },
      "token": "00",
      "spent_by": "00a286d1438c0980ab665d0d9c30f982367db3b2414efc5811a61420f492bb64"
    },
    {
      "value": 99998800000,
      "token_data": 0,
      "script": "dqkUseBbRWUAtMEhowIW2u1KfySzbbqIrA==",
      "decoded": {
        "type": "P2PKH",
        "address": "WetZFSrJpo7tuGx4xD8DdTNqnttBFvVg9W",
        "timelock": null
      },
      "token": "00",
      "spent_by": null
    }
  ],
  "parents": [
    "009f5274c1a5b5c5db5b8d1f0b63382d9b7efdaf91727297c12890a812fb38e7",
    "00bce2c6eb3f1731b1f60acba968a7508bd70136472ee34f5f1b66435469c7e8"
  ]
}, ...]
```

### How to check if a transaction is a valid deposit?

The following verification must be done:

- `version = 1`. The transaction data is from a regular transaction (not a block or a create token transaction).
- `token = "00" and token_data = 0`. It's an output of HTR token, not any other custom token created in our network.
- `is_voided = false`. The transaction is valid.
- `value` is an integer, so `1000` means `10.00`.
- For each output you must check:
  - Address is one of the user’s address;
  - `timelock = null`. The output is not locked, it can be spent anytime.
  - `type = "P2PKH"`. The type of the output is P2PKH, not a multisig output.

5. Number of confirmations to accept a deposit

Exchanges usually requires a minimum number of confirmations before accepting deposits. It is a common practice to avoid double spending attacks and protect its funds.

The recommended number of confirmation depends on the policy of each exchange. We recommend at minimum of 50 confirmation before accepting a deposit, which means it will take an average of 25 minutes to confirm deposits.

#### Get `first_block` from the transaction

```
$ curl https://{full-node}/v1a/transaction/?id=0000000044a7a3b4c24e26a405fb052e0935945616d504bd4f412507971dab68 | jq .meta.first_block

"0000000000000000acbe471376c06925e6f90585c7d3e205e14d140e44f3311b" -> This is the first block in the best blockchain that confirms the transction.
```

#### Get the height of a block

```
$ curl https://{full-node}/v1a/transaction/?id=0000000000000000acbe471376c06925e6f90585c7d3e205e14d140e44f3311b | jq .meta.height

1013585 -> This is the height of the block

height_tx = 1013585
```

#### Get network current height

The network current height is the height of the head of the blockchain, i.e., the latest block in the best blockchain.

```
$ curl https://{full-node}/v1a/get_block_template | jq .metadata.height

1013608 -> This is the next height of the network, so the current height is always the return of this command minus one (1013608 - 1 = 1013607)

height_network = 1013607
```

One can calculate the number of confirmations subtracting the height of the network and the height of a given block, as follows:

`height_network - height_block = 22`

6. Get an address for the withdrawal wallet

The withdrawal wallet will receive all HTR from deposits already confirmed and, in case it needs a new address, just need to send a request to the headless. But you can use a single address for the withdrawal wallet.

```
$ curl -H "X-Wallet-Id: idWithdrawal" http://{headless-wallet}/wallet/address

{
  "address": "WksLdBKvDYUNvAkApYHB9evmCcpHAftmvo"
}
```

7. Transfer deposit HTR to the withdrawal wallet

In this transaction it's important to select as an input the utxo received in the deposit. The value here is **ALWAYS** an integer, so `100` means `1.00`. You must use always an integer (e.g. `100`), never a float (e.g. `1.00`) or a string (e.g. `"100"`).

```
$ curl -X POST -H "X-Wallet-Id: idDeposit" -H "Content-type: application/json" --data '{"outputs": [{"address": withdrawalWalletAddress, "value": depositValue}], "inputs": [{"hash": depositTransactionHash, "index": indexOfOutputToSpend}]}' http://{headless-wallet}/wallet/send-tx
```

8. Withdrawal

User selects to withdraw to address X, so you first need to validate this address.

```
$ curl http://localhost:8084/v1a/validate_address/<INVALID_ADDRESS>
{"valid":false, "error": "InvalidAddress", "msg": "Address size must have 25 bytes"}

$ curl http://localhost:8084/v1a/validate_address/<VALID_ADDRESS>
{"valid":true, "script": "SCRIPT", "address": "VALID_ADDRESS", "type": "p2pkh"}
```

In this request the `X-Wallet-Id` parameter must be the withdrawal wallet ID because it's the wallet that will send the transaction. There is no need to select the input in this case because it can use any utxo. If you would like to keep your withdrawal wallet using the same address always you can also use the `change_address` parameter.

```
$ curl -X POST -H "X-Wallet-Id: idWithdrawal" -H "Content-type: application/json" --data '{"outputs": [{"address": X, "value": withdrawalValue}]' http://{headless-wallet}/wallet/send-tx
```

# Troubleshooting
[troubleshooting]: #troubleshooting

## Check if an address belongs to the wallet

To check whether a given address belongs to the wallet, use `GET /wallet/index-address?address=PUT_THE_ADDRESS_HERE`. If the address belongs to the wallet, the API will return its index.

## Get all addresses tracked by the wallet

To get a list of all addresses tracked by the wallet, use `GET /wallet/addresses`.

## Get the address of a specific index

To get the address of a specific index, use `GET /wallet/address?index=PUT_THE_INDEX_HERE`.

This API will return the address for any index even if this address is not being tracked by the wallet. Notice that the wallet only tracks the addresses until the GAP limit is reached. All addresses after the GAP limit belong to the wallet but they are not tracked, i.e., their transactions will not be loaded.
