- Feature Name: P2P Multiprocess
- Start Date: 2024-09-25
- Initial Document: [P2P Multiprocess](https://docs.google.com/document/d/1afh9ObSgePevcZ8-WNZJU4-LwFVEKqR9F5yTDeppFjU/edit)
- Author: Gabriel Levcovitz <<gabriel@hathor.network>>

# Summary
[summary]: #summary

Some parts of the full node are independent and could run in separate processes, so we can improve the performance and security. We should start separating the P2P network. This would already give us some speed and the full node wouldn't stop responding to HTTP requests when a heavy P2P sync method is called.

# Motivation
[motivation]: #motivation

All full node components (P2P network, HTTP API, consensus, etc) currently run in a single thread, in a single process. They could be split into multiple processes, so we can scale and support more requests and transactions per second.

Also, since the mechanism is blocking, the full node can't reply HTTP requests when we have heavy sync calls in the P2P protocol.

# Guide-level explanation
[Guide-level explanation]: #guide-level-explanation

The problem to be solved by this project can be described as uncoupling the P2P subsystem from the full node. In order to do this, we'll have to refactor all coupling points out of the system. This means we must separate every IO, every callback, and every connection, direct or indirect, between the P2P classes and the rest of the codebase.

Once this is done, we should take this independent P2P subsystem and run it in different processes, ideally one for each connection with the network. Of course, we can't use multithreading in Python because of the GIL (in Python 3.13 you can disable it, but it's still experimental). Even if we could, running on separate processes instead of threads has the added benefit that one process does not interfere with the other, as they have separate memory spaces at the OS level.

During sync P2P agents have to query data from the full node, such as accessing the storage. Then, when processing is done, the agent sends the downloaded vertex to the full node through the `VertexHandler.on_new_vertex` method. This means we must design a way for the subprocess to communicate with the main process, bidirectionally.

Currently, The P2P manager calls a Twisted factory that builds protocols that inherit from `HathorProtocol`, which drives the sync with p2p-related classes.

Ideally we could keep the P2P manager and the factory in a secondary "P2P-main" process, and then each protocol (i.e., connection) would also have its own independent subprocess. The protocols would communicate with the P2P-main process, and in turn the P2P-main process would communicate with the main process to send and retrieve data to/from the full node. In other words, this would mean that every TCP message would be received and handled directly in each respective subprocess, and then the `HathorProtocol` would use some form of IPC to communicate with the `Node` for querying the storage and sending new vertices.

However, this implementation is not straightforward because being in a separate process means using a completely separate Python interpreter, which means using separate Twisted reactors. By default, a Twisted protocol must run in the same reactor as its Twisted factory. To make protocols work on separate processes, we would need at least to work with lower-level socket primitives. At worst, the intrinsic coupling between Twisted factories and protocols could be a blocking factor.

Then, we propose splitting this project into two:

1. **P2P Multiprocess Phase One** - Split the P2P manager, the factory, and its protocols into a new single P2P subprocess. All connections still share the same process between each other, but are isolated from the rest of the full node (Hathor manager, storage, APIs, etc.). This still has benefits for performance, and there could be a monitor on the main process watching the P2P-main process. When it detects failures, staleness, or high memory usage, it kills and restarts the subprocess.
2. **P2P Multiprocess Phase Two** - In this phase, we try to split each connection into a single subprocess. This has the advantage of isolating connections, and therefore preventing an attacker from blocking the whole sync, but with a complexity tradeoff and possibly higher IPC overhead.

By doing this we are able to plan and design each phase independently, possibly with different priorities, since Phase One solves most of our problems.

# Reference-level explanation
[Reference-level explanation]: #reference-level-explanation

According to what was proposed in the Guide-level section, this section details the implementation for P2P Multiprocess Phase One. In order to validate the proposal, I created a [POC](https://github.com/HathorNetwork/hathor-core/pull/1161) that provides a working version of a full node that is able to sync while running P2P components in a single separate subprocess, without blocking the rest of the full node. For information on how to run it and current limitations, check the link.

## Subprocess management

Subprocess management, that is, spawning, terminating, checking for erros, etc., is done using Twisted's `ProcessProtocol` ([docs](https://docs.twisted.org/en/stable/core/howto/process.html)). Using it facilitates implementation as it integrates seamlessly with the Twisted framework, transforming process events in events that are handled by the Twisted reactor. We override handlers to react appropriately.

The caveat is that it takes an executable path in the system as its target, instead of nicely integrated with Python functions (like Python's `multiprocessing` module). We work around this by creating a new P2P "main" file and executing it with Python itself.

## Interprocess Communication

Twisted's `ProcessProtocol` provides a simple way of performing IPC through file descriptors that can be mapped to the parent's file descriptors when the subprocess is spawned. Then, the parent can react accordingly when, for example, the subprocess writes to stdout. This is suitable for simple cases but not so much for our use case of performing bidirectional request/response calls between the processes. Therefore, in the POC we simply ignore this mechanism.

When the subprocess is spawned, it works as a standalone Python process with its own interpreter and reactor. We then use Twisted's AMP protocols to implement IPC. AMP is Twisted's implementation of Asynchronous Messaging Protocol, an abstraction for remote request/response calls that provides (de)serialization, typing of request and response payloads, and handling of remote exceptions ([docs](https://docs.twisted.org/en/stable/core/howto/amp.html)). While the POC uses AMP, in the Drawbacks, alternatives, and future possibilities section we describe other options for this component which could be used in the final version.

We use UDS endpoints to create connections between the processes, and attach the AMP protocols to it. Since we need bidirectional communication, we create two sockets, with one AMP server/client pair for each. In the implementation, we may be able to use the file descriptor mapping described in the previous paragraph to perform those connections, instead of creating our own sockets.

There are multiple request/response calls between the P2P process and the rest of the full node. All those calls must be converted to remote, IPC calls. Since AMP's `callRemote` returns a `Deferred`, we need to convert all of them to `async`. This amounts for a few refactor PRs. They're listed below.

### Calls from the P2P process to the main process

```python
# Calls on VertexHandler
def on_new_vertex(self, vertex: Vertex, *, fails_silently: bool = True) -> bool: ...

# Calls on VerificationService
def verify_basic(self, vertex: Vertex) -> None: ...

# Calls on PubSub
def publish(self, key: HathorEvents, **kwargs: Any) -> None: ...

# Calls on TransactionStorage and indexes
def get_genesis(self, vertex_id: VertexId) -> Vertex | None: ...
def get_vertex(self, vertex_id: VertexId) -> Vertex: ...
def get_block(self, block_id: VertexId) -> Block: ...
def get_latest_timestamp(self) -> int: ...
def get_first_timestamp(self) -> int: ...
def vertex_exists(self, vertex_id: VertexId) -> bool: ...
def can_validate_full(self, vertex: Vertex) -> bool: ...
def get_merkle_tree(self, timestamp: int) -> tuple[bytes, list[bytes]]: ...
def get_hashes_and_next_idx(self, from_idx: RangeIdx, count: int) -> tuple[list[bytes], RangeIdx | None]: ...
def compare_bytes_with_local_vertex(self, vertex: Vertex) -> bool: ...
def get_best_block(self) -> Block: ...
def get_n_height_tips(self, n_blocks: int) -> list[HeightInfo]: ...
def get_tx_tips(self, timestamp: float | None = None) -> set[Interval]: ...
def get_mempool_tips(self) -> set[VertexId]: ...
def height_index_get(self, height: int) -> VertexId | None: ...
def get_parent_block(self, block: Block) -> Block: ...
def get_best_block_tips(self) -> list[VertexId]: ...
def partial_vertex_exists(self, vertex_id: VertexId) -> bool: ...
```

To minimize work, we may refactor calls that are performed by Sync-v2 only, and disable support for Multiprocess P2P when Sync-v1 is enabled, since we'll deprecate it soon.

Since all request and response payloads are classes are serializable, it's not too hard to convert them to IPC calls. When sending vertices though, we must serialize `static_metadata` and send it together. Considering `TransactionMetadata`, there are only some asserts that we may remove (moving them to the main process) so we don't have to serialize and send it. However, in the POC the node only works as a sync client. It's possible that in the sync server version we have other edge cases that use metadata — this has not been checked yet.

About converting them to `async`, this is mostly not too hard since all calls can bubble up to an async TCP message handling call. However, when we introduce `async` calls in the middle of methods that are currently serial, we must be careful to uphold any ordering invariants required by such methods. Using [DeferredLocks](https://docs.twistedmatrix.com/en/stable/api/twisted.internet.defer.DeferredLock.html) may be useful in those situations.

### Calls from the main process to the P2P process

The full node also communicates with the `P2PManager` from some methods, such as endpoints and some control mechanisms:

```python
def start(self) -> None: ...
def stop(self) -> None: ...
def get_connections(self) -> set[HathorProtocol]: ...
def get_server_factory(self) -> HathorServerFactory: ...
def get_client_factory(self) -> HathorClientFactory: ...
def enable_localhost_only(self) -> None: ...
def has_synced_peer(self) -> bool: ...
def send_tx_to_peers(self, tx: BaseTransaction) -> None: ...
def reload_entrypoints_and_connections(self) -> None: ...
def enable_sync_version(self, sync_version: SyncVersion) -> None: ...
def add_peer_discovery(self, peer_discovery: PeerDiscovery) -> None: ...
def add_listen_address_description(self, addr: str) -> None: ...
def disconnect_all_peers(self, *, force: bool = False) -> None: ...
```

Those are trickier to convert to IPC, since some of those types are not serializable, unlike the previous ones. Also conversely, some of them are called from non-`async` contexts and therefore changing them to `async` will be harder. We'll have to examine each call case by case and determine the best solution for each. For example, the `get_connections` method is only called by `Metrics` and it only uses `str(connection.entrypoint)` and `connection.metrics`. Those are in fact serializable, so we should change the method to return them instead of the whole `HathorProtocol`, which is not.


## Not covered by the POC

The following points where mostly ignored in the POC and must be addressed in the implementation:

1. Make sure logging is consistent through all processes
    - All of them should log to the same output.
    - All of them should use the same configuration.
2. Deal with exceptions, specially when a remote call fails
    - Close connections when appropriate.
    - Send errors to the other process when appropriate.
3. Deal with process termination in both ways
    - What happens when the subprocess dies? Kill the connection.
    - What happens when the main process dies? We shouldn't leave zombie subprocesses.

## Drawbacks, alternatives, and future possibilities

I'm condensing those sections into one as they're all related.

### Twisted's AMP

Specifically for using Twisted's AMP, I see two drawbacks:

1. AMP remote calls can only be `async`. If there are any methods that are too hard to convert, this may be an issue.
2. AMP messages have a fixed limit in size: no values can be larger than 65535 bytes. Coincidentally, this the same line length limit that we have in the sync protocol, which means vertices will never be larger. However, it's possible that some of the other serialized types that are sent through IPC reach this limit. If this happens, the call will fail and the sync may stall. We have to consider this.

To fix or prevent those problems, we can use alternative technologies to perform the IPC calls:

#### gRPC

[gRPC](https://grpc.io/) is a robust RPC framework by Google that uses protobufs to serialize messages. It has great support for Python and native support for both synchronous and asynchronous calls, which means it doesn't present the same drawbacks as AMP.

The only "drawback" that I see is that it requires defining static schemas and using code generation for its services, which may not be bad, but may be a bit overkill for this use case. We must algo test whether its more expensive (de)serialization introduces a significant overhead.

#### ZeroMQ

[ZeroMQ](https://zeromq.org/) is a lightweight abstraction over sockets that acts as a concurrency framework. It also has great support for Python and native support for both synchronous and asynchronous calls. It doesn't provide its own serialization like gRPC, so we must provide our own or use its utilities for sending messages using json or Python's pickle.

ZeroMQ is used in the official [bitcoin client](https://github.com/bitcoin/bitcoin/blob/master/doc/zmq.md) to provide a notification system that looks like it's similar to our Event Queue system, which uses WebSockets.

#### Other considerations

However, async support for both gRPC and ZeroMQ are integrated with Python's asyncio. In order to use them with Twisted, we may use the `--x-asyncio-reactor` option in the full node, which is currently experimental, or integrate with lower level future primitives. This has to be experimented.

For ZeroMQ, there are two options, its [native support for asyncio](https://pyzmq.readthedocs.io/en/latest/howto/eventloop.html#asyncio) or the third-party [aiozmq](https://github.com/aio-libs/aiozmq/) lib, which is in the same organization as the famous `aiohttp` lib. The latter has a great abstraction for RPC calls that looks like method calls, but it looks like its repo is stale.

Also, even though both gRPC and ZeroMQ support blocking remote calls, this must be used with caution because bidirectional blocking remote calls may cause deadlocks. When making a call from a client to a server, we may only block if we are sure that there are no callbacks from the server to client in the handling path of the initial request. This may be easy to guarantee if the communication from the P2P process to the main process only uses async calls, and use blocking calls only from the main process to the P2P process. Modularizing the P2PManager may also help with this, isolating its IPC endpoints.

### RocksDB's secondary DB

RocksDB has the ability to open a secondary instance of a DB from another process. This instance is read-only, and a `try_catch_up_with_primary` method has to be actively called to update its contents in relation to the primary instance. This could be used in the P2P process instead of performing IPC calls to the `TransactionStorage` and indexes, but I tested it and it was too slow. Even though the method returned quickly in my tests, the DB would actually take about 5 seconds to be updated with the contents from the primary DB, with the method being called in a loop.

This is unusable when syncing from a full node client perspective, but may be useful for syncing from a server perspective, as we can respond the other full node with outdated data and update the secondary DB in the background. However, that would require a refactor to modularize the server and client protocols. This may also be useful for the Multiprocess HTTP API project for the same reason.

### Rate limit and watchdog

As a future possibility, to make the multiprocess system more robust, we may add a rate limit for IPC calls and a watchdog for killing a subprocess when it uses too much memory, for example. This would prevent an attacker from affecting the full node if an exploit is found in the sync. This may be a separate project implemented between the Phase One and Phase Two described in the Guide-level section.

# Task breakdown

Here's a table of main tasks:

| Task                                                     | Dev days |
|----------------------------------------------------------|----------|
| Experiment with gRPC and ZeroMQ                          | 1        |
| Implement P2P process IPC server (including refactors)   | 2        |
| Implement main process IPC server (including refactors)  | 2        |
| Implement IPC details (logging, exceptions, termination) | 2        |
| Implement CLI command to enable multiprocess P2P         | 0.2      |
| Run benchmarks and tests                                 | 1.8      |
| **Total**                                                | **9**    |

All tasks include writing unit tests for the respective feature.
