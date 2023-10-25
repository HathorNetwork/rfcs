- Feature Name: scaling_use_cases
- Start Date: 2021-07-21
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

This document discusses different approches to integrate applications with Hathor where the number of users might grow indefinitely.

# Motivation
[motivation]: #motivation

There are multiple ways to developing such integrations and it is not trivial to decide which approach works best for each case.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The context here is that a company has a growing user base, and each user has one or more addresses in Hathor. We can say that each user has a wallet, but we need to carefully understand what "wallet" means in this case.

The company has custody of its users' tokens. In other words, the users' wallets are fully controlled by the company.

Let's explore different ways of generating and maintaining the users' wallets.

<!-- ##### -->
<!-- ##### -->

## Global Wallet

In this case, the company will run one global wallet. Each user will be associated to one or more addresses of the global wallet through their indexes.

The global wallet has an unlimited number of enumerable addresses. Each index will be associated to a user. For instance:

```
address_index | user
--------------|-----------
            0 | Alice
            1 | Bob
            2 | Alice
            3 | Charlie
            4 | Charlie
            5 | Dave
            6 | Eve
            7 | Alice
```

Notice that each index belongs to only one user. But the same user may have one or more indexes.

This approach does not scale easily because the global wallet will have to keep track of more and more addresses over time. The more users, the more addresses. The more addresses, the more memory is required.

Another challenge is to scan the transactions of the global wallet. The most common stop criteria for the scanner is the GAP Limit, which stops the scanner when `N` addresses in a row are empty. An address is empty when it has no transactions.

To illustrate the problem, let's say the GAP Limit is three. If Charlie and Dave have not made any deposits, addresses #3, #4, and #5 will be empty, and the wallet will not find the deposit made by Eve in address #6.

A temporary solution might be to increase the GAP Limit, but the issue will likely happen again in the future.

Another solution might be to artificially generate transactions every three users.

<!-- ##### -->
<!-- ##### -->

## Global Wallet with Derivation Paths

This case is similar to the Global Wallet. But, instead of associating one index per user, it associates one derivation path per user. For instance:

```
derivation_path | user
----------------|----------
  m/44'/0'/0'/0 | Alice
  m/44'/0'/0'/1 | Bob
  m/44'/0'/0'/2 | Charlie
  m/44'/0'/0'/3 | Dave
  m/44'/0'/0'/4 | Eve
```

Notice that each user has one, and only one, derivation path.

Each derivation path leads to an unlimited number of enumerable addresses. Thus, each user has addresses #0, #1, #2, #3, and so on. Notice that Alice's address #0 is different from Bob's address #0 because they had different derivation paths.

This approach has a similar scaling issue of the Global Wallet. The wallet will have to keep track of all these derivation paths and their addresses.

Another challenge is to scan the derivation paths. The most common stop criteria for the scanner is the GAP Limit M/N, which stops the scanner when `M` derivation paths in a row are empty. It considers that a given derivation path is empty when its first `N` addresses are empty.

## One Wallet Per User

This case differs from the previous ones because there will be multiple wallets running.

First of all, there are multiple ways to associate a wallet with a user. To illustrate, some ways are: (i) one seed per user, (ii) one global seed with one passphrase per user, or (iii) one global seed with one derivation path per user.

The challenges here are the same regardless the way the wallet is associated with a user.

The naive implementation here is to run one wallet per user. But it would consume a lot of resources (CPU and memory) and it is hard to scale to millions of users.

A better implementation is to run a pool of wallets that automatically starts and stops wallets on demand. For instance, when a request is made to a wallet that is not running, the pool starts that wallet automatically. When a wallet is idle for too long, it is automatically stopped.

Such pool would consume resources according to the number of concurrent users (instead of total users). It can easily scale up through the addition of more servers to the pool.

## Full-length Wallet Service

A Full-Length Wallet Service is a service that keeps track of all addresses in the blockchain and allows the creation of wallets.

Wallets can have two types: manual or xpub.

- Manual wallets have a list of addresses that is manually maintened, i.e., one must manually add or remove addresses from the wallet.
- Xpub wallets have an xpub, and the addresses are scanned using the GAP Limit stop criteria.

In this case, it is easier to use xpub wallets and ensure that the deposits meets the requirements of the GAP Limit.

# Useful Links

1. [BIP-0032: Hierarchical Deterministic Wallets](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
2. [BIP-0039: Mnemonic code for generating deterministic keys](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
3. [BIP-0044: Multi-Account Hierarchy for Deterministic Wallets](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
4. [SLIP-0044: Registered coin types for BIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md)
