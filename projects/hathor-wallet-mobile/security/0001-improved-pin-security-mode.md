# Summary
[summary]: #summary

A new way to use biometrics on the mobile wallet, using a longer pin for security, this does not allow a fallback to the configured pin in case of too many failures.

# Motivation
[motivation]: #motivation

When using biometrics the pin is saved on the phone system, meaning the storage data is still encrypted with the 6 digit pin, using a longer and more random pin makes the storage data harder to crack.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When enabling the biometrics on the mobile wallet we just set a flag and the pin is always saved on the device keychain, this makes it so enabling and disabling biometrics does not require the user to input the pin.
This approach makes it easier to set up biometrics and allows the usual pin screen to be used as a fallback.

The new configuration would use the same configuration for biometrics but use a longer and unguessable password (random 32 bytes for instance), it will trust the system prompt completely, so being locked out due to too many attemps will lock the user out of the wallet.

## Access control

The dependency used to manage keychain password is [react-native-keychain](https://www.npmjs.com/package/react-native-keychain) and we use the [setGenericPassword](https://oblador.github.io/react-native-keychain/docs/api/functions/setGenericPassword) and [getGenericPassword](https://oblador.github.io/react-native-keychain/docs/api/functions/getGenericPassword) to save and retrieve the pin from the keychain.

The [access control](https://oblador.github.io/react-native-keychain/docs/api/enumerations/ACCESS_CONTROL) and [accessible](https://oblador.github.io/react-native-keychain/docs/api/enumerations/ACCESSIBLE) configurations are used with the `setGenericPassword` to manage how this password can be used and by whom.

This new password should be saved with:

- Access control **`BIOMETRY_ANY`**: Constraint to access an item with Touch ID for any enrolled fingers.
- Accessible **`WHEN_UNLOCKED_THIS_DEVICE_ONLY`**: The data in the keychain item can be accessed only while the device is unlocked by the user. Items with this attribute do not migrate to a new device.

## Unlocking the wallet

Since the access data is not encrypted with the pin it cannot be used as a fallback if the user fails the system authentication.
If the user cancels the system authentication prompt or fails too many times it should show a warning along with the "reset wallet" button to allow users to start a new wallet.

## Feature flag

We should create a feature flag to control this feature rollout.
This feature flag should be first enabled to trusted users, then we can incrementally rollout to all users.

## Migration

We should create a configuration migration if a user currently uses biometry the pin should be exchanged by a random password the next time he opens the wallet.
When the user input his pin and it is verified we should:

- generate a random password.
- decrypt the access data with the pin and encrypt with the new random password.
- Save the random password and the pin on the keychain.
- Save mark this migration as complete.

## Disable biometry

Since the access data is not encrypted with the pin anymore we should get the pin from the keychain and encrypt the access data with the pin.
Since the pin is also saved on the keychain this means the user can only do this if he passes the system biometry check.

# Future possibilities
[future-possibilities]: #future-possibilities

## Passcode or biometry

If we could give the user this option he could choose which authentication type he wants (e.g. biometry, passcode, etc.).

## 2 levels of encryption

We could have the keys of the access data encrypted with the pin and the access data object encrypted with the random password.
This would make the wallet only open with a biometry check and sending transactions would require the biometry or passcode.
