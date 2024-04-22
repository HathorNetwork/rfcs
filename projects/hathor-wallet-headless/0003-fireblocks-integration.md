# Summary
[summary]: #summary

Hathor headless wallet integration with Fireblocks using RAW Signing API.

# Motivation
[motivation]: #motivation

Fireblocks manages the private key so removing the private key from the headless instance will allow users to run the headless in a more secure manner.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

<details>

 <summary><code>POST</code> <code><b>/wallet/fireblocks/start</b></code> <code>(Start a wallet with the fireblocks integration)</code></summary>

This API will use the wallet's `setExternalTxSigningMethod` to register a method that will start a client for fireblocks and use Fireblock's RAW signing API to sign the transaction.

##### Parameters

> | name | type | data type | description | location |
> | --- | --- | --- | --- | --- |
> | xpub-id | required | string | The id of the xpub in the config | body | 
> | wallet-id | required | string | create a wallet with this id | body |

##### Responses

> | http code | content-type | response |
> | --- | --- | --- |
> | `200` | `application/json` | `{"success":true}` |
> | `400` | `application/json` | `{"success": false, "message":"Bad Request"}` |

##### Example cURL

> ```javascript
>  curl -X POST -H "Content-Type: application/json" --data '{"xpub-id": "cafe", "wallet-id": "cafe", "raw": true}' 'http://localhost:8000/fireblocks/start'
> ```

</details>

Fireblocks has documented a few possibilities for integrations, we will use the [REST API](https://developers.fireblocks.com/docs/rest-api-guide) to implement our client, this means that we need to implement the authorization workflow and API calls.

## Fireblocks Client

### BIP44 path derivation

Fireblocks follows a derivation path very similar to BIP44 but without any hardened derivations.
This is why we can use the root xPub from the console to generate the account level xPub, even though it is not a BIP44 account level it will work exactly the same with our wallet-lib.

The usual account derivation path of `m/44'/280'/0'` will be replaced with `m/44/280/0`, we will need to create a script that derives the root xPub to the fireblocks account level path.

From the account level the change and address derivation will work as usual.

### Fireblocks API Authorization

[reference](https://developers.fireblocks.com/reference/signing-a-request-jwt-structure)

The Fireblocks authorization workflow is defined by having 2 headers:
- `X-API-Key`: The api key identifying the api user.
- `Authorization`: Bearer token using JWT, with custom claims and signed with the api user's secret key (downloaded from Fireblocks dashboard).

The JWT token requires the claims:
- `uri`: path with querystring.
- `nonce`: UUID4 or any random generator.
- `iat` and `exp`: "issued at time" and "expiration".
	- SInce it's a per-call token the difference between them can be very short (e.g. 30s).
- `sub`: The api key used in the first header.
- `bodyHash`: The sha256 hash of the JSON encoded body of the request
	- Or sha256 hash of an empty string if the body is empty (e.g. GET requests).

The token must be signed with `RS256` (RSA + SHA256) using the api user's private key (downloaded from Fireblocks).

Some examples are provided on how to implement this (see [reference](https://github.com/fireblocks/developers-hub/tree/main/authentication_examples)).

### RAW Signing API

[reference](https://developers.fireblocks.com/reference/post_transactions)
[guide](https://developers.fireblocks.com/docs/raw-message-signing-overview)

The RAW Signing uses the normal transaction POST to create a `RAW` operation where we can request a content (hex encoded) to be signed by the derivation path we send.
The operation is treated as a transaction and returns a transaction id which can be used with the Transaction API to check the status.

### Transaction API

[reference](https://developers.fireblocks.com/reference/get_transactions-txid)

With the transaction id we can check the transaction status, once the status is `COMPLETED` the return object will have the requested signatures.

The signatures are returned as raw r, s values so we need to encode them in DER to use with our wallet lib.
The signature also comes with the public key of the key used to sign the content which can be used to check against the locally generated public key to assert that the correct private key was used.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Start API

The start API is very straightforward, we only need to evaluate the wallet-id, xpub-key and configured credentials to fireblocks.

To check that the credentials are valid we can use the [public_key_info API](https://developers.fireblocks.com/reference/get_vault-public-key-info) to fetch the first address at derivationPath `[44, 280, 0, 0, 0]`.

## External signing function

Since Fireblocks API a new token is required for each request we can have a method that gathers the "signature requests" from a transaction and sends a single request for RAW signing.
The resulting signatures will be fetched from the Transation API which we will need to encode to DER to use with the wallet-lib.

DER encoded signature has the following structure:

| sequence tag | len(sequence) in bytes | integer tag | len(r) in bytes |   r   | integer tag | len(s) in bytes |   s   |
| :----------: | :--------------------: | :---------: | :-------------: | :---: | :---------: | :-------------: | :---: |
|     0x30     |         1 byte         |    0x02     |     1 byte      | bytes |    0x02     |     1 byte      | bytes |
Where `r` and `s` may be prefixed by 0x00 if they are negative (first bit is 1).

The Fireblocks signing method will be registered with the wallet facade and all subsequent transactions from this facade will be signed with Fireblocks, giving full compatibility with the other APIs in the headless wallet.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Fireblocks SDK

Fireblocks provides a javascript, typescript and python SDKs but while trying to use the SDK some dependency of it conflicted with bitcore-lib making the headless fail and not startup at all.
The SDK also adds many unneeded dependencies, so using the REST API will allow us to add some minimal dependencies at the cost of implementing all AIPs we want to use.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

This design only contemplates the RAW signing flow using our own wallet-lib to create, mine and push the transaction to the network.
The full integration with Fireblocks will not be contemplated because it requires additional implementation from Fireblocks which may happen in the future.
