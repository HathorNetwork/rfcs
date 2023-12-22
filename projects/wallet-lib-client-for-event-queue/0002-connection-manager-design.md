# Summary
[summary]: #summary

Organize the connection manager classes and move responsabilities from the wallet facade to the connection manager.

# Motivation
[motivation]: #motivation

The connection manager should be responsible for managing the state of the
connection and how the events from the connection are used, this way we can have
a multiple connection classes to handle the different behaviors but maintaining
a common interface.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


## Current implementation

The current `Connection` class implementation is a wrapper around a `WebSocket`
class, the `WebSocket` class is responsible for managing the connection and
actually sending and receiving data.
The `Connection` class will interpret the events from the websocket instance and
emit events that are used by the facade, it also subscribes to specific events
and handles them differently depending on the type of event.

The layers of abstraction make the connection very easy to instantiate and use
but create a black box of events that are interpreted by the facade.

## New connection classes

### FullnodePubSubConnection and FullnodeEventQueueWSConnection

Will connect to the fullnode websocket and subscribe to the apropriate
events, meaning the best block updates and the transaction events.

Transaction events will be inserted into the storage and the history will be
processed (calculating metadata, balance, etc) by the connection instance.
Best block updates will be processed as usual.

#### Events

- `new-tx` and `update-tx`
  - The event data will have the transaction id. // or data
- `best-block-update`
  - The event data will have the new height
- `conn_state`
  - Event data is a `ConnectionState`, so the wallet can know if it is receiving events in real
    time or not.
- `sync-history-partial-update`
  - This is so the wallet facade can update the UI when we are waiting for the
  wallet to load.
  - Will have 3 stages: `syncing`, `processing` and `fetching-tokens`.
    - `syncing` means that the wallet is still fetching the history from the
    fullnode.
    - `processing` means that we are calculating the balance of the wallet.
    - `fetching-tokens` means we are fetching the token data for our tokens.

#### Methods

- subscribeAddresses, unsubscribeAddresse(s)
  - Start/stop listening for transactions with the provided addresses.
- start
  - Start the websocket and add listeners
- stop
  - Close the websocket and remove listeners
- getter methods for the configuration
  - getNetwork, getServerURL

The other methods will be internal.


#### pubsub vs event queue

These classes have the same methods and the same events because they are meant
to be used by the same wallet facade (i.e. `HathorWallet`).
The difference between them is the underlying websocket class and how it is
managed.

The events are also derived in a very different way, the pubsub will receive
only transactions from the wallet but the event queue will receive all events
from the fullnode and will have to filter them locally.

Even with the different implementations since we will expose a common interface
the facade will be able to work with both of them interchangeably.

### WalletServiceConnection

The wallet-service connection does not have to handle transaction processing
since all of this is handled by the wallet-service.
The main concern of this connection is to re-emit events from the wallet-service
and to signal the wallet facade that it needs to update the "new addresses" list.

#### Events

- `new-tx` and `update-tx`
  - The event data will have the transaction id. // or data
- `conn_state`
  - Event data is a `ConnectionState`, so the wallet can know if it is receiving events in real
    time or not.
- `reload-data`
  - When the connection becomes available after it went offline this event will
    be emitted to signal that the data needs to be reloaded.

#### Methods

- start
- stop
- setWalletId
- getter methods for the configuration

The other methods will be internal.

### FullnodeDataConnection

This is the only connection class not meant to be used by a wallet facade but to
be used by an external client to get real-time events from the fullnode, an
example of how to use this is the explorer main page, to update with new
transactions and blocks.

#### Events

- `network-update`
  - network events are related to peers of the connected node or when a new tx
  is accepted by the network.
- `dashboard`
  - Dashboard data from the fullnode.

## Changes to the wallet facade

The wallet facade will be updated to use the new connection classes.
This means that instead of receiving a connection instance the wallet facade
will receive the connection params and options (i.e. `connection_params`) and
will instantiate the appropriate connection class.

The wallet-service facade can only use the `WalletServiceConnection`, but the
`HathorWallet` will need an aditional parameter to choose which connection class
to use.
The `connection_mode` argument will be introduced to resolve this issue, it will
be one of the following:

- `pubsub`
  - Uses `FullnodePubSubConnection`
- `event_queue`
  - Uses `FullnodeEventQueueConnection`
- `auto`

If not provided, the `pubsub` mode will be used, since this is the most common
connection used by our wallets.

Ideally the `auto` mode would decide to use either `FullnodePubSubConnection` or
`FullnodeEventQueueConnection` based on the size of the wallet and other
parameters, but until we have a better strategy it will simply use
`FullnodePubSubConnection`.

## FullnodeEventQueueWSConnection connection managementstate

Now that the connection class is responsible for syncing the history, we can
implement a special logic for the event queue.
When we are reestablishing the connection we can start from the last
acknowledged event.

Usually when the connection is lost we will need to clean and sync the history
of transactions from the beginning since we may have lost some events, but the
event queue allows us to start streaming events from the moment we lost
connection.
This makes it so we don't have to clean the history and can just start the
stream from where we left off.
Although this is only applicable when we are connected to the same fullnode, so
we need to check if the fullnode we are connected to is the same and if it isn't
we will need to re-sync as usual.

To make sure we are connected to the same fullnode we will use the peer-id, an
unique 32 byte identifier.
