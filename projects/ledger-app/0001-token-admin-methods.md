# Summary
[summary]: #summary

Support for admin operations for custom tokens on the Ledger app.
This includes:

- Creating tokens and NFTs.
- Minting new tokens and NFTs.
- Melting new tokens and NFTs.
- Delegating authorities.
- Destroying authorities.

# Motivation
[motivation]: #motivation

Today Hathor's Ledger app only allow sending transactions with custom tokens, so users and use-cases can hold and send custom tokens but to create and manage token supply it is required to use a software wallet.
This design is to allow users who use Ledger, a hardware cold wallet, to fully manage custom tokens without using any software wallet.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Before entering the operations, we should address the differences between custom tokens and NFTs.
In Hathor, NFTs are custom tokens, where the creation transaction (which is the uid of the NFT) has the NFT data associated which is usually an IPFS link to the NFT document (image, pdf, video, etc.).
After the creation the operations and transactions involving NFT are exactly the same regarding the protocol, but the balance is shown differently to the user, where the custom token unit is worth 0,01 the NFT is always integer, so it is 1. Both of these are the same value on the transaction it is just shown different on the UI.
Since the Ledger cannot check if a token is an NFT or not, we should disregard any changes and treat both as custom tokens (which is valid if we follow the protocol).

## Transaction parsing updates

To allow admin operations we will introduce new concepts into the app that were being ignored or not treated.

---

The `token_data` of an output is a byte with the information:

- Whether the token is a custom token or HTR
- Which custom token it is (index on the token array)
- If the output is an authority output (which authority will be encoded on the `value`)
  - Currently on the app we ignore the last piece of information since we do not allow authority outputs. The way we do it is by throwing an error if `token_data & TOKEN_DATA_AUTHORITY_MASK` is not 0. To allow authority outputs we just need to change this and create a special screen for authority outputs when the user is confirming them.
  
Usually the output confirmation screen involves 4 steps:

- Address step (with pagination)
- Amount step (Showing as `ABC 0.01` or `HTR 0.01`, with pagination for big values)
- Approve step
- Reject step

When confirming an authority output we should show:

- Address step (with pagination)
- Type step (`mint authority` or `melt authority`)
  - Other authorities will throw.
  - We will not allow joined authority, for instance `mint+melt authority` (i.e. `value = 3`)
- Approve step
- Reject step

Obs: With authority outputs we already have enough to enable mint, melt, delegate and destroy operations.

---

Currently we only allow P2PKH scripts (and addresses) but NFTs have a data output.
Data outputs are allowed on the protocol but we will only allow on NFT creation scripts because any other is just an HTR burn.
Data outputs are allowed to use custom tokens on the protocol but we will only allow HTR (token index 0) since NFT creation requires HTR.

The user will be required to confirm the data on the script, which means that only ascii characters will be allowed since Ledger does not support UTF-8.

---

When creating a custom token or NFT the transaction information also includes the name and symbol of the token, these should only exist when a transaction has the version byte 2.

---

> NOTE: Version field has been split into `signalBits` and `version`
> The version field of the transaction was 2 bytes (first byte was reserved for later use) and since Ledger app version 1.1.0 the version field has become a 1 byte field and the first byte became the `signalBits`

Version byte needs to be refactored to a 1 byte field and support for `signalBits` will not be added in this design so we will only allow `0x00` as `signalBits` for now.

## Admin operations

### Mint, Melt, Delegate and Destroy

Sending a mint, melt, delegate or destroy transactions should use the same protocol as sending normal transactions.
Mint and Melt involve destroying/creating custom token outputs and since the Ledger app does not have access to the input data it can only see the outputs, which means that a mint operation is a transaction with custom tokens outputs (possibly HTR outputs for change) and possibly authority outputs, same for melt operations.

Delegate is simpler since its actually just sending an authority output.

Destroy operations usually do not even have authority outputs, so it actually is already supported.
Granted that the token array will have the token uid of the authority being destroyed so the user still has to sign the token data even with no outputs of the token.

### Create token

Token creation transactions are special for a number of reasons:

1. Transaction version byte is `0x03`
2. Extra fields at the end of the transaction, namely `token_info_version`, `name` and `symbol`
3. There will be an output with the created token, the token uid will not be on the tokens array since it does not exist yet
    1. Token uid is the transaction id of the created transaction, which is created during mining.

To deal with these we will create a new command to load the token data of the token being created.
We will use the existing `SEND_TOKEN_DATA` for instruction code `0x08` and use the `P1` field to indicate which data is being sent.
The first command in the table below is the existing `SEND_TOKEN_DATA` command, we will use `P1` of the APDU protocol to switch to the correct handler.

| Description                                 | P1   | P2   | CData                                                                                                                                                                   |
| ------------------------------------------- | ---- | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Send token data<br>for signing transactions | 0x00 | 0x00 | `version (1)` \|\|<br> `uid (32)` \|\|<br>`symbol_len (1)` \|\|<br>`symbol (symbol_len)` \|\|<br>`name_len (1)` \|\|<br>`name (name_len)` \|\|<br>`signature (32)`      |
| Send token data<br>for creating a token     | 0x01 | 0x00 | `version (1)` \|\|<br> `symbol_len (1)` \|\|<br>`symbol (symbol_len)` \|\|<br>`name_len (1)` \|\|<br>`name (name_len)`                                                  |
| Send token data<br>for creating an NFT      | 0x02 | 0x00 | `version (1)` \|\|<br> `symbol_len (1)` \|\|<br>`symbol (symbol_len)` \|\|<br>`name_len (1)` \|\|<br>`name (name_len)` \|\|<br>`data_len (1)` \|\|<br>`data (data_len)` |

The new commands will require a new flow for user confirmation

1. Screen showing the user this is a "Create token" data.
2. Screen with the symbol
3. Screen with the name
4. If creating an NFT a screen with the data (ascii encoded), paginated
5. Approve screen
6. Reject screen

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Authority outputs can be inferred through the `token_data` and `value` so we should not add extra fields but methods to check if an output (`tx_output_t`) is an authority and which authority it is.

```c
bool is_authority_output(tx_output_t output) {
    return (output.token_data & OUTPUT_AUTHORITY_MASK) > 0
}

bool is_mint_authority(tx_output_t output) {
    return is_authority_output(output) && ((output.value & MINT_AUTHORITY_MASK) > 0);
}

bool is_melt_authority(tx_output_t output) {
    return is_authority_output(output) && ((output.value & MELT_AUTHORITY_MASK) > 0);
}
```

For data outputs we will introduce a new `data` field which will hold the actual data script.
We should fail if the data script is not ascii.

```c
typedef struct {
    uint8_t index;
    uint64_t value;
    uint8_t token_data;
    uint8_t pubkey_hash[PUBKEY_HASH_LEN];
    // New data field
    uint8_t data[MAX_DATA_SCRIPT_LEN]
} tx_output_t;
```

---

During the transaction processing we use the method `receive_data` which internally calls `_decode_elements` untill we reach a fully formed output (which triggers a user confirmation) or we reach the end of the buffer (which returns a request for more data) when we parse all outputs (as defined in the initial portion of the wallet) and there is no more data to parse we start the UI step to request the user for a final tx confirmation.

To add support for our new transactions we require some changes on output confirmation.

## Output confirmation

The method `ui_display_tx_outputs` prepares the outputs (address base58 encoding, value formatting for UI, etc.) and shows the screens with the output information to the user (with `ux_display_tx_output_flow`).
To adjust this we need to check if the output is a P2PKH output, a data output or an authority output, each output type have different data for the user to check so it makes sense that each has its own flow (i.e. screens that display information and ask confirmation or rejection).

- normal P2PKH outputs
  - `pubkey_hash` field of the output should not be empty.
  - `token_data` should indicate that this **IS NOT** an authority output.
  - This is the current supported format.
  - For a token creation, we can check if this is the token being created and use the loaded symbol for the amount.
- authority P2PKH outputs
  - `pubkey_hash` field of the output should not be empty.
  - `token_data` should indicate that this **IS** an authority output.
  - Should confirm the address and authority of the output.
  - The authority step should have the title "Authority" and text either "Mint \<symbol\>" or "Melt \<symbol\>" depending on the type of authority.
- Data outputs
  - Should only exist if the transaction version byte is `0x03`
  - `data` should be ascii safe
  - `value` should always be 1
  - No need for confirmation, we just need to check that the data matches the user confirmed data.

## Transaction parsing

The current process to parse the transaction will parse the 2 version bytes then the length of tokens, intputs and outputs (1 byte each) after this we parse the token uids, inputs and outputs as the length bytes described.
For "create token transactions" (identified by version byte `0x03`) we have some extra fields at the end of the transaction, these fields should already be confirmed by the user so we just need to make sure the transaction information matches the user confirmed data.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Separate create tx command

We could create a separate command for create token transactions, but the transaction parsing methods are very complex and duplicating this code is not desirable.

## Do not create a new `SEND_TOKEN_DATA` command

Instead of the new command to load the token data being created we could just confirm the data when they appear on the transaction parsing, this would create an issue for outputs containing the token being created since during parsing the symbol is unknown (since it is the last piece of information) so we would need to show the amount as "Amount created" or similar (same with authorities) to signify that the output is of the created token.

# Future possibilities
[future-possibilities]: #future-possibilities

## Nano contract

For signing we just require an extra signature for the caller that can be called during the usual process.
The issue is that we need to parse the nano contract data and add a confirmation step for these.
Since the arguments can be very technical the app UX will not be very user friendly.
This UX challenge should be addressed in its own design.

## P2SH

We currently only support P2PKH scripts and we are adding support for data scripts (under strict conditions).
Identifying P2SH addresses in the transaction is possible but the difficulty of generating P2SH addresses has some is the lack of the `redeemScript` which is required to derive the address, since we would require the multisig configuration to be loaded in the app and we do not have support for this yet.
Without address derivation we cannot check that any addresses are from our wallet (making change output checks fail).
Also, we do not have permission to derive in the Hathor multisig path derivation (see [here](https://github.com/HathorNetwork/hathor-wallet-lib/blob/v1.4.0/src/constants.ts#L267)) so this would also need to change.

# Task breakdown

- Parse version separate from signalBits and verify signalBits is `0x00` (0.5 dev day)
- Parse authorities from token_data when parsing outputs (1 dev day)
- Create new screen for authority output confirmation (2 dev days)
- Create new command to allow token and NFT creation (3 dev days)
- New integration tests for new commands and outputs (2 dev days)
