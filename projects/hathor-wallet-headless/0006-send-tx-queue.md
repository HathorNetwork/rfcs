- Feature Name: send_tx_queue
- Start Date: 2024-09-20
- Author: Andre Carneiro

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

The wallet-lib implementation of the `PromiseQueue` works by only letting a defined number of tasks to run concurrently, this can be used as a queue of requests to send transactions.

The task on queue will be a simple send-tx lock acquire loop that resolves when the lock is acquired.
After the task resolves the transaction sending can proceed as usual.

Since the wallet-lib `PromiseQueue` works with an underlying `PriorityQueue` we need to change the queue to be able to use a normal `Queue` since a `PriorityQueue` does not guarantee the order of tasks with the same priority.

## Improvements to send-tx process

In a scenario where we want to send many transactions in a short amount of time the queue of requests will automatically improve the throughput by minimizing the time between requests.
Once a transaction is prepared (UTXOs chosen and `Transaction` instance created) the next one will start, instead of the current implementation where we release the lock and wait for the next call.
The client, not knowing when the lock is released may add a long wait time, then the time to make the request itself would make the time to start a new transaction higher.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Wallet-Lib

### PromiseQueue

The `PromiseQueue` implementation needs to be improved to accept another queue implementation (`Queue` in  this case).

The queue methods used by `PromiseQueue` are `isEmpty`, `push` and `pop` so we can easily make both `PriorityQueue` and `Queue` use a compatible interface for `PromiseQueue`.
The `AddTaskOptions` will also need to change since `priority` only works for `PriorityQueue`.

## Headless SendTx Queue

```ts
enum QueueClass {
  PRIORITY,
  FIFO,
}

// We should use the FIFO implementation and concurrency should always be 1.
const sendTxQueue = new PromiseQueue({
  concurrency: 1,
  queueClass: QueueClass.FIFO,
});


// To enqueue a new task

async function acquireLock(walletId: string, signal: AbortSignal): CallableFunction {
  while(!signal.aborted) {
    const canStart = lock.get(walletId).lock(lockTypes.SEND_TX);
    if (canStart) {
      break;
    }

    await delay(100);
  }

  if (signal.aborted) {
    // The task was aborted
    throw new Error('Task aborted');
  }
}

// pseudo logic on the controller

function controllerMethod(req, res) {
  const abortController = new AbortController();
  req.on('close', () => {
    // If a client disconnects, cancel the task
    abortController.abort();
  });

  await sendTxQueue.add(
    async ({ signal }) => { return acquireLock(req.walletId, signal); },
    { signal: abortController.signal },
  );

  try {
    // Send transaction normally
    ...
  } finally {
    lock.get(req.walletId).unlock(lockTypes.SEND_TX);
  }
}
```

# Future possibilities
[future-possibilities]: #future-possibilities

## Retry tasks

To improve reliability we can check any errors on the transaction that are not an impediment (for instance timeout when mining/pushing the transaction) and add the request back in the queue to retry.
This can be configured to retry a few times before actuallly confirming the error.

## Task selection

We could optionally configure the queue to use the `PriorityQueue` so the user can define a priority to his request, allowing transactions to go first depending on how important the user deems them.

The usual requests can also be ordered by using a decreasing priority, starting at $-1$ and decreasing with each new task.
This means that the highest priority task will always be the user defined ones (because they're always positive) then the ones enqueued first.
We can also reset the counter when the queue becomes empty so that we don't decrease the counter forever.

## Fire and forget requests

Usual requests will leave the client connected until the request is finished, this way the user can receive the transaction he created.

A "fire and forget" request will validate the parameters and add a task on the queue to send the transaction.
The response will be immediate and will not wait for the transaction to be completed, i.e. `HTTP 200 { "success": true, "taskId": "foobar" }`.

The task id will be added on a cache so the client can poll for the result.

This allows the transaction "time to send" not bounded by the request timeout.
To change the type of request the client should just add an argument `task` as `true` in the request.

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
