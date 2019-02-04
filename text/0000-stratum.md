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

Server should be able to answer the request - asking for credentials or saying that everything is all right:

```
{
   "id":"2085195499",
   "jsonrpc":"2.0",
   "result": "ok
}
```

And also send a new `job` request to the miner:
```
{
   "method":"job",
   "result":{  
      "weigth":27.235,
      "job_id":1949732007,
      "data":"e67be79550661604d9b9c646911c80d74f6c691613224c207fa7088c20698acd",
      "nonce_size":16
   }
}
```

Miner should look for a solution to the desined job. The client should look for a `nonce_size` byte `nonce` such that `sha256d` function of the concatenation of `data` and `nonce` meets the target difficulty. Target difficulty `diff` can be calculated from `weigth` using the formula `diff = 2^(256 - weight) - 1`.

To notify the client of new jobs, the server sends new `job` messages:

```
{
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

Since there is no `id` field on the message, the client should not create any response for this request - it is just a notification.

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
   "result":"ok",
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All messages should conform with the [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification).
This means that every messsage is a JSON object terminated by a `\n` character and may contain the following fields:


| Field         | Content                                   |
| :------------ | :---------------------------------------- |
| id            | ID of the request                         |
| jsonrpc       | "2.0"                                     |

`id` field should be a random `64`-bit integer. Requests without this field are **notifications**. They are not suposed to be answered.


All **requests** should also contain the fields:

| Field         | Content                                                       |
| :------------ | :------------------------------------------------------------ |
| method        | Stratum method id                         |
| params        | nullable structured value containing the request parmameters  |


And all the **responses** can have either - but never both:

| Field         | Content                                           |
| :------------ | :------------------------------------------------ |
| result        | nullable structured value containing result data  |
| error         | Object containing error `code` and `message`      |

## Client to Server

### `subscribe` 
Miner subscribe for job notifications.
`params` should be `null` or not present.

Request:

```
{  
   "id":"39821739821",
   "jsonrpc":"2.0",
   "method":"subscribe",
   "params":null
}
```

Response:
```
{
   "id":"39821739821",
   "jsonrpc":"2.0",
   "result": "ok
}
```

Errors may include messages with code `10` and `20` - see [Error Messages](#error-messages) secion for further details
.

### `submit`
Miner submits a complete mining job. `params` should contain:

| Field         | Type       | Content                                              |
| :------------ | :--------- | :--------------------------------------------------- |
| job_id        | int        | `64`-bit integer that identifies the job                      |
| nonce         | string     | String that contains the nonce that solves the job   |

Note that the nonce size depends on the type of the job. `nonce_bytes = block_job ? 16 : 4`.

Request:

```
{
   "id":"2312398",
   "jsonrpc":"2.0",
   "method":"submit",
   "params":{
      "job_id":784707720,
      "nonce":1200613,
   }
}
```

Response:

```
{  
   "id":"2312398",
   "jsonrpc":"2.0",
   "result":"ok",
}
```

Errors may include any of the messages listed in the [Error Messages](#error-messages) section.


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

This is not crucial for the communication between miners and servers to be stablished securily.
So it won't be implemented in the first version.

Request example:
```
{
   "id":"48971503",
   "jsonrpc":"2.0",
   "method":"authenticate",
   "params":{
      "username": "foo",
      "password": "bar",
      "method": "password"
   }
}
```

Response example:

```
{  
   "id":"48971503",
   "jsonrpc":"2.0",
   "result":"ok",
}
```

Errors may include messages with code `10` and `21`. See [Error Messages](#error-messages) section for further information.


## Server to Client

### `job`
Server notifies the miner of a new job available. `params` should contain:

| Field         | Type       | Content                                              |
| :------------ | :--------- | :--------------------------------------------------- |
| nonce_size    | int        | Size of the available `nonce`                        |
| hash          | string     | Hash to be used as base for the job                  |
| job_id        | int        | Identifier of the job                                |
| weigth        | double     | Weigth of the job, used to calculate the difficulty  |

Since it is a notification the miner should not answer it.

Request:

```
{
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

## Error Messages
[error]: #error

Other than the error messages specified in the [JSON RPC 2.0 Specification](https://www.jsonrpc.org/specification#error_object), this protocol defines the following error messages:

| Error code  | Error Message               | Meaning                                         |
| :---------- | :-------------------------- | :---------------------------------------------- |
| 10          | Node syncing                | Miner should wait for node to sync to network   |
| 20          | Login required              | Miner should `authenticate` with node           |
| 21          | Authentication failed       | Credentials were not accpeted                   |
| 30          | Invalid solution            | Solution didn't meet required difficulty        |
| 31          | Stale job submited          | Job was already stale when submited             |
| 32          | Job not found               | Node couldn't find any job with submited id     |
| 33          | Solution propagation failed | Node couldn't propagate the submited solution   |

# Drawbacks
[drawbacks]: #drawbacks

From the point of view of chosing Stratum as the protocol, there's no major drawbacks of choosing Stratum other than the time spent on implementing it. This does not block any other similar solutions from being implemented since it's just an API.

From the point of view of what is communicated through Stratum, the major drawback that is faced is the fact that the whole serialized transaction is sent right now.


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

- Should we change the mining algorithm (e.g. hash the transaction without nonce and then with nonce), to reduce network traffic, allow miners to update timestamp, etc.?
- Who killed PC Farias?

# Future possibilities
[future-possibilities]: #future-possibilities
On the Stratum roadmap:

- Add `authorization` request, response and error messages.
- Support pools - get feedback from pools on what is the best way to support them.
- Multi-mining support

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