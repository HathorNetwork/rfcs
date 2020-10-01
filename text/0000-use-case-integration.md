- Feature Name: use_case_integration
- Start Date: 2020-09-30
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

This document describes different ways to integrate Hathor in a use case.


# Motivation
[motivation]: #motivation

This document is a useful resource for any individual or company that would like to integrate Hathor in their product. It covers different ways to do the integration, good practices, security tips, and so on.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Use cases usually will create their own token, and they want to perform the following operations:

1. Send tokens
2. Receive tokens
3. Check the balance
4. Get the transaction history

In order to run your own environment that does not depend on third party services, the following services should be used:

- [Hathor Wallet Headless](https://github.com/HathorNetwork/hathor-wallet-headless/) (`hathor-wallet-headless`)
- [Hathor Wallet Lib](https://github.com/HathorNetwork/hathor-wallet-lib) (`hathor-wallet-lib`)
- [Hathor Full Node](https://github.com/HathorNetwork/hathor-core/) (`hathor-core`)
- [Tx Mining Service](https://github.com/HathorNetwork/tx-mining-service) (`tx-mining-service`)

All services are available as [Docker Images](https://hub.docker.com/u/hathornetwork) and can easily be executed in containers in any operating system.

## Basic Concepts

In Hathor's blockchain, tokens are stored in addresses. For example, if there are 10 HTR stored in the address `H8bt9nYhUNJHg7szF32CWWi1eB8PyYZnbt`, and you're the owner of this address, it means that you are the only person who is able to transfer these tokens. We say that you are the owner of an address if, and only if, you have control of the private key associated with the address.

A wallet is a software program that manages multiple addresses in which tokens are stored. The balance of a wallet is the sum of the individual balances of its addresses. Besides the addresses, a wallet also manages the private keys of the addresses.

<pre>
┌───────────────────────────────────────┐
│                Wallet                 │
│            Balance: 28 HTR            │
└┬─────────────────────────────────────┬┘
 │┌──────────────────┐ ┌──────────────┐│
 ││  Private Key 1  ◀┼─┼▶ Address 1   ││
 ││                  │ │    10 HTR    ││
 │└──────────────────┘ └──────────────┘│
 │┌──────────────────┐ ┌──────────────┐│
 ││  Private Key 2  ◀┼─┼▶ Address 2   ││
 ││                  │ │    6 HTR     ││
 │└──────────────────┘ └──────────────┘│
 │┌──────────────────┐ ┌──────────────┐│
 ││  Private Key 3  ◀┼─┼▶ Address 3   ││
 ││                  │ │    12 HTR    ││
 │└──────────────────┘ └──────────────┘│
 │┌──────────────────┐ ┌──────────────┐│
 ││  Private Key 4  ◀┼─┼▶ Address 4   ││
 ││                  │ │    0 HTR     ││
 │└──────────────────┘ └──────────────┘│
 └─────────────────────────────────────┘
</pre>

A transaction is used to move tokens from one address to another. For example, let's say you want to send 1 HTR to your friend who owns the address `HCAQb2H5EUqv9AoThwHQcibZe5nvppscMh`. To do this, you will create a transaction that moves tokens from your address to your friend's address. After the transaction is created, you will propagate it into Hathor's network, and the transaction will soon be confirmed by the network.

A transaction has the following main parts (others are omitted for clarity):

1. Unique Identifier (`tx-id`)
2. List of Inputs
3. List of Outputs

Technically speaking, a transaction moves tokens from its inputs to its outputs. Each input points to the output of another transaction, and we say that this input is spending that output. So, simply put, a transaction spends its inputs and transfers them to its outputs. Outputs may be spent only once, so we say that an output is either spent or unspent.

<pre>
     ┌───────────────────────────────────┐           ┌───────────────────────────────────┐
     │           Transaction 1           │           │           Transaction 3           │
     └┬────────────────┬────────────────┬┘           └┬────────────────┬────────────────┬┘
      │┌──────────────┐│┌──────────────┐│             │┌──────────────┐│┌──────────────┐│
◀─────┼┼─  Input 1    │││   Output 1  ◀┼┼─────────────┼┼─  Input 1    │││   Output 1   ││
      │└──────────────┘││    6 HTR     ││             │├──────────────┤││    14 HTR    ││
      │                │└──────────────┘│      ┌──────┼┼─  Input 2    ││└──────────────┘│
      │                │┌──────────────┐│      │      │└──────────────┘│                │
      │                ││   Output 2   ││      │      │                │                │
      │                ││    1 HTR     ││      │      │                │                │
      │                │└──────────────┘│      │      └────────────────┴────────────────┘
      │                │┌──────────────┐│      │
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

Each output is associated with one address, and one address may be associated with many outputs. The balance of an address is calculated as the sum of all its unspent outputs.

Here is an example of a transaction with one input and one output in JSON format. Notice that it has some fields we haven't discussed yet. In fact, they are not necessary to understand how transactions move tokens from one address to another. The address of the output is encoded in the `script` field.

```json
{
  "tx_id": "00001bc7043d0aa910e28aff4b2aad8b4de76c709da4d16a48bf713067245029",
  "nonce": 33440807,
  "timestamp": 1579656120,
  "version": 1,
  "weight": 16.827294220302488,
  "parents": [
    "000036e846dee9f58a724543cf5ee14cf745286e414d8acd9563963643f8dc34",
    "000000fe2da5f4cc462e8ccaac8703a38cd6e4266e227198f003dd5c68092d29"
  ],
  "inputs": [
    {
      "tx_id": "000000fe2da5f4cc462e8ccaac8703a38cd6e4266e227198f003dd5c68092d29",
      "index": 0,
      "data": "RzBFAiEAyKKbtzdH7FjvjUopHFIXBf+vBcH+2CKirp0mEnLjjvMCIA9iSuW4B/UJMQld+c4Ch5lIwAcTbzisNUaCs+JpK8yDIQI2CLavb5spKwIEskxaVu0B2Tp52BXas3yjdX1XeMSGyw=="
    }
  ],
  "outputs": [
    {
      "value": 1,
      "token_data": 0,
      "script": "dqkUtK1DlS8IDGxtJBtRwBlzFWihbIiIrA=="
    }
  ],
  "tokens": []
}
```


## Services

Here we will describe all Hathor Services that may be used in an integration. The following diagram shows how they are connected between them.

<pre>
┌─────────────────────────────────────────────────────────────────────────────────┐
│Hathor Services                                                                  │
│                                                                                 │
│                                                                                 │
│    ┌──────────────────────────────┐        ┌──────────────────────────────┐     │
│    │                              │        │                              │     │
│    │                              │        │                              │     │
│    │       Hathor Full Node     ◀─┼────────┼─▶                            │     │
│    │                              │        │                              │     │             ┌─────────────────────┐
│    │               ▲              │        │                              │     │             │                     │
│    └───────────────┼──────────────┘        │                              │ HTTP API          │                     │
│                    │                       │    Hathor Wallet Headless  ◀─┼─────┼─────────────┼─▶ Services To Be    │
│                    │                       │                              │     │             │     Integrated      │
│    ┌───────────────┼──────────────┐        │                              │     │             │                     │
│    │               ▼              │        │                              │     │             │                     │
│    │                              │        │                              │     │             └─────────────────────┘
│    │      Tx Mining Service     ◀─┼────────┼─▶                            │     │
│    │                              │        │                              │     │
│    │                              │        │                              │     │
│    └──────────────────────────────┘        └──────────────────────────────┘     │
│                                                                                 │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
</pre>

### Hathor Wallet Headless

All wallet operations can be easily integrated consuming [Hathor Wallet Headless](https://github.com/HathorNetwork/hathor-wallet-headless/)'s HTTP API. This wallet is a service developed to run as a daemon (without a graphical user interface) and to be controlled by its endpoints.

It usually is the only service you need to develop your integrations with. At the time of writing, it supports the following operations:

1. Create and control one more wallets. Each wallet is identified by a unique identifier (`wallet-id`).
2. Get a wallet balance.
3. Send transaction.
4. Get transaction history.
5. Get an address to receive funds.

It can be executed as an internal service without any internet access. It also supports a simple API Key verification for basic access control. One can easily customize a more advanced authentication scheme using an authenticated proxy.

It has two dependencies: (i) Hathor Full Node, and (ii) Tx Mining Service.

Hathor Wallet Headless needs to connect to a Hathor Full Node to perform almost all its operations, including sending and receiving transactions.

Hathor Wallet Headless needs a Tx Mining Service exclusively to send transactions.


### Hathor Wallet Lib

[Hathor Wallet Lib](https://github.com/HathorNetwork/hathor-wallet-lib) is an easy-to-use library designed to make it easier to develop wallets or integrate wallets with mobile and desktop apps.

If you are using Hathor Wallet Headless, you probably don't need to use this library.


### Hathor Full Node

[Hathor Full Node](https://github.com/HathorNetwork/hathor-core/) (shortly, full node) is a daemon that connects to Hathor's mainnet and keep a local copy of the blockchain synchronized with the p2p network. It must have internet access and consumes a low bandwidth.

When the full node is started, it connected to the p2p mainnet and starts to sync. After it gets synchronized, you're ready to use your wallet.

You usually don't have to integrate anything with the full node. It is used by both Hathor Wallet Headless and Tx Mining Service.


### Tx Mining Service

[Tx Mining Service](https://github.com/HathorNetwork/tx-mining-service) is a daemon with a simple HTTP API to resolve transactions.

Tx Mining Service needs to connect to a Hathor Full Node to perform its operations. In this case, you can point to the same full node as you're pointing your wallet.

Besides running this service, you will need to connect at least one miner to it. You can use a CPU miner, a GPU miner or an ASIC miner.

You usually don't have to integrate anything with the tx mining service. It is exclusively used by Hathor Wallet Headless.


## Testing with public services

Hathor Labs maintains two public full nodes and one public tx mining service. You can safely use them for testing. But you shouldn't
use them for your production environment because (i) they have rate limits in their endpoints, (ii) they can be turned off anytime,
and (iii) their endpoint may be blocked ocasionally.

Anyway, here are the public endpoints:

- node1: https://node1.mainnet.hathor.network/
- node2: https://node2.mainnet.hathor.network/
- tx-mining-service: https://txmining.mainnet.hathor.network/


# Code Examples in Python
[code-examples]: #code-examples

These examples expect a `hathor-wallet-headless` running locally on port 8000.

## Getting current balance

```python
import urllib.parse
import requests

base_url = 'http://localhost:8000'
url = urllib.parse.urljoin(base_url, '/wallet/balance')
headers = {
    'X-Wallet-Id': 'default',  # your wallet-id
}
response = requests.get(url, headers=headers)
print(response.json())
```

## Sending a transaction

```python
import urllib.parse
import requests

base_url = 'http://localhost:8000'
url = urllib.parse.urljoin(base_url, '/wallet/simple-send-tx')
data = {
    'address': 'H8bt9nYhUNJHg7szF32CWWi1eB8PyYZnbt',  # address of the recipient
    'amount': 1025,  # amount in cents (10.25 HTR)
}
headers = {
    'X-Wallet-Id': 'default',  # your wallet-id
}
response = requests.post(url, data, headers=headers)
print(response.json())
```
