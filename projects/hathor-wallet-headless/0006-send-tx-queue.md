- Feature Name: send_tx_queue
- Start Date: 2024-09-20
- Author: Andre Carneiro <andreluizmrcarneiro@gmail.com>

# Summary
[summary]: #summary

Make all APIs that send transactions on the network to enqueue a task instead of checking a lock.

# Motivation
[motivation]: #motivation

A call to send a transaction on the network acquires the individual wallet send-tx lock, any subsequent call that sends transactions will fail until the lock is released.
This is so that only 1 wallet can choose UTXOs at a time to avoid choosing conflicting UTXOs in different transactions.
The wallet-lib can expose the `PromiseQueue` class so that the headless can add tasks to send transactions on the queue instead of rejecting calls.
The goal being to increase transaction throughput since the caller does not need to waste time retrying until the wallet is free.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## PromiseQueue

The wallet-lib implementation of the `PromiseQueue` works by only letting X tasks run concurrently, this can be used as a queue of requests to send transactions.

To minimize the time on queue we could try to have only the UTXO selection on queue, since the wallet facade does not grant such control granularity we can instead add a task to acquire the send-tx lock.
Once the lock is acquired we can prepare the transaction and release the lock, the next task should start while the current execution proceeds to push the tx.

## Overview of operations

Most operations should work similarly to the code below.

```javascript
// Await to acquire lock
const lock = await taskQueue.add(acquireLockTask);

try {
  // Prepare transaction: each api will use a different method
  const sendTx = await wallet.methodX();
  await sendTx.prepareTx();
} finally {
  // Release the lock once the utxo selection is done.
  lock.release();
}

// Mine and push the transaction on the network.
// If this step fails the UTXOs will be automatically released.
await sendTx.runFromMining();
```

## Improvements to send-tx process

### Task selection

The default method os task selection is based on a `PriorityQueue` on which the next task is always the one with higher priority, but the order among tasks of the same priority is not guaranteed.
This means that priority selection is a very important step, we can grant requests that arrive early more priority or add a `priority` argument where the client can give higher/lower priority to a request.

We can also change the queue implementation to use a FIFO (First In First Out) queue which would make priority not relevant.

The type of queue used will be configured on a per-instance basis under `sendTxQueueType` with the options `priority` and `fifo`.
Or with the environment variable (when running on docker) `HEADLESS_SEND_TX_QUEUE_TYPE` with the same options.

### Fire and forget requests

While usual requests will leave the client connected until the request is finished, so the client will receive the complete transaction when it finishes, the task is also aborted if the client disconnects.

A "fire and forget" request will add a similar task to the queue but will return 200 to the client with a task id once the task is enqueued.
The task id will be added on the task cache, so the client can poll for the completion status.

Using the "fire and forget" tasks will unblock the client and not tie the time to send the transaction with the client request timeout.
To change the type of request the client should just add an argument `task` as `true` to the headless.

### Time to send transactions

In a scenario where we want to send many transacttions in a short amount of time the queue of requests will automatically improve the throughput by minimizing the time between requests.
Once a transaction is prepared (UTXOs chosen and `Transaction` instance created) the next one will start, instead of the current implementation where we release the lock and wait for the next call.
The client, not knowing when the lock is released may add a long wait time, then the time to make the request itself would make the time to start a new transaction higher.

## SendTx Task Cache

The cache will be a simple Map, where we store the task status indexed by the `taskId`.

### Task status API

<details>

 <summary><code>GET</code> <code><b>/wallet/tasks/send-tx/:taskId</b></code> <code>(Get the status of a send-tx task)</code></summary>

This API will return the task status from an internal cache.


##### Parameters

> | name        | type     | data type | description              | location |
> | ----------- | -------- | --------- | ------------------------ | -------- |
> | taskId      | required | string    | The id of the task       | path     | 
> | x-wallet-id | required | string    | Wallet owner of the task | header   |

##### Responses

> | http code | content-type       | response                                                          |
> | --------- | ------------------ | ----------------------------------------------------------------- |
> | `200`     | `application/json` | `{"success":true, "code": 3, "status": "Done", "txId": "abc123"}` |
> | `400`     | `application/json` | `{"success": false, "message":"Bad Request"}`                     |
>
> Where the possible status are:
> - Waiting (0)
> - Executing (1)
> - Error (2)
> - Done (3)

##### Example cURL

> ```javascript
>  curl -X POST -H 'X-Wallet-ID: main' 'http://localhost:8000/wallet/tasks/send-tx/123'
> ```

</details>



# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Wallet-Lib

### PromiseQueue

The `PromiseQueue` implementation needs to be improved to accept another queue implementation (`Queue` in  this case).

## Headless SendTx Queue

```ts
const sendTxQueue = new PromiseQueue();
// Concurrency should always be 1.
sendTxQueue.concurrency = 1;


/**
 * @param {string} taskId
 * @param {string} walletId
 * @param {AbortSignal} signal - Used to abort the transaction
 * @param {string} walletId
 */
async function waitForLock(taskId, walletId, signal, priority) {
  // TODO: Create task status as waiting
  try {
    await sendTxQueue.add(async ({ signal }) => {
      // Exit with error on abort
      signal?.throwIfAborted();

      while (!sendTxLock.get(walletId).lock(lockTypes.SEND_TX)) {
        await new Promise<void>(resolve => {
          setTimeout(resolve, 100);
        });
        signal?.throwIfAborted();
      }
      // TODO: Move task status to Executing
    }, { signal, priority });
  } catch (error) {
    // TODO: Move task status to Error
    throw error;
  }
}
```

## Headless SendTx Task Cache

```ts
type TaskWaiting = {
  status: 'Waiting';
  code: 0;
};

type TaskExecuting = {
  status: 'Executing';
  code: 1;
};

type TaskError = {
  status: 'Error';
  code: 2;
  error: Error;
  message: string;
  finishedAt: number;
};

type TaskDone = {
  status: 'Done';
  code: 3;
  txId: string;
  finishedAt: number;
};

type TaskStatus = TaskWaiting | TaskExecuting | TaskError | TaskDone;

const sendTxTasks = new Map<string, TaskStatus>();
const CACHE_CLEAN_TIMEOUT = 60000; // 1 minute
// Timer is just for safe cleanup when the wallet shuts down.
const timer: ReturnType<typeof setTimeout> = null;

function cleanCache() {
  const now = Date.now();
  for (const [taskId, status] in sendTxTasks.entries()) {
    const shouldClean = status.finishedAt && (now - status.finishedAt > CACHE_CLEAN_TIMEOUT);
    if (shouldClean) {
      sendTxTasks.delete(taskId);
    }
  }

  timer = setTimeout(cleanCache, CACHE_CLEAN_TIMEOUT/2);
}
```

The sendTx cache should be created on a "per-wallet" basis.


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

## Retry tasks

To improve reliability we can check any errors on the transaction that are not an impediment (for instance timeout when pushing the transaction) and add the request back in the queue to retry.
This can be configured to retry a few times before actuallly confirming the error.
