- Feature Name: Stratum
- Start Date: 2019-01-04
- RFC PR:
- Hathor Issue:

# Summary
[summary]: #summary

Stratum is a protocol that allows efficient communication between miner clients and servers.
From the Stratum homepage:

> In a simplified manner, Stratum is a line-based protocol using plain TCP socket,
> with payload encoded as JSON-RPC messages.
>
> That's all. Client simply opens TCP socket and writes requests to the server in 
>the form of JSON messages finished by the newline character.
>
> Every line received by the climient is again a valid JSON-RPC fragment containing the response.

This proposal introduces a Stratum protocol for Hathor and discusses 
the implementation of a Stratum server embedded on Hathor node.

# Motivation
[motivation]: #motivation

The major motivation to introduce Stratum in Hathor is allowing separation between 
mining client from Hathor core, providing a clear API for miners to talk efficiently to 
Hathor nodes and mining pools.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Stratum allow **mining (or stratum) clients** to get and submit mining jobs to a **stratum server**.

To subscribe to mining jobs, miner should create a JSON RPC 2 `subscribe` request, e.g.:

```
{  
   "id":"2085195499",
   "jsonrpc":"2.0",
   "method":"subscribe",
   "params":null
}
```

Server should be able to answer the request with a mining job:

```
{
   "id":"2085195499",
   "jsonrpc":"2.0",
   "method":"subscribe",
   "result":{  
      "weigth":27.235,
      "job_id":1949732007,
      "hash":"e67be79550661604d9b9c646911c80d74f6c691613224c207fa7088c20698acd",
      "nonce_size":16
   }
}
```

Miner should look for a solution to the desined job. The client should look for a `nonce_size` byte `nonce` such that `sha256d` function of the concatenation of `hash` and `nonce` meets the target difficulty. Target difficulty `diff` can be calculated from `weigth` using the formula `diff = 2^(256 - weight) - 1`.

To notify the client of new jobs, the server sends `job` messages.
the `job` request params contains the same fields that the `subscribe` response field.

```
{
   "id":"3713981664",
   "jsonrpc":"2.0",
   "method":"job",
   "params":{  
      "weigth":21,
      "job_id":784707720,
      "hash":"fba8f8b8821a83361c9c7567441b45c10bb4ce421997ff9cfc8d52fbceff0292",
      "nonce_size":4
   }
}
```

The client should not create any response for this request.

When the miner finds a solution to the `job`, it should `submit` it:

```
{
   "id":"42",
   "jsonrpc":"2.0",
   "method":"submit",
   "params":{
      "job_id":784707720,
      "nonce":1200613,
   }
}
```

The server should check if the solution indeed meets the dificulty and answer:

```
{  
   "id":"42",
   "jsonrpc":"2.0",
   "method":"submit",
   "result":"ok",
   "error":null
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All messages should conform with the [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification).
This means that every messsage is a JSON object terminated by a `\n` character and contain the following fields:


| Field         | Content                                   |
| :------------ | :---------------------------------------- |
| id            | ID of the request                         |
| jsonrpc       | "2.0"                                     |
| method        | Stratum method id                         |

All **requests** should also contain the fields:

| Field         | Content                                                       |
| :------------ | :------------------------------------------------------------ |
| params        | nullable structured value containing the request parmameters  |


And all the **responses** can have either:

| Field         | Content                                           |
| :------------ | :------------------------------------------------ |
| result        | nullable structured value containing result data  |
| error         | Object containing error `code` and `message`      |

## Client to Server

### `subscribe` 
Miner subscribe for job notifications.
`params` should be `null`.

### `submit`
Miner submits a complete mining job. `params` should contain:

| Field         | Type       | Content                                              |
| :------------ | :--------- | :--------------------------------------------------- |
| job_id        | int        | Integer that identifies the job                      |
| nonce         | string     | String that contains the nonce that solves the job   |

Note that the nonce size depends on the type of the job. `nonce_bytes = block_job ? 16 : 4`.


### `authenticate`
Miner authenticates itself with server.
`params` should contain:

| Field         | Type       | Content                                              |
| :------------ | :--------- | :--------------------------------------------------- |
| username      | string     | Integer that identifies the job                      |
| password      | string     | String that contains the nonce that solves the job   |
| method        | string     | `password`                                           |

If the desired authentication method is `password`.
If the miner wants to authenticate using an API key, it should include the fields:

| Field         | Type       | Content                                              |
| :------------ | :--------- | :--------------------------------------------------- |
| key           | string     | API Key to access server                             |
| method        | string     | `key`                                                |


## Server to Client

### `job`
Server notifies the miner of a new job available. `params` should contain:

| Field         | Type       | Content                                              |
| :------------ | :--------- | :--------------------------------------------------- |
| nonce_size    | int        | Size of the available `nonce`                        |
| hash          | string     | Hash to be used as base for the job                  |
| job_id        | int        | Identifier of the job                                |
| weigth        | double     | Weigth of the job, used to calculate the difficulty  |



# Error Messages

Other than the error messages specified in the [JSON RPC 2.0 Specification](https://www.jsonrpc.org/specification#error_object), this protocol defines the following error messages:

| Error code  | Error Message         | Meaning                                         |
| :---------- | :-------------------- | :---------------------------------------------  |
| 10          | Node syncing          | Miner should wait for node to sync to network   |
| 20          | Login required        | Miner should `authenticate` with node           |
| 30          | Invalid Solution      | Solution didn't meet required difficulty        |
| 31          | Stale job submited    | Job was already stale when submited             |
| 32          | Job not found         | Node couldn't find any job with submited id     |

# Drawbacks
[drawbacks]: #drawbacks

This does not block any other similar solutions from being implemented and since it's just an API, there's no major drawbacks other than the time spent on implementing it.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Other alternatives includes:
- [Getblocktemplate](https://en.bitcoin.it/wiki/Getblocktemplate)
- [Getwork](https://en.bitcoin.it/wiki/Getwork)

Getwork is a legacy protocol - Stratum and Getblocktemplate were developed to take it's place.

Getblocktemplate is a little bit more complicated, but also very similar to Stratum.

Stratum suffices as a standard API for miners to connect to a node, given it's simplicity and wide adoption.


# Prior art
[prior-art]: #prior-art

As shown in the references sections there are several coins that have Stratum implementation - some which have it natively integrated with the coin. For examples, please refer to [References].

# Unresolved questions
[unresolved-questions]: #unresolved-questions
- Who killed PC Farias?

# Future possibilities
[future-possibilities]: #future-possibilities


# References
[references]: #references

- [Stratum Protocol Official Page](https://slushpool.com/help/manual/stratum-protocol)
- [Stratum Mining Protocol on Bitcoin Wiki](https://en.bitcoinwiki.org/wiki/Stratum_mining_protocol)

## Stratum Servers / APIs
- [Grin's Stratum Docs](https://github.com/mimblewimble/grin/blob/master/doc/stratum.md#reference-implementation)
- [Beam's Stratum Implementation](https://github.com/BeamMW/beam/blob/master/pow/stratum.h)
- [Monero Pool Stratum Implementation](https://github.com/sammy007/monero-stratum/blob/master/stratum/stratum.go) 

## Stratum Clients
- [CGMiner Github Project](https://github.com/ckolivas/cgminer)
- [CPUMiner Github Project](https://github.com/pooler/cpuminer)
- [BFGMiner Github Project](https://github.com/luke-jr/bfgminer)

## Other
- [Bitcoin Blockchain Dev Reference](https://bitcoin.org/en/developer-reference#block-chain)
- [Stratum Documentation Compilation](https://bitcointalk.org/index.php?topic=557866.5)
- [Stratum Flow Example](https://bitcoin.stackexchange.com/questions/22929/full-example-data-for-scrypt-stratum-client)