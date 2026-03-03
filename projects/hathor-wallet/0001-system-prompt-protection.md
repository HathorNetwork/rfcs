# Summary
[summary]: #summary

Create an option to add system prompt protection for sensitive data.

# Motivation
[motivation]: #motivation

Current desktop wallet encryption requires an user password to protect sensitive data but users may choose weak passwords.
The proposed improvement is to use a higher difficulty encryption key to protect the sensitive data, but this is not viable for users to remember so we will use the system keychain to save this key.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We will use electron's [safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage) to manage encryption and system prompts, this allows us to enable the system prompt only if it is available for the system (using [isEncryptionAvailable](https://www.electronjs.org/docs/latest/api/safe-storage#safestorageisencryptionavailable)).

The system prompt protection will be implemented on each pin and password input on the wallet.
To achieve this we will add an option on the settings menu to use system prompt protection, here we will generate a new random key to use as pin, this random key will be saved encrypted by the system.
This option will also be used on all pin and password inputs to check if system prompt protection is enabled of not.

All attempts to decrypt the key (i.e. pin/password inputs) will be made using the `IPC` communication channel with the main process to call [safeStorage.decryptString](https://www.electronjs.org/docs/latest/api/safe-storage#safestoragedecryptstringencrypted) and get the actual key, this key will be used as the pin or password on any wallet-lib operations.

Check operations are simpler since we only need to check that user has successfully passed the prompt.

The system prompt protection deactivation process will require the user to choose a new password and pin.
# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

// TODO
# Prior art
[prior-art]: #prior-art

This [blog post](https://freek.dev/2103-replacing-keytar-with-electrons-safestorage-in-ray) gives some light to the `safeStorage` module, which is somewhat obscure because this module was meant to be a renderer module for encrypted cookies with `os_crypto`.
This is why the api requires you to save the encrypted value instead of saving on the keychain itself.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Use a keychain dependency

We could use a dependency to manage this and some seem better, e.g. node-keytar, but most, if not all (from what I could find) have been deprecated in favor of `safeStorage`.


