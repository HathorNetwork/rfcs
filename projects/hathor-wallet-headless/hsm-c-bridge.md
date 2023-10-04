- Feature Name: hsm_c_bridge
- Start Date: 2023-07-31
- RFC PR:
- Hathor Issue:
- Author: Tulio Vieira tulio@hathor.network

# Summary
[summary]: #summary

This document describes the C Bridge between the Headless Wallet and the Dinamo Networks Hardware Security System (from [Dinamo Networks](https://www.dinamonetworks.com/en/hardware-security-module-hsm/)) ( HSM ).

# Motivation
[motivation]: #motivation

This bridge is necessary to make NodeJS [client applications](https://github.com/HathorNetwork/rfcs/blob/feat/master/projects/hathor-wallet-headless/hsm-integration.md) communicate with HSM devices while there is no official SDK from Dinamo for the javascript language.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The bridge is comprised mostly of code from the [examples on Dinamo docs site](https://manual.dinamonetworks.io/c/examples.html), with added structures to facilitate communication with the NodeJS through `stdin` input and output interpretation of the `stdout` and/or `stderr`.

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
This will likely be the first interaction between the Hathor environment and the Dinamo HSM, generating the extended private key of a wallet.

A simple command will be sent to the HSM requesting the creation of an `xPriv` with the key name where it should be stored. No response is expected on success.

### Get `xPub` for a wallet
Requests the `xPub` of an `xPriv` associated with a given key name. The resulting derivation will be already on the Hathor coin path.

### Sign data
Requests a given arbitrary string to be signed using the `xPriv` from the given key name.

| ⚠ Incomplete PoC - sign tx                                                           |
|:-------------------------------------------------------------------------------------|
| This step was unfinished on the PoC and can add uncertainty to the development phase |

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Installation
To successfully compile any code using the Dinamo libs, it's necessary to, first install their CLI application, available on their [downloads page](https://docs.hsm.dinamonetworks.io/soft_client/downloads).

### Code compilation
After that, for UNIX-like systems, compile the generated file using a command similar to this sample below ( generating a `mainExc` executable from a `main.c` file and linking with the Dinamo shared libraries:
> `gcc -o mainExc main.c -ltacndlib`

Another more robust approach is to use [CMake](https://cmake.org/), generating a makefile based on the local operating system configurations.

This compilation would be necessary for each kind of environment where the Headless Wallet will be run. For this first version, our focus will be compiling it to [run on Alpine Linux 3.14](https://github.com/HathorNetwork/hathor-wallet-headless/blob/master/Dockerfile), the version used by [our public dockerized application](https://hub.docker.com/r/hathornetwork/hathor-wallet-headless).

### Code Delivery
On environments where this HSM communication will happen, the executables should be moved to the main Headless Wallet application folder, to ensure they will be accessible by the running Node.js process within its docker environment.

Since the usage of this kind of device is a niche of large corporate partners, this should be implemented initially in a dedicated `Dockerfile` for HSM-enabled clients, ensuring a reproducible environment for both tests and production code.

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
	 * Based on https://manual.dinamonetworks.io/c/sign_verify_bchain_8c-example.html
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

| Dinamo return code       | Return value | Hathor return code |
|--------------------------|:------------:|:------------------:|
| D_CONNECT_FAILED         |     -12      |   NETWORK_ERROR    |
| D_SEND_FAILED            |     -13      |   NETWORK_ERROR    |
| D_RECV_FAILED            |     -14      |   NETWORK_ERROR    |
| D_SETSOCKOPT_FAILED      |     -16      |   NETWORK_ERROR    |
| D_CRL_OPERATION_TIMEDOUT |     105      |   NETWORK_ERROR    |
| D_CRL_SEND_ERROR         |     108      |   NETWORK_ERROR    |
| D_CRL_RECV_ERROR         |     109      |   NETWORK_ERROR    |
| ERR_NET_FAIL             |     5001     |   NETWORK_ERROR    |

All other errors should be passed to the Node.js client as fatal errors, along with their Dinamo return codes for easier debugging.

# Drawbacks
[drawbacks]: #drawbacks

### Implementation by third-party
This Bridge approach will be implemented by the Hathor Labs team, not the Dinamo Networks one that is the actual code owner. That way, the possibility of having minor bugs or documentation misunderstandings exist.

The Dinamo Networks team is also working on a Javascript implementation that could be imported as a library through _npm_ directly to the Headless Wallet application, avoiding many of the issues of this simpler approach proposed here. However, the ETA for this library is still undefined.

### Multiple processes running on docker
For applications running on Docker environments, such as the Hathor Headless Wallet, it would be necessary to invoke the C executables from the NodeJS main process. This is not a recommended best practice as mentioned on the [_Decouple Applications_ guideline](https://docs.docker.com/develop/develop-images/guidelines/#decouple-applications).

This could be solved by compiling the C files as _shared object_ (`.so`) file and invoke it directly from the NodeJS through dependencies such as [`node-addon-api`](https://www.npmjs.com/package/node-addon-api) or [`node-ffi-napi`](https://www.npmjs.com/package/ffi-napi). ( Reference [article](https://medium.com/ai-innovation/a-guide-for-javascript-developers-to-build-c-add-ons-with-node-addon-api-28c84a0c0cb1)) ).

By using this approach, it would only be necessary, in each Docker environment with access to the HSM:
- Install the Dinamo libraries on the instance
- Copy the C files provided by Hathor on a temporary folder
- Compile them into shared objects
- Move the compiled objects to a folder within the headless application one
- Remove the temporary C files folder

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Bridge communication through TCP
A more standardized way of communicating with the javascript applications would be through common HTTP and websocket channels.

This would mean this bridge would need to become a standalone daemon and webserver application instead of a single request process, requiring a dockerized environment for itself.

That would increase its maintainability at the cost of the added complexity both for code construction and process maintenance.

Because of that, this solution was discarded as the first version of this client.

### NodeJs to Shared Object communication
By compiling the C files as _shared object_ (`.so`) file and invoking it directly from the NodeJS, the client applications would use a more reliable pattern, used by NodeJS itself, to communicate with a low level C program. This could be implemented through dependencies such as [`node-addon-api`](https://www.npmjs.com/package/node-addon-api) or [`node-ffi-napi`](https://www.npmjs.com/package/ffi-napi). ( Reference [article](https://medium.com/ai-innovation/a-guide-for-javascript-developers-to-build-c-add-ons-with-node-addon-api-28c84a0c0cb1)) ).

By using this approach, it would only be necessary, in each Docker environment with access to the HSM:
- Install the Dinamo libraries on the instance
- Copy the C source files provided by Hathor on a specific folder within the docker instance, accessible by the application
- Interact with the source file directly from the javascript application

It would also be possible to have a single C file acting as a facade, and offering all the APIs as methods to be called instead of separate applications. Another benefit of this approach is to have the bridge application in memory at all times, with no need for firing up/dismantling a whole process within the operating system for each client call.

### Connection and credentials through `stdin`
To increase the flexibility of this bridge while retaining its current architecture, all credentials and connection information could be passed at runtime on each call. That would allow a single Headless Wallet application to connect to many HSM devices.

Since this case is more advanced, it was decided this approach would not offer sensible advantages, and direct access to the environment variables would be simpler and just as secure.

# Prior art
[prior-art]: #prior-art

This approach has not been taken by other applications on Hathor Labs at the time. It is, however, common practice in a UNIX-like environment to use inter process communication like this to allow applications built in different languages to interact.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

As indicated above, the following questions remain to be answered by the proof of concepts:
- Is our code correctly encoding and decoding input data to sign a transaction successfully?

# Future possibilities
[future-possibilities]: #future-possibilities

### HSM available through the default `Dockerfile`
A conditional could be added to add the HSM files through a docker initialization command, removing the need for a dedicated `Dockerfile` for the interested partners.
