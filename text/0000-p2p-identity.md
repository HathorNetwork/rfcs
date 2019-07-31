- Feature Name: P2P Identity
- Start Date: 2019-07-30
- RFC PR: [MR !12](https://gitlab.com/HathorNetwork/rfcs/merge_requests/12)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

Define how a peer will be identified and the secure protocol of connection between two of them. The peer id is designed in a secure way, so you can trust the peer you are connecting to.

# Motivation
[motivation]: #motivation

The motivation is to increase security in the peer connections and real time communication. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The communication protocol between two peers is message oriented and it's assumed that all messages arrive in the correct order, which is not a problem, since we are using TCP as the transport protocol, that ensures the order.

The peer ID is defined as the hash of the public key of the connection. The peer connection takes place in 3 steps (peer A is trying to connect to peer B):

1. The peer will do a DNS discovery using the peer entrypoint (A will do a text search for B entrypoint), so it can estabilish the initial connection;
2. After that, A sends its peer ID (hash of public key) to B. B must validate that it matches the connection public key;
3. Both peers share their list of entrypoints and must validate that they are in each other list, so A must be in B's list of entrypoints and vice versa.

In case step 2 is not valida, it could be because one of the connections is behind a NAT. In that scenario, we shouldn't refuse this connection but we add a flag to it making it clear that this validation couldn't be completed.

After this initial handshake, the connection is marked as ready to exchange messages and, from this moment on, all real time communication between the connected peers will be encrypted to ensure the confidentiality of the information.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

-

# Drawbacks
[drawbacks]: #drawbacks

We are implementing the p2p network assuming all messages are in the correct order (since we are using TCP, this is correct), however if we stop using TCP this might not be true and we will need to change some part of the code.

Another drawback is that, since we are requiring all connections to use ssl, we are adding an extra processing in the real time communication, which increase the security of the protocol.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- We could use UDP instead of TCP for the transport protocol. This would increase the message exchange speed, however would increase the complexity of the code, because right now we are assuming that all messages arrive in the correct order (which is not true when using UDP);
- We could use WebSocket instead of LineReceiver for the messaging protocol but the advantage is not real clear, so we've decided to keep the LineReceiver.

# Prior art
[prior-art]: #prior-art

-

# Unresolved questions
[unresolved-questions]: #unresolved-questions

-

# Future possibilities
[future-possibilities]: #future-possibilities

- Peer reputation: with a secure method to define a peer id and to estabilish a connection between two peers we could start developing the reputation of one peer, so users can trust more in peers with higher reputation.

# References
[references]: #references