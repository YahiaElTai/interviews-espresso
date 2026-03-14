# Concurrency & Parallelism

> **21 questions** — 14 theory, 7 practical

- Concurrency vs parallelism: definitions, single-core interleaving vs multi-core execution, how Node.js achieves concurrency without parallelism
- Concurrency hazards: race conditions, deadlocks, starvation, livelocks
- Node.js event loop: phases (timers, pending, poll, check, close), microtask queue starvation, CPU-bound blocking symptoms, starvation causes
- Threading models: single-threaded event loop vs thread-per-request vs goroutines — cooperative vs preemptive scheduling, how each handles long-running tasks
- Race conditions in async/await: check-then-act, concurrent API calls, lost updates
- Deadlocks: four conditions, database transactions, distributed locks, Promise chains
- Mutexes, semaphores, and locks: multi-threaded synchronization, Node.js single-threaded equivalents (async mutex, queue-per-key)
- Async patterns: callbacks, Promises, async/await — evolution, subtle bugs (lost stack traces, unhandled rejections), error propagation differences
- I/O-bound vs CPU-bound work: why Node.js excels at I/O, offloading options for CPU (worker threads, child processes, separate services), and how to identify which category a workload falls into
- Worker threads: main thread/worker communication, SharedArrayBuffer vs message passing
- libuv thread pool: which operations use it, default size, saturation behavior
- Backpressure: fast producer / slow consumer problem, how Node.js streams handle it automatically, memory exhaustion from ignoring highWaterMark signals
- Concurrency limiting: unbounded async operations (Promise.all over thousands of items), connection pool saturation, patterns like p-limit and async queue
- Promise combinators: all, allSettled, race, any — behavior differences, error handling, and when each fits

---

## Foundational

<details>
<summary>1. What is the difference between concurrency and parallelism — why is the distinction important, how does single-core interleaving differ from multi-core simultaneous execution, and why does Node.js achieve concurrency without parallelism on the main thread?</summary>

**Concurrency** is about *dealing with* multiple things at once — structuring your program so multiple tasks can make progress. **Parallelism** is about *doing* multiple things at once — actually executing instructions simultaneously on multiple CPU cores.

**Single-core interleaving vs multi-core execution:**

- On a single core, concurrency is achieved by rapidly switching between tasks. Only one task runs at any instant, but by interleaving (task A does some work, pauses, task B does some work, pauses, back to A), all tasks make progress. This is concurrency without parallelism.
- On multiple cores, tasks literally execute at the same instant on different cores. This is parallelism. Parallelism requires concurrency (you need multiple tasks to run in parallel), but concurrency doesn't require parallelism.

**Why Node.js is concurrent but not parallel on the main thread:**

Node.js runs all JavaScript on a single thread. When an async operation (like a database query) is initiated, Node doesn't block — it registers a callback and moves on to the next task. When the I/O completes, the callback is queued and eventually executed. This means thousands of I/O operations can be "in flight" simultaneously, but JavaScript code only executes one piece at a time. The event loop is the scheduler that interleaves these tasks.

**Why the distinction matters:** It determines what problems you face. Parallelism introduces shared-memory hazards (data races, need for locks). Concurrency without parallelism (Node.js) avoids data races but still has logical race conditions across `await` boundaries — a different class of bugs that developers often overlook because they assume "single-threaded = safe."

</details>

<details>
<summary>2. How do the three major threading models — single-threaded event loop (Node.js), thread-per-request (Java/Spring), and lightweight coroutines (Go goroutines) — each handle concurrent workloads, what is the difference between cooperative and preemptive scheduling, and what happens in each model when a task takes a long time to complete?</summary>

**Single-threaded event loop (Node.js):**
One thread runs all JavaScript. I/O is delegated to the OS or libuv thread pool. The event loop picks up completed I/O callbacks and runs them. Scheduling is **cooperative** — each piece of code runs to completion (until it hits an `await` or returns), then yields control back to the event loop.

**Thread-per-request (Java/Spring traditional):**
Each incoming request gets its own OS thread. The thread blocks on I/O (waiting for DB response, file read, etc.). The OS kernel uses **preemptive scheduling** — it can interrupt any thread at any time to give another thread CPU time. Thousands of requests = thousands of threads, each consuming ~1MB of stack memory.

**Lightweight coroutines (Go goroutines):**
Goroutines are user-space "green threads" multiplexed onto a small number of OS threads by the Go runtime. They're cheap (~4KB initial stack). Scheduling is **partially preemptive** — the Go runtime inserts yield points at function calls and can preempt long-running goroutines (since Go 1.14). This gives the best of both worlds: lightweight like an event loop, but can utilize multiple cores.

**Cooperative vs preemptive scheduling:**
- **Cooperative**: A task must explicitly yield (by hitting an `await`, completing, or calling a yield function). If it doesn't yield, nothing else runs.
- **Preemptive**: The scheduler can forcibly interrupt a running task to give time to others, regardless of what the task is doing.

**What happens when a task takes a long time:**

| Model | Long-running CPU task | Long-running I/O |
|---|---|---|
| **Node.js** | Blocks the entire event loop. No other request can be processed. Healthchecks fail, timeouts cascade. | No problem — I/O is non-blocking, the event loop continues handling other work. |
| **Thread-per-request** | Only blocks that one thread. Other threads continue. But if many requests do this, thread pool exhaustion occurs. | Blocks that thread (unless using async I/O). Thread sits idle, consuming memory while waiting. |
| **Go goroutines** | Runtime preempts the goroutine at yield points, other goroutines still run. Truly CPU-bound tight loops can still cause issues, but the runtime handles most cases. | Goroutine is parked (nearly zero cost), OS thread is freed for other goroutines. |

**Key tradeoff:** Node.js is the most memory-efficient for I/O-heavy workloads but the most fragile under CPU load. Thread-per-request is simplest to reason about but wastes memory. Go balances both but requires understanding goroutine lifecycle and channel patterns.

</details>

<details>
<summary>3. Why does Node.js excel at I/O-bound work but struggle with CPU-bound work — what is the fundamental architectural reason, what symptoms appear when CPU-bound work runs on the main thread, and what are the options for handling CPU-intensive tasks in a Node.js application?</summary>

**The fundamental architectural reason:**

Node.js has a single thread for JavaScript execution. When that thread is waiting for I/O, it's not actually waiting — it delegates to the OS (epoll/kqueue/IOCP) and processes other callbacks. This means one thread can handle tens of thousands of concurrent I/O operations. But when the thread is doing CPU work (parsing, computation, encryption), it's genuinely occupied — nothing else can run until it finishes.

**Symptoms of CPU-bound work on the main thread:**

- **Event loop lag** — all request handlers stall. A 200ms computation means every concurrent request waits at least 200ms, even simple healthchecks.
- **Increased tail latency** — p99 latencies spike because requests queue behind the CPU-bound work.
- **Healthcheck failures** — load balancers mark the instance as unhealthy because it can't respond to pings.
- **Timeout cascades** — upstream services timeout waiting for responses, triggering retries that make things worse.
- **Apparent "memory leaks"** — requests pile up in memory waiting to be processed, inflating RSS.

You can detect this by monitoring the event loop delay (`monitorEventLoopDelay()` in `perf_hooks`) or using a simple `setInterval` that measures drift.

**Options for handling CPU-intensive tasks:**

1. **Worker threads** (`worker_threads` module) — spawn a worker to run CPU work off the main thread. Good for tasks like image processing or JSON parsing of large payloads. Communication via message passing (structured clone) or `SharedArrayBuffer`. Best when the CPU work is intermittent and you want to keep everything in one process.

2. **Child processes** (`child_process.fork()`) — spawn a separate Node.js process. More isolation (separate memory space, crash doesn't affect parent), but higher overhead for communication (IPC serialization). Good for untrusted or crash-prone workloads.

3. **Separate service** — offload CPU-bound work to a dedicated service (possibly written in a language better suited for it, like Go or Rust). Communicate via HTTP, gRPC, or a message queue. Best for heavy, sustained CPU workloads (video transcoding, ML inference).

4. **Chunking/yielding** — break CPU work into small chunks and yield back to the event loop between chunks using `setImmediate()`. Works for moderate workloads but adds complexity and latency.

```typescript
// Detecting event loop blockage
import { monitorEventLoopDelay } from 'perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
  console.log(`Event loop p99: ${h.percentile(99) / 1e6}ms`);
  h.reset();
}, 5000);
```

**Rule of thumb:** If a synchronous operation takes >5ms, it's worth investigating. If it takes >50ms, it needs to be offloaded.

</details>

## Conceptual Depth

<details>
<summary>4. How does the Node.js event loop actually work — what happens in each phase (timers, pending callbacks, poll, check, close), how does the microtask queue (Promise callbacks, process.nextTick) interact with the phases, and why can microtask queue starvation starve I/O even when the event loop isn't blocked by CPU work?</summary>

**The event loop phases (in order):**

1. **Timers** — executes callbacks scheduled by `setTimeout()` and `setInterval()` whose threshold has elapsed. Note: timers are not guaranteed to fire at the exact time — they fire as soon as possible after the threshold, once this phase is reached.

2. **Pending callbacks** — executes I/O callbacks deferred from the previous loop iteration (e.g., certain TCP errors like `ECONNREFUSED`).

3. **Idle/prepare** — internal housekeeping only. Not relevant to application code.

4. **Poll** — the most important phase. Retrieves new I/O events and executes their callbacks (file reads, network data, etc.). If there are no timers scheduled, the loop will block here waiting for I/O. If there are timers, it will wait only until the nearest timer threshold.

5. **Check** — executes `setImmediate()` callbacks. Always runs after the poll phase, which is why `setImmediate()` fires before `setTimeout(fn, 0)` when called from within an I/O callback.

6. **Close callbacks** — executes close event callbacks (e.g., `socket.on('close', ...)`).

**Microtask queue interaction:**

Between every phase transition (and after every individual callback within a phase), Node.js drains the microtask queue. The microtask queue has two sub-queues:

1. **`process.nextTick` queue** — drained first, always before Promise microtasks.
2. **Promise microtask queue** (`.then()`, `.catch()`, `.finally()` callbacks) — drained second.

Both are drained completely before moving on. This means: finish current callback -> drain all nextTick callbacks -> drain all Promise callbacks -> next callback (or next phase).

**Why microtask starvation starves I/O:**

```typescript
// This starves the event loop WITHOUT CPU-bound work
function recursiveNextTick() {
  process.nextTick(() => {
    // some lightweight async work
    recursiveNextTick(); // schedule another microtask
  });
}
recursiveNextTick();

// This I/O callback NEVER fires — the event loop never
// leaves the microtask queue to reach the poll phase
const server = net.createServer((socket) => {
  socket.end('hello'); // never reached
});
```

The key insight: microtasks don't run *inside* a phase — they run *between* phases and between callbacks. If you keep adding microtasks from within microtasks, the queue is never empty, so the event loop never advances to the next phase. I/O completion events sit in the kernel queue, never picked up.

This is different from CPU blocking. CPU blocking means the thread is busy computing. Microtask starvation means the thread is running callbacks (each one is fast), but it's stuck draining an infinitely refilling queue. The event loop is "running" but never progressing.

**Practical implication:** Be careful with recursive `process.nextTick()` or Promise chains that spawn new microtasks in a loop. Use `setImmediate()` instead to defer work to the check phase, which allows I/O in the poll phase to be processed first.

</details>

<details>
<summary>5. What are the four main concurrency hazards — race conditions, deadlocks, starvation, and livelocks — how does each one manifest differently, and why are some of these still possible in single-threaded Node.js even without shared memory?</summary>

**1. Race conditions** — The outcome depends on the timing/ordering of operations. Two or more operations access shared state, and the result changes depending on which one runs first.

- *Multi-threaded*: Two threads increment a counter simultaneously, resulting in a lost update.
- *Node.js*: Two concurrent requests read a user's balance, both see $100, both deduct $80 — user ends up with $20 instead of -$60. The "shared state" is the database, not memory. Still a race condition.

**2. Deadlocks** — Two or more tasks are each waiting for the other to release a resource, so neither can proceed. Everything just... stops.

- *Multi-threaded*: Thread A holds lock X, waits for lock Y. Thread B holds lock Y, waits for lock X.
- *Node.js*: Transaction A locks row 1, then tries to lock row 2. Transaction B locks row 2, then tries to lock row 1. Both database transactions hang. Also possible with Promises: function A awaits a result from function B, which awaits a result from function A.

**3. Starvation** — A task is perpetually denied access to a resource it needs because other tasks keep getting priority.

- *Multi-threaded*: A low-priority thread never gets CPU time because high-priority threads always preempt it.
- *Node.js*: Recursive `process.nextTick()` calls prevent the event loop from reaching the poll phase, starving I/O callbacks. Or a long synchronous loop starves all other callbacks.

**4. Livelocks** — Tasks are not blocked — they're actively running — but they keep changing state in response to each other without making progress. Like two people in a hallway who keep stepping aside in the same direction.

- *Multi-threaded*: Two threads detect a conflict, both back off and retry, detect the conflict again, back off again, forever.
- *Node.js*: Two distributed services both detect a conflict on a shared resource, both release and retry with the same backoff timing, repeatedly collide. Solved with randomized jitter in retry logic.

**Why these are possible in single-threaded Node.js:**

The common misconception is "single-threaded = no concurrency problems." Single-threaded eliminates *data races* (two threads writing to the same memory location simultaneously), but it doesn't eliminate *logical* concurrency hazards. Every `await` is a point where another callback can run and modify external state (database, file system, external APIs, in-memory Maps). The single thread guarantees atomicity only within synchronous code blocks — once you `await`, your "critical section" is broken.

</details>

<details>
<summary>6. How do race conditions occur in async/await code even though Node.js is single-threaded — explain the check-then-act pattern, how concurrent API handler invocations lead to lost updates, and why the absence of threads doesn't protect you from logical races across await boundaries?</summary>

**The check-then-act pattern:**

A race condition occurs when you: (1) read some state, (2) make a decision based on that state, (3) act on the decision — but between steps 1 and 3, something else changes the state. In Node.js, every `await` is where "something else" can happen.

```typescript
// Classic check-then-act race condition
async function withdrawFunds(userId: string, amount: number) {
  const user = await db.query('SELECT balance FROM users WHERE id = $1', [userId]);
  //                          ^ await #1: other handlers can run here

  if (user.balance >= amount) {
    // Between the check above and the update below,
    // another request for the same user can read the SAME old balance
    await db.query('UPDATE users SET balance = balance - $1 WHERE id = $2', [amount, userId]);
    //              ^ await #2
  }
}
```

**How interleaving causes a lost update:**

Imagine two requests arrive nearly simultaneously for the same user (balance = $100), both withdrawing $80:

```
Time  | Request A                        | Request B
------|----------------------------------|----------------------------------
  1   | SELECT balance → $100            |
  2   |                                  | SELECT balance → $100
  3   | $100 >= $80? Yes                 |
  4   |                                  | $100 >= $80? Yes
  5   | UPDATE balance = 100-80 = $20    |
  6   |                                  | UPDATE balance = 100-80 = $20
```

Both requests saw $100, both approved the withdrawal, both deducted $80. The user withdrew $160 from a $100 balance. The second update overwrites the first — a **lost update**.

This happens because each `await` yields control to the event loop. Between `await` points, other callbacks (including other request handlers) execute. Within a single synchronous block, Node.js is atomic. But async functions are *not* atomic — they're sequences of synchronous blocks separated by `await` points.

**Why single-threaded doesn't protect you:**

Single-threaded eliminates **data races** (concurrent unsynchronized access to shared memory). But the "shared state" in a web application is the database, not memory. Two request handlers don't share JS variables, but they share the database. Every `await` is a context switch point, and any other handler can run and modify the database between your awaits.

Think of it this way: an async function with N awaits is not one operation — it's N+1 separate synchronous operations, and anything can happen between them.

**Fixes:**
- **Database-level**: Use `SELECT ... FOR UPDATE` (row-level lock) or optimistic concurrency with version columns.
- **Application-level**: Use an async mutex keyed by the entity being modified (covered in question 8 and 16).
- **Atomic operations**: Combine check and act into a single SQL statement: `UPDATE users SET balance = balance - $1 WHERE id = $2 AND balance >= $1`.

</details>

<details>
<summary>7. What are the four necessary conditions for a deadlock (Coffman conditions), and how do deadlocks manifest in different contexts — database transactions acquiring locks in different orders, distributed systems with multiple lock acquisitions, and even Promise chains that wait on each other — what strategies prevent each?</summary>

**The four Coffman conditions** (all four must hold simultaneously for a deadlock):

1. **Mutual exclusion** — at least one resource is held in a non-shareable mode (only one task can use it at a time).
2. **Hold and wait** — a task holding at least one resource is waiting to acquire additional resources held by other tasks.
3. **No preemption** — resources cannot be forcibly taken from a task; they must be released voluntarily.
4. **Circular wait** — a circular chain of tasks exists where each is waiting for a resource held by the next in the chain.

Breaking *any one* of these conditions prevents deadlock.

**Database transaction deadlocks:**

```
Transaction A:                        Transaction B:
  BEGIN                                 BEGIN
  UPDATE accounts SET ... WHERE id=1    UPDATE accounts SET ... WHERE id=2
  -- holds lock on row 1                -- holds lock on row 2
  UPDATE accounts SET ... WHERE id=2    UPDATE accounts SET ... WHERE id=1
  -- waits for row 2 (held by B)        -- waits for row 1 (held by A)
  -- DEADLOCK                           -- DEADLOCK
```

**Prevention:** Always acquire locks in a consistent order. If you need to lock rows 1 and 2, always lock the lower ID first. Most databases detect deadlocks and abort one transaction — the application should catch the error and retry.

```typescript
// Fix: always lock in consistent order
async function transfer(fromId: string, toId: string, amount: number) {
  const [firstId, secondId] = fromId < toId ? [fromId, toId] : [toId, fromId];

  await db.query('BEGIN');
  await db.query('SELECT * FROM accounts WHERE id = $1 FOR UPDATE', [firstId]);
  await db.query('SELECT * FROM accounts WHERE id = $1 FOR UPDATE', [secondId]);
  // Now safely perform the transfer
  await db.query('COMMIT');
}
```

**Distributed system deadlocks:**

Service A holds a lock on resource X (via Redis/ZooKeeper) and needs resource Y. Service B holds Y and needs X. Without a global lock ordering convention, deadlock occurs. Worse: distributed locks can be held by processes that crash, making deadlocks permanent.

**Prevention:**
- Consistent lock ordering across all services.
- Lock timeouts (TTL) — break the "no preemption" condition. If a lock isn't released within N seconds, it expires.
- Try-lock with backoff — attempt to acquire all needed locks, and if any fails, release all and retry with jitter.

**Promise chain deadlocks:**

```typescript
// Deadlock: A waits for B, B waits for A
const resultA = new Promise((resolve) => {
  // Waits for resultB before resolving
  resultB.then((b) => resolve(b + 1));
});

const resultB = new Promise((resolve) => {
  // Waits for resultA before resolving
  resultA.then((a) => resolve(a + 1));
});
// Neither ever resolves
```

This also happens with resource pools: if a function acquires a connection from a pool, then calls another function that also needs a connection from the same pool, and the pool is exhausted — deadlock.

**Prevention:** Avoid circular dependencies between Promises. For connection pools, pass the connection/transaction object through the call chain rather than acquiring a new one.

**General deadlock prevention strategies (mapped to Coffman conditions):**

| Condition to break | Strategy |
|---|---|
| Mutual exclusion | Use shared/read locks where possible; not always avoidable |
| Hold and wait | Acquire all resources atomically (all-or-nothing) |
| No preemption | Lock timeouts, TTLs, forced release |
| Circular wait | Consistent global lock ordering |

</details>

<details>
<summary>8. What are mutexes, semaphores, and locks in multi-threaded systems, why do they exist, and what are the Node.js single-threaded equivalents — how do async mutexes and queue-per-key patterns solve the same coordination problems without OS-level threading primitives?</summary>

**In multi-threaded systems:**

- **Mutex (mutual exclusion)** — a lock that allows exactly one thread to enter a critical section at a time. Thread A locks the mutex, does its work, unlocks. Thread B trying to lock while A holds it will block (sleep) until A releases. Binary: locked or unlocked.

- **Semaphore** — a generalization of a mutex that allows up to N concurrent accessors. A semaphore with count 5 lets 5 threads in simultaneously. Each `acquire()` decrements the count, each `release()` increments it. When count reaches 0, further acquires block. Use case: connection pools, rate limiting.

- **Lock** — general term for any synchronization mechanism. Can be read-write locks (multiple readers OR one writer), spin locks (busy-wait instead of sleep), etc.

These exist because multiple threads share memory, and without coordination, concurrent reads/writes produce corrupted data.

**Node.js equivalents:**

Node.js doesn't need OS-level mutexes for JavaScript code (single-threaded), but it *does* need logical coordination for async operations that span multiple `await` points and access shared external state (databases, files, caches).

**Async mutex (simplified — see Q16 for the complete implementation with `withLock` and cleanup):**

```typescript
class AsyncMutex {
  private queue: (() => void)[] = [];
  private locked = false;

  async acquire(): Promise<void> {
    if (!this.locked) {
      this.locked = true;
      return;
    }
    return new Promise<void>((resolve) => {
      this.queue.push(resolve);
    });
  }

  release(): void {
    const next = this.queue.shift();
    if (next) {
      next(); // hand the lock to the next waiter
    } else {
      this.locked = false;
    }
  }
}
```

Usage pattern: `await mutex.acquire(); try { ... } finally { mutex.release(); }`. This ensures only one async operation runs the critical section at a time — others queue up.

**Queue-per-key pattern:**

A single global mutex serializes *everything*, which is too aggressive. Usually you only need to serialize operations on the *same entity* (same user, same order). A queue-per-key gives you fine-grained locking — a separate mutex per key. Operations on user A are serialized, operations on user B are serialized, but A and B run concurrently. This is the application-level equivalent of database row-level locking. See Q16 for the complete `KeyedMutex` implementation.

**Why not just use database locks?** Sometimes you should. But application-level mutexes are useful when: (1) the critical section involves multiple external calls (DB + cache + API), (2) you want to prevent the work from even starting rather than blocking at the DB, or (3) you need to coordinate access to non-database resources.

**Semaphore equivalent in Node.js:** A concurrency limiter (covered in question 13/18) — allows up to N async operations to proceed simultaneously. Libraries like `p-limit` implement this pattern.

</details>

<details>
<summary>9. How did JavaScript's async patterns evolve from callbacks to Promises to async/await -- what specific problem did each generation solve that the previous couldn't, and why wasn't each new pattern a strict upgrade (what did you lose or trade away with each transition)?</summary>

**Generation 1: Callbacks**

The original async pattern. Pass a function to be called when the operation completes. Enabled non-blocking I/O in a single-threaded environment.

**Problems it created:**
- **Callback hell** — nested callbacks become deeply indented and hard to follow.
- **Error handling fragmentation** — every callback needs its own `if (err)` check. Forgetting one silently swallows errors.

**Generation 2: Promises**

A Promise represents a future value. It's an object you can chain, combine, and pass around.

```typescript
readFile('/path')
  .then((data) => db.query(data.toString()))
  .then((result) => process(result))
  .catch((err) => handleError(err)); // single error handler
```

**Problems it solved:**
- **Flat chaining** instead of nesting.
- **Unified error handling** — `.catch()` at the end handles errors from any step in the chain.
- **Composition** — `Promise.all()`, `Promise.race()`, etc. for combining async operations.
- **Guaranteed single resolution** — a Promise resolves or rejects exactly once.

**What you lost/traded:**
- **Error visibility** — early Promise implementations lost stack context, and forgetting `.catch()` silently swallowed errors (pre-Node 15 this was a warning; Node 15+ crashes the process by default). V8 has improved stack traces with `--async-stack-traces`, but gaps remain.
- **Eager execution** — a Promise starts executing the moment it's created. You can't defer execution like you can with a callback-accepting function. This makes cancellation hard.
- **Verbosity for sequential operations** — chaining `.then()` for sequential steps is more verbose than it needs to be.

**Generation 3: async/await**

Syntactic sugar over Promises. Makes async code look synchronous.

```typescript
try {
  const data = await readFile('/path');
  const result = await db.query(data.toString());
  return process(result);
} catch (err) {
  handleError(err);
}
```

**Problems it solved:**
- **Readability** — sequential async code reads top-to-bottom, like synchronous code.
- **Error handling** — standard try/catch works, consistent with synchronous patterns.
- **Debugging** — step-through debugging works naturally; stack traces are better.
- **Control flow** — loops, conditionals, and early returns work naturally with `await`.

**What you lost/traded:**
- **Accidental serialization** — developers write `await a(); await b();` when `a` and `b` are independent and should run concurrently (`Promise.all([a(), b()])`). The synchronous-looking syntax encourages sequential thinking.
- **try/catch verbosity** — wrapping every await in try/catch (or every function) can be verbose. Some teams adopt a `[err, result]` tuple pattern instead.
- **Hidden Promises** — developers forget that `async` functions always return Promises. Forgetting to `await` an async call is a silent bug — the function fires and its errors/results are ignored.
- **No top-level await in older module systems** — required workarounds (now solved with ES modules).

**Key insight:** Each generation didn't replace the previous — it built on it. async/await is Promises under the hood, which are callbacks under the hood. Understanding all three matters because libraries and older codebases mix them, and debugging requires knowing what's actually happening.

</details>

<details>
<summary>10. When should you use Worker threads in Node.js instead of other concurrency options — how does main thread/worker communication work, what is the difference between transferring data via structured clone (message passing) vs SharedArrayBuffer, and what are the tradeoffs and pitfalls of each approach?</summary>

**When to use Worker threads:**

Use worker threads when you have CPU-bound work that would block the event loop but doesn't justify a separate service. Common examples: image/video processing, heavy JSON parsing, cryptographic operations, data compression, complex calculations.

**Don't use worker threads for:**
- I/O-bound work — the event loop handles this efficiently already.
- Simple tasks — the overhead of spawning a worker and serializing data isn't worth it for work that takes <10ms.
- Work that needs its own process isolation — use `child_process.fork()` instead (separate memory space, crash isolation).

**How communication works:**

Workers and the main thread communicate via message passing using `postMessage()` and `on('message')`:

```typescript
// main.ts
import { Worker } from 'worker_threads';

const worker = new Worker('./heavy-task.js');
worker.postMessage({ data: largeArray }); // sends data to worker

worker.on('message', (result) => {
  console.log('Result:', result); // receives result from worker
});

worker.on('error', (err) => console.error('Worker error:', err));
```

```typescript
// heavy-task.ts
import { parentPort } from 'worker_threads';

parentPort?.on('message', (msg) => {
  const result = expensiveComputation(msg.data);
  parentPort?.postMessage(result); // sends result back
});
```

**Structured clone (message passing) vs SharedArrayBuffer:**

| Aspect | Message passing (structured clone) | SharedArrayBuffer |
|---|---|---|
| **How it works** | Data is serialized (deep-copied) when sent, deserialized on receipt. Each side has its own copy. | A shared block of raw memory that both threads can read/write directly. No copying. |
| **Data types** | Most JS types: objects, arrays, Maps, Sets, Dates, RegExps, ArrayBuffers (transferable). Cannot clone functions, DOM nodes, or Symbols. | Raw binary data only. You work with typed arrays (`Int32Array`, `Uint8Array`, etc.) over the shared buffer. |
| **Performance** | Copying overhead scales with data size. Transferring (moving) an ArrayBuffer is O(1) but invalidates the sender's reference. | Zero-copy. Reads and writes are immediate. But requires `Atomics` for synchronization. |
| **Safety** | Safe by default — no shared state, no data races. Each side has its own copy. | Unsafe — true shared memory means true data races. You need `Atomics.wait()`, `Atomics.notify()`, `Atomics.compareExchange()` for safe coordination. |
| **Use case** | Most worker thread scenarios. Send a task, get a result back. | High-performance scenarios where copying is too expensive: large matrices, real-time audio processing, shared counters. |

**Transferable objects** are a middle ground: you can *transfer* ownership of an `ArrayBuffer` to the worker (zero-copy), but the sender loses access to it. Good for passing large buffers one-way.

```typescript
// Transfer instead of copy — zero-copy, but sender loses the buffer
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
worker.postMessage(buffer, [buffer]); // second arg = transfer list
// buffer.byteLength === 0 after transfer — sender can't use it
```

**Pitfalls:**

- **Structured clone cost** — sending a 100MB object copies 100MB. For large data, use transfer or SharedArrayBuffer.
- **SharedArrayBuffer complexity** — you're doing manual memory management and synchronization in JavaScript, which defeats much of why you chose JS in the first place.
- **Worker startup overhead** — creating a worker thread isn't free (~30-50ms). For recurring tasks, maintain a worker pool rather than spawning and destroying workers per-task.
- **No shared JS objects** — you cannot share regular JS objects between threads. SharedArrayBuffer is raw bytes. If you need to share structured data, you must manually serialize/deserialize into the buffer.
- **Memory** — each worker has its own V8 isolate (~10MB overhead). Don't spawn hundreds.

</details>

<details>
<summary>11. What is the libuv thread pool, which Node.js operations use it (and which don't), what is the default pool size, and what happens when the pool is saturated — why does DNS resolution use the thread pool while TCP sockets don't, and how does saturation cause seemingly unrelated operations to slow down?</summary>

**What is the libuv thread pool?**

libuv is Node.js's underlying C library that handles async I/O. Most I/O uses OS-level async primitives (epoll on Linux, kqueue on macOS, IOCP on Windows) — these are truly non-blocking and don't need threads. But some operations don't have async OS APIs, so libuv maintains a thread pool to run them in the background without blocking the event loop.

**Operations that use the thread pool:**

- **DNS lookups** (`dns.lookup()` — the one that uses the OS resolver, not `dns.resolve()` which uses c-ares)
- **File system operations** (`fs.*`) — most OS file system APIs are blocking, so libuv offloads them
- **Some crypto operations** (`crypto.pbkdf2()`, `crypto.randomBytes()`, `crypto.scrypt()`)
- **Zlib compression/decompression** (`zlib.deflate()`, `zlib.gzip()`, etc.)

**Operations that DON'T use the thread pool:**

- **TCP/UDP sockets** — the OS provides async APIs (epoll/kqueue) for network I/O
- **DNS resolution via c-ares** (`dns.resolve()`, `dns.resolve4()`) — uses its own async implementation
- **Timers** — managed by libuv's event loop directly
- **Child processes** — use OS-level process management

**Why DNS lookup uses the thread pool but TCP doesn't:**

`dns.lookup()` calls `getaddrinfo()`, which is a POSIX blocking call — it reads `/etc/hosts`, checks nsswitch.conf, potentially calls NSS plugins. There's no async version of this in most operating systems. TCP sockets, on the other hand, use `epoll_ctl`/`kqueue` — the OS notifies Node when data arrives without blocking.

**Default pool size and saturation:**

The default thread pool size is **4 threads** (configurable via `UV_THREADPOOL_SIZE`, max 1024). With only 4 threads:

```
// Scenario: 4 slow file reads saturate the pool
fs.readFile('large1.dat', cb);  // thread 1
fs.readFile('large2.dat', cb);  // thread 2
fs.readFile('large3.dat', cb);  // thread 3
fs.readFile('large4.dat', cb);  // thread 4
// Pool is now full

dns.lookup('api.example.com', cb);  // QUEUED — waits for a thread
// Your HTTP request can't even resolve DNS until a file read finishes
```

This is why saturation causes "seemingly unrelated" slowdowns. DNS resolution, file reads, and crypto all share the same pool. A burst of file system operations can make DNS lookups (and therefore all outgoing HTTP requests that use `dns.lookup()`) queue up.

**Diagnosing and fixing:**

- Increase `UV_THREADPOOL_SIZE` to 64 or 128 if your workload is thread-pool-heavy (set it before any `require()` calls or via environment variable).
- Use `dns.resolve()` instead of `dns.lookup()` where possible — it bypasses the thread pool.
- Monitor thread pool utilization (not directly exposed, but high event loop delay combined with heavy fs/dns usage is a strong signal).
- For HTTP clients, some libraries let you configure the DNS resolver to avoid the thread pool bottleneck.

</details>

<details>
<summary>12. What is backpressure, why does the fast producer / slow consumer problem cause memory exhaustion, how do Node.js streams handle backpressure automatically, and what happens when you ignore backpressure signals — why is this one of the most common causes of Node.js memory issues in production?</summary>

**What is backpressure?**

Backpressure is the mechanism by which a slow consumer signals a fast producer to slow down. It's borrowed from fluid dynamics — when a pipe can't drain as fast as it fills, pressure builds up.

**The fast producer / slow consumer problem:**

Imagine reading a file at 500MB/s from an SSD and writing it to a network socket at 10MB/s. Without backpressure, the reader keeps producing data that the writer can't consume fast enough. That data has to go somewhere — it accumulates in memory buffers. For a 10GB file, you could exhaust your entire heap.

**How Node.js streams handle backpressure automatically:**

Node.js streams have a built-in `highWaterMark` (default 16 objects for object mode streams, 64 KiB for byte streams). The mechanism:

1. When you `readable.pipe(writable)`, the pipe reads data from the source and writes it to the destination.
2. `writable.write(chunk)` returns `false` when the internal buffer exceeds `highWaterMark` — this is the backpressure signal.
3. When `pipe()` sees `write()` return `false`, it **pauses** the readable stream (stops reading).
4. When the writable drains its buffer, it emits a `'drain'` event.
5. `pipe()` listens for `'drain'` and **resumes** the readable stream.

This keeps memory usage bounded — roughly `highWaterMark` bytes in each stream's buffer.

```typescript
import { createReadStream, createWriteStream } from 'fs';

// pipe() handles backpressure automatically
createReadStream('huge-file.dat')
  .pipe(createWriteStream('output.dat'));
// Memory stays bounded regardless of file size
```

**What happens when you ignore backpressure:**

```typescript
import { createReadStream, createWriteStream } from 'fs';

const readable = createReadStream('huge-file.dat');
const writable = createWriteStream('output.dat');

// BAD: ignoring the return value of write()
readable.on('data', (chunk) => {
  writable.write(chunk); // returns false when buffer is full — we ignore it
  // Data keeps piling up in writable's internal buffer
  // Memory grows unbounded until OOM crash
});
```

The correct manual approach (if you can't use `pipe()`):

```typescript
readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    readable.pause(); // stop reading until writable catches up
  }
});

writable.on('drain', () => {
  readable.resume(); // writable caught up, resume reading
});
```

**Why this is one of the most common Node.js memory issues:**

- Developers use `.on('data')` without checking `write()` return values.
- HTTP response streaming without backpressure — reading from a database cursor faster than the client can receive.
- Logging pipelines where the log transport is slower than the event rate.
- ETL jobs that read from a fast source (S3, database) and write to a slow destination (API, another database).

These issues don't appear in development (small files, fast local I/O) but explode in production with large payloads or slow networks. The memory grows gradually, making it look like a "memory leak" when it's really unbounded buffering.

**Rule of thumb:** Always use `pipe()` or `pipeline()` (from `stream/promises`) when connecting streams. If you must handle data events manually, always respect the `write()` return value.

</details>

<details>
<summary>13. Why is unbounded concurrency dangerous when running many async operations simultaneously — what happens when you fire off thousands of concurrent HTTP requests or database queries without a concurrency limit, how does this differ from backpressure in streams, and what patterns exist to control the level of concurrency?</summary>

**The danger of unbounded concurrency:**

```typescript
// Looks innocent — processes 50,000 users
const userIds = await getAllUserIds(); // 50,000 IDs
await Promise.all(userIds.map((id) => fetchAndUpdateUser(id)));
// This fires 50,000 concurrent HTTP requests / DB queries simultaneously
```

**What happens:**

- **Connection pool exhaustion** — your database connection pool (typically 10-50 connections) is instantly saturated. The remaining 49,950 operations queue internally, consuming memory while waiting.
- **File descriptor exhaustion** — each open socket needs a file descriptor. Linux defaults to ~1024 per process. 50,000 connections exceeds this.
- **Memory spike** — each in-flight request holds its request/response data in memory. 50,000 concurrent requests can easily consume gigabytes.
- **Downstream service overload** — you effectively DDoS your own database or external API. Connection timeouts, circuit breakers tripping, cascading failures.
- **TCP port exhaustion** — outbound connections use ephemeral ports (typically ~28,000 available). 50,000 simultaneous connections exhaust them.
- **Timeouts and retries** — slow responses trigger timeouts, which trigger retries, which add more load — a death spiral.

**How this differs from backpressure in streams:**

Backpressure (covered in question 12) is about a single data pipeline: one producer, one consumer, data flowing through buffers. The stream API has built-in signals (`write()` returning `false`, `drain` event) to regulate flow.

Unbounded concurrency is about *fan-out*: launching many independent operations simultaneously. There's no built-in signal from `Promise.all()` that says "slow down" — it just fires everything at once. You need to add concurrency control yourself.

| | Backpressure | Unbounded concurrency |
|---|---|---|
| **Pattern** | Pipeline (A -> B) | Fan-out (A -> [B1, B2, ...Bn]) |
| **Built-in protection** | Stream API handles it | None — you must add it |
| **Failure mode** | Memory growth from buffering | Resource exhaustion (connections, memory, ports) |

**Patterns to control concurrency:**

**1. Concurrency limiter (semaphore pattern):**

```typescript
// Using p-limit (most common in production)
import pLimit from 'p-limit';

const limit = pLimit(10); // max 10 concurrent operations

const results = await Promise.all(
  userIds.map((id) => limit(() => fetchAndUpdateUser(id)))
);
// Only 10 run at a time; others wait their turn
```

**2. Chunked processing:**

```typescript
// Process in batches of 10
for (let i = 0; i < userIds.length; i += 10) {
  const batch = userIds.slice(i, i + 10);
  await Promise.all(batch.map((id) => fetchAndUpdateUser(id)));
}
// Downside: waits for slowest item in each batch before starting next
```

The limiter approach (option 1) is superior because it keeps exactly N operations in flight at all times, while chunking leaves slots unused while waiting for the slowest item in each batch.

**3. Async queue (for more control):**

For complex scenarios (priority, rate limiting, retries), use a queue library like `p-queue` or `bull`. These provide features beyond simple concurrency limiting.

See question 18 for a full implementation of a concurrency limiter from scratch.

</details>

<details>
<summary>14. What are the four Promise combinators (all, allSettled, race, any), how does each one behave when some promises resolve and others reject, and when does each one fit — why does the difference between all and allSettled matter for production error handling, and when would you choose race over any?</summary>

| Combinator | Resolves when | Rejects when | Short-circuits? |
|---|---|---|---|
| **`Promise.all`** | All promises resolve | Any one rejects | Yes — rejects immediately on first rejection |
| **`Promise.allSettled`** | All promises settle (resolve or reject) | Never rejects | No — always waits for everything |
| **`Promise.race`** | First promise settles (resolve or reject) | First promise settles with rejection | Yes — settles with the first result, win or lose |
| **`Promise.any`** | First promise resolves | All promises reject | Yes — resolves with the first success |

**`Promise.all` — all must succeed:**

Use when every result is needed and a single failure means the whole operation is invalid. Example: loading all required data for a page.

If one rejects, the other promises **keep running** (they're already started) — their results are just ignored. This is a common misconception: `all` doesn't cancel anything.

**`Promise.allSettled` — get every result regardless of failures:**

Use when you want to attempt everything and handle successes/failures individually. Example: sending notifications to multiple channels — you want to know which succeeded and which failed, not abort on the first failure.

```typescript
const results = await Promise.allSettled([
  sendEmail(user),
  sendSMS(user),
  sendPush(user),
]);

// Each result is { status: 'fulfilled', value: ... }
//             or { status: 'rejected', reason: ... }
const failures = results.filter((r) => r.status === 'rejected');
if (failures.length > 0) {
  logger.warn('Some notifications failed', { failures });
}
```

**Why the `all` vs `allSettled` difference matters in production:**

With `Promise.all`, if you're fetching data from 5 microservices and one fails, you lose the results from the other 4 — even if they succeeded. You might want partial results. `allSettled` gives you everything that worked plus explicit failure information for what didn't.

**`Promise.race` — first to settle wins (success OR failure):**

Use for timeouts or "first response wins" patterns. The key behavior: it settles with whatever the first promise does, even if that's a rejection.

```typescript
// Timeout pattern
const result = await Promise.race([
  fetchData(),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), 5000)
  ),
]);
```

**`Promise.any` — first success wins, ignoring failures:**

Use when you have multiple sources and want the first successful one. Failures are ignored unless ALL fail, in which case it rejects with an `AggregateError` containing all rejection reasons.

```typescript
// Try multiple CDN endpoints, use whichever responds first
const data = await Promise.any([
  fetch('https://cdn1.example.com/data'),
  fetch('https://cdn2.example.com/data'),
  fetch('https://cdn3.example.com/data'),
]);
```

**When to choose `race` over `any`:**

- Use `race` when you care about the first *outcome* regardless of success/failure — timeouts, cancellation patterns.
- Use `any` when you want the first *success* and want to ignore failures — redundant sources, fallback patterns.

See question 17 for additional real-world scenarios.

</details>

## Practical — Implementation & Patterns

<details>
<summary>15. Given two concurrent Express request handlers that both read a user's balance, check if it's sufficient, and then deduct an amount — show the TypeScript code that demonstrates the race condition (check-then-act), explain exactly how two requests interleave across await boundaries to produce a lost update, and then show how to fix it using an async mutex or database-level locking</summary>

**The buggy handler (race condition):**

```typescript
import express from 'express';
import { Pool } from 'pg';

const pool = new Pool();
const app = express();

app.post('/withdraw', async (req, res) => {
  const { userId, amount } = req.body;

  // Step 1: READ — check current balance
  const { rows } = await pool.query(
    'SELECT balance FROM accounts WHERE id = $1',
    [userId]
  );
  const balance = rows[0].balance;

  // Step 2: DECIDE — is the balance sufficient?
  if (balance < amount) {
    return res.status(400).json({ error: 'Insufficient funds' });
  }

  // Step 3: ACT — deduct the amount
  await pool.query(
    'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
    [amount, userId]
  );

  res.json({ newBalance: balance - amount });
});
```

**How two requests interleave:**

The same check-then-act interleaving from question 6 applies here — two requests read the same stale balance across `await` boundaries, both approve the withdrawal, and the second UPDATE overwrites the first. The result: both withdrawals succeed, and the user withdraws $160 from a $100 account.

**Fix 1: Database-level locking (SELECT FOR UPDATE)**

```typescript
app.post('/withdraw', async (req, res) => {
  const { userId, amount } = req.body;
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // FOR UPDATE acquires a row-level lock — other transactions
    // trying to SELECT FOR UPDATE the same row will WAIT here
    const { rows } = await client.query(
      'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE',
      [userId]
    );
    const balance = rows[0].balance;

    if (balance < amount) {
      await client.query('ROLLBACK');
      return res.status(400).json({ error: 'Insufficient funds' });
    }

    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, userId]
    );
    await client.query('COMMIT');

    res.json({ newBalance: balance - amount });
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
});
```

Now Request B's `SELECT FOR UPDATE` blocks until Request A commits or rolls back. B reads the updated balance ($20), sees `20 < 80`, and correctly rejects.

**Fix 2: Atomic SQL (combine check and act)**

```typescript
app.post('/withdraw', async (req, res) => {
  const { userId, amount } = req.body;

  // Single atomic statement — no gap between check and act
  const { rowCount, rows } = await pool.query(
    `UPDATE accounts
     SET balance = balance - $1
     WHERE id = $2 AND balance >= $1
     RETURNING balance`,
    [amount, userId]
  );

  if (rowCount === 0) {
    return res.status(400).json({ error: 'Insufficient funds' });
  }

  res.json({ newBalance: rows[0].balance });
});
```

This is the simplest fix when the logic can be expressed in a single SQL statement. No transaction needed, no lock management.

**Fix 3: Application-level async mutex (using the keyed mutex from question 8)**

```typescript
import { KeyedMutex } from './keyed-mutex';

const userLocks = new KeyedMutex();

app.post('/withdraw', async (req, res) => {
  const { userId, amount } = req.body;

  // Serializes all operations for the same userId
  const result = await userLocks.withLock(userId, async () => {
    const { rows } = await pool.query(
      'SELECT balance FROM accounts WHERE id = $1',
      [userId]
    );

    if (rows[0].balance < amount) {
      throw new Error('Insufficient funds');
    }

    await pool.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, userId]
    );

    return rows[0].balance - amount;
  });

  res.json({ newBalance: result });
});
```

**Which fix to use:**

- **Atomic SQL** — best when the logic fits a single statement. Simplest, most performant.
- **SELECT FOR UPDATE** — best when the critical section involves multiple reads/writes within a transaction. The database handles the coordination.
- **Application mutex** — best when the critical section spans multiple external calls (DB + cache + API) or when you want to avoid even starting the work while another operation on the same entity is in progress. Caveat: only works within a single process — doesn't help with multiple server instances (use distributed locks for that).

</details>

<details>
<summary>16. Implement an async mutex and a queue-per-key pattern in TypeScript — show how the mutex serializes access to a shared resource, show how queue-per-key ensures operations on the same entity (e.g., same user ID) run sequentially while different entities run concurrently, and explain why this is the Node.js equivalent of fine-grained locking</summary>

**Async Mutex — full implementation with `withLock` convenience:**

```typescript
class AsyncMutex {
  private _locked = false;
  private _waitQueue: Array<() => void> = [];

  get isLocked(): boolean {
    return this._locked || this._waitQueue.length > 0;
  }

  acquire(): Promise<void> {
    if (!this._locked) {
      this._locked = true;
      return Promise.resolve();
    }
    return new Promise<void>((resolve) => {
      this._waitQueue.push(resolve);
    });
  }

  release(): void {
    const next = this._waitQueue.shift();
    if (next) {
      // Hand lock directly to next waiter (stays locked)
      next();
    } else {
      this._locked = false;
    }
  }

  async withLock<T>(fn: () => Promise<T>): Promise<T> {
    await this.acquire();
    try {
      return await fn();
    } finally {
      this.release();
    }
  }
}
```

**Usage — serializing access to a shared resource:**

```typescript
const mutex = new AsyncMutex();
let sharedCounter = 0;

async function safeIncrement(): Promise<number> {
  return mutex.withLock(async () => {
    const current = sharedCounter;
    await someAsyncWork(); // other code CAN'T modify sharedCounter here
    sharedCounter = current + 1;
    return sharedCounter;
  });
}

// Even with concurrent calls, the counter increments correctly
await Promise.all([safeIncrement(), safeIncrement(), safeIncrement()]);
// sharedCounter === 3 (guaranteed)
```

**Queue-per-key — fine-grained locking by entity:**

```typescript
class KeyedMutex {
  private locks = new Map<string, AsyncMutex>();

  async withLock<T>(key: string, fn: () => Promise<T>): Promise<T> {
    // Get or create a mutex for this key
    let mutex = this.locks.get(key);
    if (!mutex) {
      mutex = new AsyncMutex();
      this.locks.set(key, mutex);
    }

    await mutex.acquire();
    try {
      return await fn();
    } finally {
      mutex.release();
      // Clean up mutex when no one else is waiting
      // Prevents unbounded Map growth
      if (!mutex.isLocked) {
        this.locks.delete(key);
      }
    }
  }
}
```

**Usage — concurrent operations on different entities, serialized on same entity:**

```typescript
const userLocks = new KeyedMutex();

async function processOrder(userId: string, orderId: string) {
  return userLocks.withLock(userId, async () => {
    const balance = await getBalance(userId);
    const order = await getOrder(orderId);
    if (balance >= order.total) {
      await deductBalance(userId, order.total);
      await markOrderPaid(orderId);
    }
  });
}

// These run CONCURRENTLY (different users)
processOrder('user-A', 'order-1');
processOrder('user-B', 'order-2');

// These run SEQUENTIALLY (same user)
processOrder('user-A', 'order-3');
processOrder('user-A', 'order-4');
```

**Why this is fine-grained locking:**

In multi-threaded systems, a global mutex (like Java's `synchronized` block) serializes ALL access. A `ReentrantLock` per entity serializes only operations on the same entity. The queue-per-key pattern is exactly this: a mutex per key. It's the application-level equivalent of database row-level locking.

| Locking granularity | Multi-threaded equivalent | Node.js equivalent | Tradeoff |
|---|---|---|---|
| Global | Global mutex | Single `AsyncMutex` | Safe but serializes everything — poor throughput |
| Per-entity | `ConcurrentHashMap<Key, Lock>` | `KeyedMutex` | Best balance — serializes only conflicting operations |
| None | No synchronization | Raw `Promise.all` | Maximum throughput but race conditions |

**Important caveats:**

- **Single-process only.** If you have multiple server instances behind a load balancer, application-level mutexes don't help — two requests for the same user can hit different instances. Use database-level locking or distributed locks (Redis Redlock) for multi-instance coordination.
- **Memory cleanup matters.** Without the `if (!mutex.isLocked) this.locks.delete(key)` cleanup, the Map grows indefinitely as new keys are seen. In high-cardinality scenarios (millions of unique keys), this becomes a memory leak.
- **Deadlock risk.** If the function inside `withLock` tries to acquire the same key's lock again (re-entrancy), it deadlocks. Unlike Java's `ReentrantLock`, this implementation is not reentrant.

</details>

<details>
<summary>17. Write TypeScript examples that demonstrate when to use each Promise combinator (all, allSettled, race, any) — show a real scenario for each one, demonstrate how each handles partial failures differently, and show the error handling patterns that prevent silent failures in production</summary>

**`Promise.all` — loading required data for a dashboard:**

```typescript
// All three are required — if any fails, the page can't render
async function loadDashboard(userId: string) {
  try {
    const [user, orders, settings] = await Promise.all([
      fetchUser(userId),
      fetchOrders(userId),
      fetchSettings(userId),
    ]);
    return { user, orders, settings };
  } catch (err) {
    // Rejects with the FIRST error — other promises keep running
    // but their results (or errors) are silently discarded
    logger.error('Dashboard load failed', { userId, error: err });
    throw err;
  }
}

// Gotcha: if fetchOrders AND fetchSettings both fail,
// you only see the first rejection. To see all errors, use allSettled.
```

**`Promise.allSettled` — sending notifications across multiple channels:**

```typescript
async function notifyUser(userId: string, message: string) {
  const results = await Promise.allSettled([
    sendEmail(userId, message),
    sendSMS(userId, message),
    sendPushNotification(userId, message),
  ]);

  // Inspect each result individually
  const report = results.map((result, i) => {
    const channel = ['email', 'sms', 'push'][i];
    if (result.status === 'fulfilled') {
      return { channel, success: true };
    }
    // Log each failure without losing any
    logger.warn(`${channel} notification failed`, {
      userId,
      error: result.reason,
    });
    return { channel, success: false, error: result.reason.message };
  });

  // Decide: is partial success acceptable?
  const successes = report.filter((r) => r.success);
  if (successes.length === 0) {
    throw new Error('All notification channels failed');
  }

  return report;
}
```

**`Promise.race` — timeout wrapper:**

```typescript
function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
  );

  // race settles with whichever finishes first — success OR failure
  return Promise.race([promise, timeout]);
}

// Usage
try {
  const data = await withTimeout(fetchFromSlowService(), 3000);
} catch (err) {
  if (err.message.includes('Timed out')) {
    // Handle timeout specifically — maybe use cached data
    return getCachedData();
  }
  throw err; // actual service error
}

// Important: the original promise keeps running even after timeout.
// race doesn't cancel anything — it just ignores the slower promise's result.
```

**`Promise.any` — redundant service fallback:**

```typescript
async function fetchWithFallback(key: string): Promise<Data> {
  try {
    const data = await Promise.any([
      fetchFromPrimaryCache(key),   // Redis in same region
      fetchFromSecondaryCache(key), // Redis in another region
      fetchFromDatabase(key),       // PostgreSQL (slowest but authoritative)
    ]);
    // Returns the FIRST successful result — fastest wins
    return data;
  } catch (err) {
    // Only reaches here if ALL three fail
    // err is an AggregateError with all rejection reasons
    if (err instanceof AggregateError) {
      logger.error('All data sources failed', {
        errors: err.errors.map((e) => e.message),
      });
    }
    throw err;
  }
}
```

**Error handling patterns to prevent silent failures:**

```typescript
// DANGEROUS: fire-and-forget with Promise.all
Promise.all(tasks); // no await, no catch — rejections are unhandled

// DANGEROUS: allSettled without checking results
await Promise.allSettled(tasks); // results ignored — failures are silent

// SAFE: always inspect allSettled results
const results = await Promise.allSettled(tasks);
const failures = results.filter(
  (r): r is PromiseRejectedResult => r.status === 'rejected'
);
if (failures.length > 0) {
  logger.error('Partial failures', {
    count: failures.length,
    errors: failures.map((f) => f.reason),
  });
}
```

For the behavioral differences summary, see the comparison table in question 14.

</details>

<details>
<summary>18. Implement a concurrency limiter in TypeScript that processes an array of async tasks with a maximum number of concurrent operations — show the implementation, demonstrate what happens without the limiter (memory exhaustion, connection pool saturation), and explain how libraries like p-limit solve this in production</summary>

**Without a limiter (the problem):**

```typescript
// 10,000 users to process
const userIds = await getAllUserIds();

// BAD: fires ALL 10,000 at once
const results = await Promise.all(
  userIds.map((id) => processUser(id)) // 10,000 concurrent DB queries
);
// Symptoms: connection pool exhaustion, ENFILE (too many open files),
// OOM crash, downstream service overload
```

**Concurrency limiter implementation:**

```typescript
function pLimit(concurrency: number) {
  const queue: Array<() => void> = [];
  let activeCount = 0;

  function next() {
    if (queue.length > 0 && activeCount < concurrency) {
      activeCount++;
      const run = queue.shift()!;
      run();
    }
  }

  function limit<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      const run = () => {
        fn()
          .then(resolve, reject)
          .finally(() => {
            activeCount--;
            next(); // pull next task from queue when one finishes
          });
      };

      if (activeCount < concurrency) {
        activeCount++;
        run();
      } else {
        queue.push(run); // queue it for later
      }
    });
  }

  return limit;
}
```

**Usage:**

```typescript
const limit = pLimit(10); // max 10 concurrent operations

const results = await Promise.all(
  userIds.map((id) => limit(() => processUser(id)))
);
// Only 10 processUser calls run at any time
// As one finishes, the next queued task starts immediately
```

**How it works step by step:**

1. First 10 calls to `limit()` start immediately (`activeCount` goes from 0 to 10).
2. Calls 11+ are queued — `limit()` returns a Promise that doesn't resolve until the wrapped function runs.
3. When any running task finishes (`.finally()`), `activeCount` decrements and `next()` pulls the next queued task.
4. The outer `Promise.all` waits for ALL returned promises to resolve — they just don't all start at once.

**Comparison: chunking vs limiter:**

```typescript
// Chunking — suboptimal
for (let i = 0; i < items.length; i += 10) {
  await Promise.all(items.slice(i, i + 10).map(process));
  // Problem: if 9 tasks finish in 10ms and 1 takes 5s,
  // 9 slots sit idle for 5 seconds before next batch starts
}

// Limiter — optimal
const limit = pLimit(10);
await Promise.all(items.map((item) => limit(() => process(item))));
// Always keeps exactly 10 in flight — no idle slots
```

**How p-limit works in production:**

The real `p-limit` library follows the same pattern as our implementation but adds:
- Proper microtask scheduling (uses `async` functions for better stack traces)
- A `clearQueue()` method to cancel pending tasks
- An `activeCount` and `pendingCount` getter for monitoring
- TypeScript types out of the box

```typescript
import pLimit from 'p-limit';

const limit = pLimit(5);

// Process database migrations with controlled concurrency
const migrations = await getMigrations();
await Promise.all(
  migrations.map((m) =>
    limit(async () => {
      logger.info(`Running migration: ${m.name}`);
      await m.run();
    })
  )
);
```

**Choosing the right concurrency limit:**

- **Database queries**: Match your connection pool size (e.g., pool size 20 -> concurrency 15-20).
- **HTTP requests to external APIs**: Respect their rate limits. Start with 5-10, increase if they allow it.
- **File system operations**: libuv thread pool size (default 4, so >4 queues internally anyway).
- **General rule**: Start low (5-10), measure throughput, and increase until you hit diminishing returns or errors.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>19. Tell me about a time you encountered a concurrency bug in production — a race condition, deadlock, or data inconsistency caused by concurrent operations — how did you discover it, how did you reproduce and diagnose the root cause, and what did you change to prevent it from happening again?</summary>

**What the interviewer is looking for:**

- Ability to recognize concurrency bugs (they're subtle and often intermittent)
- Systematic debugging approach for non-deterministic issues
- Understanding of root cause (not just "it was a race condition" but HOW the interleaving happened)
- A concrete fix that addresses the root cause, not just the symptom

**Suggested structure (STAR+):**

1. **Situation** — What system, what was the expected behavior?
2. **Symptom** — How was the bug discovered? (Inconsistent data, customer complaint, monitoring alert)
3. **Investigation** — How did you narrow it down to a concurrency issue? What made it hard to reproduce?
4. **Root cause** — The specific interleaving that caused the bug. Be precise about which operations raced.
5. **Fix** — What you changed and why that specific approach.
6. **Prevention** — What you did to catch similar issues earlier (tests, patterns, monitoring).

**Example outline to personalize:**

- **Situation**: E-commerce service processing orders. Each order deducts inventory from a product.
- **Symptom**: Customers reported buying items that showed "in stock" but then got cancellation emails. Inventory counts occasionally went negative in the database.
- **Investigation**: Couldn't reproduce with single requests. Wrote a load test that sent 50 concurrent orders for the same product with 1 unit left. 3-5 orders would succeed instead of 1. Confirmed it was a check-then-act race across `await` boundaries.
- **Root cause**: Handler read inventory count, checked `count >= requestedQuantity`, then decremented — classic check-then-act with the gap at each `await`. Under concurrent load, multiple handlers read the same count before any UPDATE executed.
- **Fix**: Replaced the read-check-update pattern with an atomic SQL: `UPDATE products SET inventory = inventory - $1 WHERE id = $2 AND inventory >= $1 RETURNING inventory`. Single statement, no race window.
- **Prevention**: Added a concurrency test to CI that fires 20 parallel requests. Added a database constraint (`CHECK (inventory >= 0)`) as a safety net. Documented the pattern for the team.

**Key points to hit:**

- Show that you understand WHY single-threaded Node.js still has race conditions (across `await` boundaries).
- Emphasize that the bug was intermittent and timing-dependent — this is what makes concurrency bugs hard.
- Show multiple fix options you considered and why you chose the one you did.
- Mention prevention: constraints, tests, or patterns that make the class of bug impossible, not just unlikely.

</details>

<details>
<summary>20. Describe a time you had to handle CPU-bound work in a Node.js application — what was the workload, what symptoms told you the event loop was being blocked, and what solution did you choose (worker threads, child processes, separate service) and why?</summary>

**What the interviewer is looking for:**

- Ability to identify event loop blocking (symptoms, metrics, not just guessing)
- Understanding of the tradeoffs between worker threads, child processes, and separate services
- A reasoned decision, not just "I picked worker threads because that's what I know"
- Awareness that this is Node.js's fundamental weakness and how to architect around it

**Suggested structure:**

1. **Context** — What was the application and what CPU-bound work was needed?
2. **Symptoms** — How did you discover the event loop was blocked? What metrics or behaviors pointed to it?
3. **Options considered** — What solutions did you evaluate and what were the tradeoffs?
4. **Decision** — What you chose and why it was the right fit for the situation.
5. **Result** — Impact on latency, throughput, reliability.

**Example outline to personalize:**

- **Context**: API service that generates PDF reports from user data. Users trigger report generation via an API endpoint. The PDF library (e.g., puppeteer or pdfkit) does significant CPU work — rendering, layout calculation, image processing.
- **Symptoms**: During report generation, healthcheck endpoints started timing out. p99 latency for all endpoints spiked from 50ms to 3+ seconds. Monitoring showed event loop delay jumping to 2-4 seconds (via `monitorEventLoopDelay`). The issue was intermittent — only happened when multiple users requested reports simultaneously.
- **Options considered**:
  - **Worker threads**: Keep everything in one process, offload PDF generation to a worker. Pro: simple deployment, shared memory possible. Con: worker crash could destabilize the main process, memory overhead per worker.
  - **Child process**: Fork a dedicated process for each PDF job. Pro: crash isolation, separate memory. Con: higher IPC overhead, more resource usage per job.
  - **Separate service**: Move PDF generation to a dedicated microservice behind a queue. Pro: independent scaling, can use a different language/runtime. Con: more infrastructure complexity, async flow (user has to poll or get notified).
- **Decision**: Chose a separate service with a message queue. Reasoning: PDF generation took 5-30 seconds (too long for synchronous response anyway), we needed to scale it independently (3x more PDF workers during end-of-month), and crash isolation was important (PDF rendering with user-provided data sometimes caused OOM). The API service publishes a job to the queue, the PDF service picks it up, generates the PDF, uploads to S3, and updates the job status. The user gets a job ID immediately and polls for completion.
- **Result**: API latency returned to normal (p99 < 100ms). PDF generation scaled independently. OOM crashes in the PDF service didn't affect the main API.

**Key points to hit:**

- Show you can identify event loop blocking from metrics (event loop delay, healthcheck failures, latency spikes on unrelated endpoints).
- Demonstrate you considered multiple approaches before choosing.
- Explain why the choice was right for YOUR specific situation — there's no universally correct answer.
- Mention what you'd choose differently if the constraints were different (e.g., "if the work took <100ms, worker threads would have been simpler").

</details>

<details>
<summary>21. Tell me about a time you dealt with backpressure, memory exhaustion, or uncontrolled concurrency in a Node.js service — what caused the problem, how did you detect it, and what changes did you make to bring the system under control?</summary>

**What the interviewer is looking for:**

- Understanding of resource management in Node.js (memory, connections, file descriptors)
- Ability to diagnose production issues from symptoms and metrics, not just code review
- Knowledge of specific patterns: backpressure in streams (question 12), concurrency limiting (questions 13/18), bounded queues
- A fix that addresses the systemic issue, not just a band-aid

**Suggested structure:**

1. **Context** — What was the service doing? What scale was it operating at?
2. **Symptoms** — How did the problem surface? (OOM crashes, memory alerts, cascading timeouts, pod restarts)
3. **Detection and diagnosis** — What metrics or tooling pointed to the root cause?
4. **Root cause** — The specific unbounded resource consumption pattern.
5. **Fix** — What you changed and why.
6. **Safeguards** — What you added to prevent recurrence.

**Example outline to personalize:**

- **Context**: Data export service that reads records from a database, transforms them, and writes CSV files to S3. Triggered by user request, some exports had millions of rows.
- **Symptoms**: Kubernetes pods kept getting OOMKilled during large exports. Memory usage climbed steadily from 200MB to the 1GB limit, then the pod restarted. Small exports worked fine. Only happened for users with large datasets.
- **Detection**: Grafana memory dashboard showed a linear climb during export jobs — classic unbounded buffering pattern, not a leak (a leak would be gradual across many requests). Heap snapshot showed millions of row objects sitting in an internal buffer.
- **Root cause**: The code used `.on('data')` from the database cursor and called `s3Stream.write(row)` without checking the return value. The database delivered rows at ~50MB/s, but S3 upload bandwidth was ~5MB/s. The 45MB/s difference accumulated in the writable stream's internal buffer. For a 500MB export, this meant ~450MB of buffered data in memory.
- **Fix**: Replaced the manual `.on('data')` + `.write()` with `pipeline()` from `stream/promises`, which handles backpressure automatically. The database readable stream pauses when the S3 writable stream's buffer fills up. Memory usage for a 500MB export dropped from ~450MB peak to ~32MB (2x the `highWaterMark`).

```typescript
// Before (broken): no backpressure
dbCursor.on('data', (row) => {
  s3Upload.write(transformRow(row)); // return value ignored
});

// After (fixed): pipeline handles backpressure
import { pipeline } from 'stream/promises';
await pipeline(
  dbCursor,
  new Transform({ transform(row, enc, cb) { cb(null, transformRow(row)); } }),
  s3Upload
);
```

- **Safeguards**: Added memory usage alerting at 70% of pod limit. Added a maximum export size (1M rows) with a message to contact support for larger exports. Documented the backpressure pattern in the team's coding guidelines.

**Alternative scenario (uncontrolled concurrency):**

- Service processes webhook events. A burst of 5,000 webhooks arrives. Each handler makes 3 HTTP calls to downstream services. That's 15,000 concurrent outbound requests — connection pools exhausted, downstream services overloaded, timeouts everywhere.
- Fix: Added a concurrency limiter (`p-limit`) around the webhook processing. Max 20 concurrent handlers. The rest queue in memory. Added a dead letter queue for webhooks that couldn't be processed within the timeout window.

**Key points to hit:**

- Show you understand the difference between a memory leak and unbounded buffering (they look similar but have different causes and fixes).
- Demonstrate familiarity with Node.js-specific tooling: heap snapshots, `process.memoryUsage()`, event loop delay monitoring.
- Connect the fix to the underlying concept (backpressure, concurrency limiting) — show you understand the pattern, not just the specific fix.
- Mention the safeguards: monitoring, limits, documentation. The best fix prevents the class of problem, not just the instance.

</details>
