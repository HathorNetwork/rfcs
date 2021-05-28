- Feature Name: semi_isolated_networks
- Start Date: 2021-06-01
- RFC PR: [HathorNetwork/rfcs#29](https://github.com/HathorNetwork/rfcs/pull/29)
- Author: Jan Segre <jan@hathor.network>

# Summary
[summary]: #summary

New architecture where we use internal (not connected to all the p2p network) nodes for essential services (wallets and mining).

# Motivation
[motivation]: #motivation

Since our first deploy the nodes that serve essential services have always been connected to the full open p2p network. This has some disadvantages and recently we've experienced some of those disadvantages. Mainly there was a situation with a large number of connection attempts on the p2p network that seems to have starved the nodes of fd handlers which caused the APIs to become unresponsive and in turn the services which rely on those nodes were affected.

The proposed architecture aims to make a semi-isolated network where the nodes that serve these services are not exposed to the open p2p network. This isn't meant to be a solution to any particular problem, and issues like the fd handlers starvation have resulted in specialized solutions/optimizations. This is meant however as a precaution and as a deploy that is generally easier to protect without impacting the network as a whole. For example we will be able to create firewall rules that restrict connections to the p2p port of some nodes, without impacting the connectivity of nodes from other operators, allowing safer targeted responses to some situations.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This guide will make use of the following nomenclature:

- Public nodes: nodes that are able to connect to any other node in open p2p network but never to internal nodes;
- Gate nodes: nodes that can connect to both public nodes and internal nodes, thus acting as _gates_ of the _walled garden_;
- Internal nodes: nodes that have a very restricted set of nodes that it can connect to, they are only directly connected to other internal nodes or gate nodes, and never to public nodes;

At a high level these are the changes:

- Unify node1 and node2 endpoints, today these are handled by two separate CloudFront Distributions, but we would now use only one that can handle these endpoints and optionally a new endpoint that we can add to the wallets in the future. The same distribution can use (and certificate manager also supports this) multiple DNS entries. This entry is the least related to the protection of nodes, but since we're changing our general architecture we can as well do this now;
- Make the wallet nodes internal nodes;
- Make the explorer a gate node (basically so it has maximal visibility of the network);
- Add a gate node for each region there is a wallet node.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation of this architecture already has all the required pieces and parts working:

- hathor-core supports a custom whitelist (and soon will have a netfilter style firewall that might make this simpler to configure);
- aws-deploy-tools supports setting up a custom configuration for a hathor-core node;

Two whitelist are used to achieve this:

- public whitelist: contains all nodes, including the wallets
- internal whitelist: only contains the gates and the wallets

The wallets have to be configured to use the interal whitelist while all other nodes use the public whitelist. The effect of this is that wallets can connect between themselve and with the gates while the gates can connect with everyone.

# Drawbacks
[drawbacks]: #drawbacks

- Because we introduce a new whitelist, some errors can lead to split-brain networks;
- We will need to run and maintain more nodes (the gate nodes), which also incur an increased cost;
- DoS situations might be harder to percieve (wallets would keep working but propagation would be affected), this might require smarter monitoring;

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- We could get away with not using extra gate nodes (or only using the explorer node(s) for this): the cost saving would be small compared to the increased risk of split brain if there is any issue with the explorer (which is more exposed because it serves the explorer frontend);
-

# Prior art
[prior-art]: #prior-art

I haven't found documentation on how Bitcoin (or other larger PoW networks) does this in practice but I heard about similarstrategies being used.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is it worth making the explorer node an internal node, or at least split it into 2 nodes: the main being an internal node and only for the public API we'd use a public node (this split API can be made at the LB lvl);
- Should we unify node1/node2 LBs and only use one region for both?
- Should we really make the node1/node2 change?

# Future possibilities
[future-possibilities]: #future-possibilities

After the netfilter firewall is released we'll have to make changes to the whitelist, it should be simpler to set-it-up such that we won't need to repeat nodes on different lists.

When we the wallet and explorer services are deployed we will have much less exposure of these nodes so we can rethink our architecture to maybe have a common set of core nodes used by these services that would never be exposed to the public.

When sync-v2 is released the open p2p network will be more exposed so we will have to consider an extension of this architecture to keep the connectivity with trusted parties (exchanges, pools, use cases) less exposed.
