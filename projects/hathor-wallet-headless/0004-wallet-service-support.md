# Summary
[summary]: #summary

Add initial support for running a wallet with wallet-service on the headless.
While we cannot have feature parity, we can start support as a beta api and
increase support over time.

# Motivation
[motivation]: #motivation

The headless wallet is intended to run as a service to manage tokens on the
network. Wallets from some services can have many transactions, utxos and
tokens, while the usual startup is enough for most use-cases we could use the
wallet-service as an alternative to a faster startup (constant time for any
number of transactions).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are a number of features that cannot be ported to the wallet-service
facade yet. These are:

- P2SH API.
- atomic-swap API.
- index-limit address scanning policy.
- nano contracts API.
- tx-proposal API.
- mint/melt API with data outputs.

Most of the issues come from how the wallet-lib uses the storage as a source of
utxos/transactions and the wallet-service only saves access data on storage.

To have feature parity on the usual routes we should implement the mint/melt API
with data outputs and leave the others for future implementations.

Since we do not have feature parity, we will start with the wallet-service using
a special endpoint prefix `/beta` to indicate that this feature is usntable,
i.e. `/beta/wallet-service/start`.

<details>

<summary><code>POST</code> <code><b>/beta/wallet-service/start</b></code> <code>(Start a wallet with wallet-service)</code></summary>

This API will start a wallet with the wallet-service facade.

##### Parameters

> | name | type | data type | description | location |
> | --- | --- | --- | --- | --- |
> | wallet-id  | required | string | create a wallet with this id.          | body |
> | seedKey    | required | string | The id of the seed in the config.      | body |
> | passphrase | optional | string | passphrase to add entropy to the seed. | body |

##### Responses

> | http code | content-type | response |
> | --- | --- | --- |
> | `200` | `application/json` | `{"success":true}` |
> | `400` | `application/json` | `{"success": false, "message":"Bad Request"}` |

##### Example cURL

> ```javascript
>  curl -X POST -H "Content-Type: application/json" --data '{"seedKey": "cafe", "wallet-id": "cafe"}' 'http://localhost:8000/beta/wallet-service/start'
> ```

</details>

Once the wallet is started, it can be used as a regular wallet on the usual
APIs.

# Future possibilities
[future-possibilities]: #future-possibilities

The issue is that some features may never be implemented on the wallet-service,
which works fine since some of them are very specific (e.g. index-limit scanning
policy) so we should create a feature guard for our routes, this feature guard
can safely return to the user that the feature he is trying to use is not
available for the stated wallet.
This can be done simply by mapping the endpoints to a set of feature tags and
which wallets can access these feature tags, a middleware can be used to check
this and return an error if the feature is not available.

For documentation purposes these tags should also be listed on the api-docs, so
any user can see the documentation and what features he will "lose" if he starts
a wallet with wallet-service or readonly, etc.

This feature guard will also allow us to safely remove the `/beta` prefix from
the wallet-service endpoint since it will protect the user from routes that are
not supported and may never be supported.

# Task breakdown
[task-breakdown]: #task-breakdown

- Add support for data outputs on mint/melt wallet-service facade. (1 devday)
- Check all wallet facade methods used by the headless are supported (1 devday)
  - May be extended based on findings
- Implement beta wallet-service start API (1 devday)
