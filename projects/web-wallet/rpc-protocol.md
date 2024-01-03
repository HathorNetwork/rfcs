# 1. Introduction

This document outlines the remote procedure call (RPC) methods for a decentralized application (dApp) to interact with Hathor wallets.

The design is currently a minimum viable product (MVP), addressing only known needs from our community. It should be noted that this API design may change as the platform evolves and additional needs arise.

The idea here is to partially replicate the headless wallet API as it is already being used by many partners and has been validated to work on their use cases

These methods are going to be implemented in the wallet, and exposed to dApps using wallet-connect, the design of the integration can be read [here](https://github.com/HathorNetwork/rfcs/blob/master/projects/web-wallet/wallet-connect/0001-design.md) and the wallet-connect design docs can be read [here](https://docs.walletconnect.com/web3wallet/about).

# 2. Reference-level explanation

## 2.1 RPC Methods

### 2.1.1 `htr_createTx`

The `htr_createTx` method creates a new transaction based on parameters. It will select the `utxos` automatically, mine and send the transaction

**Parameters**

1. `outputs` - An array of outputs
	1. `address` - Destination address of the output. Required if P2PKH or P2SH
	2. `value` - (Required if P2PKH or P2SH) The value parameter must be an integer with the value in cents, i.e., 123 means 1.23 HTR.
	3. `token` - (Defaults to `HTR`) Token id of the output.
	4. `type` - (Required if data script output) Type of output script.
	5. `data` - (Required if data script output) Data string of the data script output.
2. `inputs` - (Optional) An array of inputs to create the transaction
  1. `type` - Type of input object. Can be either 'query' which is the default or 'specific'
  1. `hash` - Hash of the transaction being spent in this input. Used if not type query
  1. `index` - Index of the transction being spent in this input. Used if not type query
  1. `max_utxos` - Maximum number of utxos to filter in the query. Optional query parameter when using type query
  1. `token` - Token uid to filter utxos in the query. Optional query parameter when using type query 
  1. `filter_address` - Address to filter utxos in the query. Optional query parameter when using type query 
  1. `amount_smaller_than` - Filter only utxos with values smaller than this. Optional query parameter when using type query 
  1. `amount_bigger_than` - Filter only utxos with value bigger than this. Optional query parameter when using type query.
3. `changeAddress` - (Optional) Address to send the change


**Example Requests:**

```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "method": "htr_createTx",
  "params": {
  	"outputs": [{
  	  "address": "H8RmX1AMQKAhWkBvf8DaKoAU7ph3Yiqg3c",
  	  "value": 100,
  	  "token": "00"
  	}],
    "inputs": [{
      "hash": "00004788e55b9b3fb90aaad2065e99ccf36ccd2fac5ed40ae62eeefd2486e96c",
      "index": 1
    }]
	}
}
```

```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "method": "htr_createTx",
  "params": {
  	"outputs": [{
  	  "address": "H8RmX1AMQKAhWkBvf8DaKoAU7ph3Yiqg3c",
  	  "value": 100,
  	  "token": "00"
  	}],
    "inputs": [{
      "filter_address": "H8RmX1AMQKAhWkBvf8DaKoAU7ph3Yiqg3c",
      "token": "00",
      "max_utxos": 10,
      "amount_bigger_than": 100
    }]
	}
}
```

**Example Response:**

```json
{
  "hash": "00000000059dfb65633acacc402c881b128cc7f5c04b6cea537ea2136f1b97fb",
  "nonce": 2455281664,
  "timestamp": 1594955941,
  "version": 1,
  "weight": 18.11897634891149,
  "parents": [
    "00000000556bbfee6d37cc099a17747b06f48ca3d9bf4af85c707aa95ad04b3f",
    "00000000e2e3e304e364edebff1c04c95cc9ef282463295f6e417b85fec361dd"
  ],
  "inputs": [
    {
      "tx_id": "00000000caaa37ab729805b91af2de8174e3ef24410f4effc4ffda3b610eae65",
      "index": 1,
      "data": "RjBEAiAYR8jc+zqY596QyMp+K3Eag3kQB5aXdfYja19Fa17u0wIgCdhBQpjlBiAawP/9WRAqAzW85CJlBpzq+YVhUALg8IUhAueFQuEkAo+s2m7nj/hnh0nyphcUuxa2LoRBjOsEOHRQ"
    },
    {
      "tx_id": "00000000caaa37ab729805b91af2de8174e3ef24410f4effc4ffda3b610eae65",
      "index": 2,
      "data": "RzBFAiEAofVXnCKNCEu4GRk7j+wHpQM6qmezRcfxHCe/PcUdbegCIE2nip27ZQtkpkEgNEhycqHM4CkLYMLVUgskphYsd/M9IQLHG6YJxXifQ6eMxPHbINFEJAUvrzKWe9V7AXXW4iywjg=="
    }
  ],
  "outputs": [
    {
      "value": 100,
      "token_data": 0,
      "script": "dqkUqdK8VisGSJuNItIBRYFfSHfHjPeIrA=="
    },
    {
      "value": 200,
      "token_data": 0,
      "script": "dqkUISAnpOn9Vo269QBvOfBeWJTLx82IrA=="
    }
  ],
  "tokens": []
}
```


### 2.1.2 `htr_createToken`

The `htr_createToken` method creates a new token transaction based on parameters 

**Parameters**

1. `name` - The name of the new token
2. `symbol` - Symbol of the token
3. `amount` - The amount of tokens to mint as an integer
4. `address` - (Optional) Destination address of the newly minted tokens
5. `change_address` - (Optional) Address to send the change amount
6. `create_mint` - (Defaults to true) If the wallet should create a mint authority for the newly created token
7. `mint_authority_address` - (Optional) Address to send the mint authority output
8. `allow_external_mint_authority_address` - (Defaults to false) Flag indicating if the mint authority address is allowed to be from another wallet. 
9. `create_melt` - (Defaults to true) If the wallet should create a melt authority for the newly created token
10. `melt_authority_address` - (Optional) Address to send the melt authority output
11. `allow_external_melt_authority_address` - (Defaults to false) Flag indicating if the melt authority address is allowed to be from another wallet.

**Example Request:**

```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "method": "htr_signWithAddress",
  "params": {
	"name": "Bathor",
	"symbol": "BTR",
	"amount": 500,
	"address": "H8RmX1AMQKAhWkBvf8DaKoAU7ph3Yiqg3c",
	"create_mint": false,
	"mint_authority_address": "H8RmX1AMQKAhWkBvf8DaKoAU7ph3Yiqg3c",
	"create_melt": false,
	"melt_authority_address": "H8RmX1AMQKAhWkBvf8DaKoAU7ph3Yiqg3c"
  }
}
```

**Example Response:**

```json
{
  "txId": "00004788e55b9b3fb90aaad2065e99ccf36ccd2fac5ed40ae62eeefd2486e96c",
  "name": "Bathor",
  "symbol": "BTR",
  "amount": 500
}
```


### 2.1.3 `htr_getUtxos`

The `htr_getUtxos` method will search for `utxos`  given a set of filters

**Parameters**

1. `maxUtxos` - (Defaults to 255) Maximum number of `utxos` to return.
2. `token` - (Defaults to HTR) Token to filter the `utxos`
3. `filterAddress` - Address to filter the `utxos`. The address must belong to the initialized wallet
4. `amountSmallerThan` - (Optional) Maximum limit of `utxo` amount to filter the list. We will only return `utxos` that have the `amount` lower than this value. Integer representation of decimals, i.e. 100 = 1.00.
5. `amountBiggerThan` - (Optional) Minimum limit of `utxo` amount to filter the list. We will only return `utxos` that have the `amount` higher than this value. Integer representation of decimals, i.e. 100 = 1.00.
6. `maximumAmount` - (Optional) Limit the maximum total amount to return summing all utxos. Integer representation of decimals, i.e. 100 = 1.00.
7. `onlyAvailableUtxos` - (Defaults to true) Get only available `utxos`, ignoring locked ones.

**Example Request:**

```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "method": "htr_getUtxos",
  "params": {
	"maxUtxos": 50,
	"token": "BTR",
	"amount": 1
  }
}
```

**Example Response:**

```json
{
  "total_amount_available": 5,
  "total_utxos_available": 10,
  "total_amount_locked": 0,
  "total_utxos_locked": 0,
  "utxos": [
    {
      "address": "HNnK9wgUVL6Cjzs1K3jpoGgqQTXCqpAnW8",
      "amount": 1,
      "tx_id": "00fff7a3c6eb95ec3343bffcfca9a3a0d3e243462ae7de1f200cdd76716140fb",
      "locked": false,
      "index": 0
    }
  ]
}
```


### 2.1.4 `htr_signWithAddress`

The `htr_signWithAddress` method requests a signed message using an address' private key that can be verified.

**Parameters**

1. `message` - String containing a message to be signed
2. `addressIndex` - Address index to sign the message with

**Request:**

```json
{
  "id": 3,
  "jsonrpc": "2.0",
  "method": "htr_signWithAddress",
  "params": {
    "message": "sign-me",
    "address": 5
  }
}
```
**Response:**

```json
{
  "message": "H2/QA48wJhvbKJH6hWXUJPHK5kUTwx4XAxuZGFzUeSPwLeFBv206lOs8ECHMTV/KilGbFP1p5e52mMGTUDFT7WM="
}
```

### 2.1.5 `htr_getBalance`

Different from account-based blockchains where users are used to always using the same address on each transaction, getting the balance on hathor involves going through a list of addresses from the user's wallet and calculating the sum of all utxos.

So in order for a `dApp` to know the balance of a given token for the connected user's wallet, we must have a method to fetch it.

The `htr_getBalance` method fetches the balance for a given token.

**Parameters**

1. `token` - (Defaults to `HTR`) May be specified for a token other than `HTR`

**Request:**

```json
{
  "id": 4,
  "jsonrpc": "2.0",
  "method": "htr_getBalance",
  "params": {
    "token": "000023a025d513c5c75410e771c297b23edc88b8fac7da1eff6fdc81e628c20d"
  }
}
```
**Response:**

```json
{
  "id": 4,
  "jsonrpc": "2.0",
  "result": {
    "available": 500,
    "locked": 0
  }
}
```

### 2.1.6 `htr_getConnectedNetwork`

This method is used to allow the dApp to validate that it's running in the same network the wallet is connected to.

**Parameters**

**Request:**

```json
{
  "id": 4,
  "jsonrpc": "2.0",
  "method": "htr_getConnectedNetwork"
}
```
**Response:**

```json
{
  "id": 4,
  "jsonrpc": "2.0",
  "result": {
    "network": "mainnet" | "testnet" | "privatenet",
    "genesisHash": "00000000000000000437c87c...c8bf9e9d2fda7de9763"
  }
}
```

### 2.1.7 `htr_getAddress`

This method allows the dApps to request an address. The wallet implementation
should decide the address to return

**Parameters**

**Request:**

```json
{
  "id": 4,
  "jsonrpc": "2.0",
  "method": "htr_getAddress"
}
```
**Response:**

```json
{
  "id": 4,
  "jsonrpc": "2.0",
  "result": {
    "address": "HNnK9wgUVL6Cjzs1K3jpoGgqQTXCqpAnW8"
  }
}
```

## 2.1.8 Error codes

We should follow the [JSON-RPC](https://www.jsonrpc.org/specification#error_object) specification for error codes, so errors generated by the "server", which in our case, is the wallet handling the requests should be in the `-32099` to `-32000` range.

| Error | Error Type | Description                        |
|------------|------------------------------------|---|
| -32050     | WALLET_NOT_READY | Wallet is not ready yet              |
| -32051     | REQUEST_REJECTED | Request rejected by the user |
| -32052     | MISSING_FUNDS | Missing funds to fulfill transaction |
| -32053 | INVALID_PARAMETERS | Invalid parameters |
| -32099     | GENERIC_FAILURE | Generic failure |

# 3. Guide-level Explanation

### 3.1 `htr_createTx`

When the request is received, we will validate if the parameters are valid, enforcing the data types and rules described in the Reference-Level explanation

If the validation fails, we will respond with a `INVALID_PARAMETERS` error code

If the validation is successful, we will display a confirmation screen, with all outputs for manual user validation (UX is yet to be discussed). If the user rejects the transaction, we will respond with a `REQUEST_REJECTED` error

If the user accepts the transaction, we will create a `SendTransaction` model (or `SendTransactionWalletService` if on the wallet-service) and execute the send transaction.

Since `JSON-RPC` is designed considering only a single response per request, we will only respond the request when the transaction either failed to be pushed to the fullnode (or the wallet-service) or when it's successfully pushed.

## 3.2 `htr_createToken`

When the request is received, we will validate if the parameters are valid by following the parameter type definitions in the Reference-Level explanation

If the validation fails, we will respond with a `INVALID_PARAMETERS` error code

At this point, we will call `prepareCreateNewToken` and display the assembled transaction for user validation (UX is yet to be discussed).

If the user accepts the transaction, we will, just like the `htr_createTx` method, create a `SendTransaction` model and execute the send transaction. Otherwise, we will answer with a `REQUEST_REJECTED` error

## 3.3 `htr_getUtxos`

When the request is received, we will validate if the parameters are valid by following the parameter type definitions in the Reference-Level explanation

If the validation fails, we will respond with a `INVALID_PARAMETERS` error code

At this point, we will call the `getUtxos` method to retrieve `utxos` that fulfill the requested filters.

Before returning, we will display a confirmation screen (UX yet to be discussed) asking for user confirmation by displaying the `utxos` that will be returned to the `dApp`.

If the user accepts the transaction, we will reply with the array of `utxos`. Otherwise, we will answer with a `REQUEST_REJECTED` error

#### Future possibilities

We might display "checkboxes" before each utxos so the user can choose exactly which ones he want to display

## 3.4 `htr_signWithAddress`

When the request is received, we will validate if the parameters are valid by following the parameter type definitions in the Reference-Level explanation

If the validation fails, we will respond with a `INVALID_PARAMETERS` error code

At this point, we will display the `message` and the `address` to the user, asking for his confirmation. If he rejects the modal, we will respond with an `INVALID_PARAMETERS` error code.

If he accepts, we will call the `signMessageWithAddress` and answer

## 3.5 `htr_getBalance`

When the request is received, we will validate if the parameters are valid by following the parameter type definitions in the Reference-Level explanation

If the validation fails, we will respond with a `INVALID_PARAMETERS` error code

At this point, we will display the `message` and the `address` to the user, asking for his confirmation. If he rejects the modal, we will respond with an `INVALID_PARAMETERS` error code.

If he accepts, we will call the `signMessageWithAddress` and answer the request
