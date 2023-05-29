
## 1. Research and Decide

#### Acceptance Criteria:
1. Research multiple options for our web wallet implementation
2. Design a solution using the selected options and get it approved
3. Implement a working proof of concept on the mobile wallet

#### Tasks:
1. WebWallet research and design with implementation suggestion
2. WalletConnect proof-of-concept implemented on wallet-mobile 

## 2. Proof-of-Concept and Initial Design

#### Acceptance Criteria:
1. We should deploy our initial proof-of-concept implementation of the wallet connect integration to production
	1. The idea here is to go through the whole process of deploying it to detect if we left anything behind or need to have any business decision, like rewriting our privacy guide
2. We should implement an initial design of the RPC inspired on the best ideas from the Ethereum and other non-EVM chains RPC documentation that fulfills our needs
	1. This initial design should be a "MVP", meaning that it will contain only methods that we already know are needed(from our community), but we should make it clear on the design that the API may change at any point

#### Tasks
1. Initial WalletConnect proof-of-concept with sign message RPC
	1. QA on wallet-mobile
2. Study Ethereum RPC and other non-EVM chains APIs
3. Design Hathor's initial RPC
	1. Initial methods should be: "Get Address", "Sign Transaction", "Proof that the wallet owns address X", "Get utxos to fulfill intent"


### 3. Implement the design and initial rollout

#### Acceptance Criteria:
1. We should implement the RPC methods approved on the design on both the wallet-mobile and the wallet-desktop
2. We should be able to allow specific wallets to use it, through our feature toggle mechanism
3. We should make it clear that we might change the RPC methods at any point in the future
4. We should make it clear that we are using wallet-connect as it is and have not implemented any security measures to prevent issues from their side.
5. We should have a mechanism to disable wallet-connect at any point.
6. Before going into production, we need to add a popup explaining that the new privacy changes (if needed) are related to wallet connect

#### Tasks:
1. Implement initial Hathor RPC methods on wallet-mobile
	1. Deep-link on wallet-mobile
2. Implement Initial Hathor RPC methods on wallet-desktop
	1. Deep-link on wallet-desktop
3. Implement a modal popup that informs the user that wallet-connect has been disabled.
	1. This needs a design to decide when to inform 
4. Initial WalletConnect Rollout

### 4. Security Issues

We've detected some security issues that we are not comfortably with on the wallet-connect repositories so before going to production, we need to create some counter measures to protect our users from those issues.

#### Acceptance Criteria:
- Whenever we disable wallet-connect from our unleash mechanism, we should display a message to our users with the reason why we disabled it
- We should create a design with security measures on our side for extra protection
	- If two requests come within X time from each other, inform the user
	- Force the renewal of every `symKey` in storage every X days/minutes...
