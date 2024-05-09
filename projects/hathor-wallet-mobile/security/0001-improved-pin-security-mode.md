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

The dependency used to manage keychain password is [react-native-keychain](https://www.npmjs.com/package/react-native-keychain) and we use the [setGenericPassword](https://github.com/oblador/react-native-keychain?tab=readme-ov-file#setgenericpasswordusername-password--accesscontrol-accessible-accessgroup-service-securitylevel-) and [getGenericPassword](https://github.com/oblador/react-native-keychain?tab=readme-ov-file#getgenericpassword-authenticationprompt-service-accesscontrol-) to save and retrieve the pin from the keychain.
Since the new security mode and the current biometrics mode cannot work at the same time we will only use 1 password saved at a time on the keychain (instead of having both pins saved).

The [access control](https://github.com/oblador/react-native-keychain?tab=readme-ov-file#keychainaccess_control-enum) and [accessible](https://github.com/oblador/react-native-keychain?tab=readme-ov-file#keychainaccessible-enum) configurations are used with the `setGenericPassword` to manage how this password can be used and by whom.

This new password should be saved with:

- Access control **`BIOMETRY_ANY_OR_DEVICE_PASSCODE`**: Constraint to access an item with Touch ID for any enrolled fingers or passcode.
	- This way either biometrics or system pin/pattern will unlock the wallet
- Accessible **`WHEN_UNLOCKED_THIS_DEVICE_ONLY`**: The data in the keychain item can be accessed only while the device is unlocked by the user. Items with this attribute do not migrate to a new device.

## Settings

The new configuration should not replace the current one, it will be presented to the user as an option called "Enable system authentication" on the settings page, once clicked the enabling process will start on a new screen.

### Enabling process

When trying to enable system authentication the user will be required to confirm that this is what he wants, this screen also has a more detailed explanation of what will be enabled along with a warning that the wallet pin unlock will be disabled.

After conffirmation the user will be required to input the pin (if biometrics is enabled it can also be used here). The pin will be used to decrypt the access data and encrypt with the new pin.

The new pin will be generated randomly the access data will then be encrypted with the new pin and both will be saved.

### Disabling process

To disable we will have a similar confirmation prompt as the first step and the system authentication as the next step to get the current pin.
The user will then have to create a new 6 digit pin and confirm it, once the new pin is chosen the access data will be encrypted with the new pin and the wallet will be restarted.

## Unlocking the wallet

The pin screen to unlock the wallet will have to know that when the system authentication is turned on, the user cannot default to the pin input since the actual pin is not 6 digit anymore, so if the user cancels the system authentication prompt or fails too many times it should show a warning to close the wallet and open again to try one more time along with the "reset wallet" button to allow users to start a new wallet.

## Feature flag

Since this is a new mode and does not impact the current ones we should keep the settings part of enabling this hidden under a feature flag and release it to some trusted users to review the feature and check for friction in the user experience.

# Drawbacks
[drawbacks]: #drawbacks

The new mode is more strict and may be confusing to new users since we already have a biometrics option.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could just change the biometrics mode for this new one, but I think we should keep the new mode in a feature flag and in the future slowly release this feature.
Initially users may have both biometrics and system authentication as options, then we can slowly phase out the old biometrics if we sense that the new system is being well received by the userbase.