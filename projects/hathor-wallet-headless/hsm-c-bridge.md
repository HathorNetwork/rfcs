- Feature Name: hsm_c_bridge
- Start Date: 2023-07-31
- RFC PR:
- Hathor Issue:
- Author: Tulio Vieira tulio@hathor.network

# Summary
[summary]: #summary

This document describes the C Bridge between the Headless Wallet and the [Dinamo Networks Hardware Security System](from [Dinamo Networks](https://www.dinamonetworks.com/en/hardware-security-module-hsm/)) ( HSM ).

# Motivation
[motivation]: #motivation

This bridge is necessary to make NodeJS [client applications](https://github.com/HathorNetwork/rfcs/blob/feat/master/projects/hathor-wallet-headless/hsm-integration.md) communicate with HSM devices while there is no official SDK from Dinamo for the javascript language.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The bridge is comprised mostly of code from the [examples on Dinamo docs site](https://manual.dinamonetworks.io/c/examples.html), added with structures to facilitate communication with the NodeJS through `stdin` input and output interpretation of the `stdout` and/or `stderr`.

The key aspects this bridge must consider are:
### Credentials security
The credentials should be passed in a secure way to the code. For this initial implementation, they can be retrieved from the environment variables directly, since they have already been configured for the _Headless Wallet_ application, and both are running on the same environment.

Other implementations are discussed on the alternatives section below.

### Input / Output security
It was decided to use the `stdin` as a form of input for potentially sensitive data such as transaction input data and derivation paths.

That way this information won't be available to the operating system in any kind of stored history, as would happen if the commmand line parameters were used directly.

## API Endpoints
The main ones will be described below.

### Create offline `xPriv` Key
This will likely be the first interaction between the Hathor environment with the Dinamo HSM, generating the extended private key of a wallet.

A simple command will be sent to the HSM requesting the creation of an `xPriv` with the key name where it should be stored. No response is expected on success.

### Get `xPub` for a wallet
Requests the `xPub` of an `xPriv` associated with a given key name. The resulting derivation will be already on the Hathor coin path.

| ⚠ Incomplete PoC - xPub |
| :--- |
| The HSM API to retrieve the `xPub`  from a generated `BIP32` wallet is still under development. An initial estimate from the Dinamo team is for this to be available on September. A degree of development uncertainty can be found when this API is available. |

### Sign data
Requests a given arbitrary string to be signed using the `xPriv` from the given key name.

| ⚠ Incomplete PoC - sign tx |
| :--- |
| This step was also not tested on the PoC and can add uncertainty to the development phase |

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Installation
To successfully compile any code using the Dinamo libs, it's necessary to, first install their CLI application, available on their [downloads page](https://docs.hsm.dinamonetworks.io/soft_client/downloads).

### Code compilation
After that, for UNIX-like systems, compile the generated file using a command similar to this sample below ( generating a `mainExc` executable from a `main.c` file and linking with the Dinamo shared libraries:
> `gcc -o mainExc main.c -ltacndlib`

Another more robust approach is to use [CMake](https://cmake.org/), generating a makefile based on the local operating system configurations.

### Sample code
Sample code for signing a transaction:
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#include "dinamo.h" /* Dinamo headers */
#include "htrUtils.h" /* Hathor headers */

int main(int argc, char **argv)
{
	// Return value of the latest operation
	int nRet = 0;
	struct AUTH_PWD authPwd;
	HSESSIONCTX hSession = NULL;

	// XPriv Key name
	char szId[] = HTR_TEST_XPRIV_KEYNAME;

	BYTE ver = DN_BCHAIN_VER_WIF_TEST_NET;

	// Address to derive child priv key from
	int nChild = 0;

	// Child Priv Key Name
	char szDest[MAX_OBJ_ID_FQN_LEN] = "HTR_CHILD_KEY";
	DN_BCHAIN_KEY_INFO pbKeyInfo;

	// Message buffer
	BYTE pbMsg[1024] = {0};

	// Output encoding buffer
	char pbOutputEncode[1024 * 2 + 1];
	DWORD dwOutputLength = sizeof(pbOutputEncode);

	// Hash buffer
	BYTE pbHash[DN_BCHAIN_HASH_KECCAK256_LEN] = {0};
	DWORD dwHashLen = sizeof(pbHash);

 	// Signature buffer
	BYTE pbSig[DN_BCHAIN_MAX_SIG_LEN] = {0};
	DWORD dwSigLen = sizeof(pbSig);

	// Public Key retrieval
	DN_BCHAIN_PBK pPbk = {0};

	//=======================================================================
	// Command Line Input Validation example, for alternative implementations
	//=======================================================================
	if (argc < 3) {
		printf("Missing input parameters\n");
		printf("Usage: signOnPath path {number}\n");
		return 1;
	}
	nChild = atoi(argv[2]);

  // Stdin input implementation, the main way to communicate with the bridge
	printf("Insert the message to be signed on stdin: ");
	if (fgets(pbMsg, sizeof(pbMsg), stdin) == NULL) {
		perror("Error reading message input");
		return 2;
	}
	printf("Received message: %s\n", pbMsg);
	printf("Will sign this message with path %d\n", nChild);

	//=========================
	// Initializing Dinamo libs
	//=========================
	nRet = DInitialize(0);
	if (nRet)
	{
			printf("Failure at: DInitialize \nError code: %d\n", nRet);
			exit(1);
	}

	// Initializing data structure to establishing connection with the HSM
	strncpy(authPwd.szAddr, HOST_ADDR, sizeof(authPwd.szAddr));
	authPwd.nPort = DEFAULT_PORT;
	strncpy(authPwd.szUserId, USER_ID, sizeof(authPwd.szUserId));
	strncpy(authPwd.szPassword, USER_PWD, sizeof(authPwd.szPassword));

	nRet = DOpenSession(&hSession, SS_USER_PWD, (BYTE *)&authPwd, sizeof(authPwd), ENCRYPTED_CONN);
	if (nRet)
	{
			printf("Failure at: DOpenSession \nError code: %d\n", nRet);
			goto clean;
	}

	// ==============================================
	// Session established. Initiating operations
	// ==============================================

	// Child Key Derivation operation (temporary data)
	nRet = DBchainCreateBip32Ckd(hSession,
								ver,
								DN_BCHAIN_SECURE_BIP32_INDEX_BASE + nChild,
								BCHAIN_KEY | TEMPORARY_KEY,
								szId,
								szDest,
								&pbKeyInfo,
								0);
	if(nRet){
		printf("Failure at: DBchainCreateBip32Ckd \nError code: %d\n", nRet);
		goto clean;
	}

	/*
		Based on https://manual.dinamonetworks.io/c/sign_verify_bchain_8c-example.html
	*/
	nRet = DBchainHashData(hSession,
													DN_BCHAIN_HASH_KECCAK256,
													pbMsg,
													sizeof(pbMsg),
													pbHash,
													&dwHashLen,
													0);
	if (nRet)
	{
			printf("Failure at: DBchainHashData \nError code: %d\n", nRet);
			goto clean;
	}
  // Outputting in hex enconding for the NodeJS interpreter
	buffer_to_hex(pbHash,
								dwHashLen,
								pbOutputEncode);
	printf("Encoded hash: %s\n\n", pbOutputEncode);

	nRet = DBchainSignHash(hSession,
													DN_BCHAIN_SIG_RAW_ECDSA,
													DN_BCHAIN_HASH_KECCAK256,
													pbHash,
													dwHashLen,
													szId,
													pbSig,
													&dwSigLen,
													0);
	if(nRet){
			printf("Failure at: DBchainSignHash \nError code: %d\n", nRet);
			goto clean;
	}
  // Outputting in hex encoding for the NodeJS interpreter
	buffer_to_hex(pbSig,
								dwSigLen,
								pbOutputEncode);
	printf("Encoded signature: %s\n\n", pbOutputEncode);
	/*
	 * Retrieves the public key from the signature for double-checking
	 */
	nRet = DBchainRecoverPbkFromSignature(hSession,
																				DN_BCHAIN_SIG_RAW_ECDSA,
																				DN_BCHAIN_HASH_KECCAK256,
																				pbHash,
																				dwHashLen,
																				pbSig,
																				dwSigLen,
																				&pPbk,
																				0);
	if (nRet)
	{
			printf("Failure at: DBchainRecoverPbkFromSignature \nError code: %d\n", nRet);
			goto clean;
	}

  // Double checking sinature validity
	nRet = DBchainVerify(hSession,
												DN_BCHAIN_SIG_RAW_ECDSA,
												DN_BCHAIN_HASH_KECCAK256,
												pbHash,
												dwHashLen,
												pbSig,
												dwSigLen,
												pPbk.bType,
												pPbk.pbPbk,
												pPbk.bLen,
												0);
	if (nRet)
	{
			printf("Failure at: DBchainVerify \nError code: %d\n", nRet);
			goto clean;
	}
  // By reaching here, the signature is completely valid

	// =====
	// Clean
	// =====
	clean:

  // No persistent key was built. Just closing the session
	if (hSession) {
			DCloseSession(&hSession, 0);
	}

	DFinalize();
}
```

### Exception Handling
A map will be made between the [Dinamo C library return codes](https://manual.dinamonetworks.io/c/_r_c.html) and the expected results on the Headless wallet in cases of retriable network errors. These are described below:

| Dinamo return code       | Hathor return code |
|--------------------------|:------------------:|
| D_CONNECT_FAILED         |   NETWORK_ERROR    |
| D_SEND_FAILED            |   NETWORK_ERROR    |
| D_RECV_FAILED            |   NETWORK_ERROR    |
| D_SETSOCKOPT_FAILED      |   NETWORK_ERROR    |
| D_CRL_OPERATION_TIMEDOUT |   NETWORK_ERROR    |
| D_CRL_SEND_ERROR         |   NETWORK_ERROR    |
| D_CRL_RECV_ERROR         |   NETWORK_ERROR    |

All other errors should be passed to the Node.js client as fatal errors, along with their Dinamo return codes for easier debugging.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For protocol, network, algorithms and other changes that directly affect the
  code: Does this feature exist in other blockchains and what experience have
  their community had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other blockchains, provide readers of your RFC with a fuller
picture. If there is no prior art, that is fine - your ideas are interesting to
us whether they are brand new or if it is an adaptation from other blockchains.

Note that while precedent set by other blockchains is some motivation, it does
not on its own motivate an RFC. Please also take into consideration that Hathor
sometimes intentionally diverges from common blockchain features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be
and how it would affect the network and project as a whole in a holistic way.
Try to use this section as a tool to more fully consider all possible
interactions with the project and network in your proposal. Also consider how
this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
