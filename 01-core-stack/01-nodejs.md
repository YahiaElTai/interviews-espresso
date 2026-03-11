# Node.js

> **37 questions** — 18 theory, 19 practical

- Single-threaded event loop architecture and why Node.js chose non-blocking I/O
- EventEmitter → Stream → HTTP/FS/Net inheritance chain
- Event loop phases, microtasks, process.nextTick(), and setImmediate; blocking patterns (sync APIs, JSON.stringify DoS, O(n^2) callbacks) and mitigation (setImmediate partitioning, worker offloading)
- libuv, the thread pool, and UV_THREADPOOL_SIZE tuning
- Streams: readable, writable, transform, backpressure, pipe() vs pipeline()
- Core APIs: Buffer for binary data, fs/promises for file I/O, crypto for hashing and randomness, path for cross-platform paths, util.promisify
- Worker threads vs child processes vs cluster module
- V8 garbage collection, memory management, heap snapshots, and leak detection
- CommonJS vs ES Modules tradeoffs and dual-package publishing
- Package management: package.json, lockfiles, semver, peer dependencies, npm workspaces for monorepos, registry and scoped packages
- Node.js releases: LTS cycle, version management (nvm/fnm), notable recent additions (built-in test runner, native fetch, permission model)
- AsyncLocalStorage for request-scoped context (tracing, logging) and async_hooks fundamentals
- HTTP frameworks: raw http module, middleware pattern (Express/Koa), plugin architecture (Fastify), schema-based validation, and performance tradeoffs
- Graceful shutdown and process lifecycle: SIGTERM/SIGINT handling, uncaughtException vs unhandledRejection, exit codes
- Error handling: operational vs programmer errors, error-first callbacks, Promise rejection chains, AbortController for cancellation, uncaught exception boundaries
- Node.js security: prototype pollution, ReDoS, dependency supply chain attacks, npm audit, lockfile integrity, child process exec vs execFile shell injection risks
- Production diagnostics: --inspect and Chrome DevTools, CPU profiling, flame graphs, event loop lag, EMFILE/ECONNRESET debugging, clinic.js

---

## Foundational

<details>
<summary>1. Why did Node.js choose a single-threaded, non-blocking I/O model instead of the multi-threaded approach used by servers like Apache — what problem does this architecture solve, what are the tradeoffs (CPU-bound work, error isolation), and when is this model a poor fit?</summary>

**The problem it solves:** Traditional servers like Apache spawn a thread (or process) per connection. Each thread blocks waiting for I/O (database queries, file reads, network calls), consuming memory (~1-2MB stack per thread) even while idle. At thousands of concurrent connections, you hit memory limits and context-switching overhead — the "C10K problem."

Node.js solves this by using a single thread that never blocks on I/O. Instead, it delegates I/O operations to the OS (via libuv) and registers callbacks. The event loop picks up completed I/O and runs the callbacks. One thread can juggle thousands of concurrent connections because it's never sitting idle waiting — it's always processing the next ready callback.

**The tradeoffs:**

- **CPU-bound work blocks everything.** Since there's one thread running JavaScript, a tight loop computing a hash for 500ms blocks all other requests for 500ms. Multi-threaded servers isolate this naturally — one thread's computation doesn't block another's.
- **No error isolation.** An uncaught exception in any request handler crashes the entire process (all connections), whereas in a thread-per-request model only that thread dies.
- **No automatic multi-core usage.** A single Node.js process uses one CPU core. You need the cluster module or container orchestration to use all cores.

**When it's a poor fit:**

- **CPU-intensive workloads** like video encoding, image processing, or heavy computation — unless you offload to worker threads or separate services.
- **Applications requiring strong fault isolation** between requests (e.g., multi-tenant systems where one tenant's bad input shouldn't risk others).
- **Long-running synchronous computations** that can't be easily partitioned or offloaded.

The model excels at I/O-heavy workloads: API servers, real-time apps, proxies, and anything where most time is spent waiting on network/disk rather than computing.

</details>

<details>
<summary>2. How does the EventEmitter → Stream → HTTP/FS/Net inheritance chain work in Node.js — why did the core library designers build HTTP servers, file system streams, and TCP sockets on top of EventEmitter and Stream instead of standalone APIs, and what does this inheritance give you as a developer?</summary>

**The inheritance chain:**

```
EventEmitter
  └── Stream (abstract base)
        ├── Readable
        │     ├── fs.ReadStream
        │     ├── http.IncomingMessage (request)
        │     └── net.Socket (readable side)
        ├── Writable
        │     ├── fs.WriteStream
        │     ├── http.ServerResponse
        │     └── net.Socket (writable side)
        └── Duplex (both Readable + Writable)
              ├── net.Socket
              └── Transform
                    └── zlib.Gzip, crypto.Cipher, etc.
```

**Why this design:**

EventEmitter provides the publish/subscribe pattern — `.on()`, `.emit()`, `.removeListener()`. Stream extends EventEmitter with a standardized data flow protocol (data/end/error events, backpressure via `highWaterMark` and `drain`). By building HTTP, FS, and Net on top of these abstractions, the core team got composability for free.

**What this gives you as a developer:**

- **Uniform API.** Every I/O source works the same way. You `.on('data', ...)` whether it's a file, HTTP request, or TCP socket. Learn one pattern, use it everywhere.
- **Composability via piping.** Because `http.IncomingMessage` is a Readable and `fs.WriteStream` is a Writable, you can pipe an HTTP upload directly to disk: `req.pipe(fs.createWriteStream('/tmp/upload'))`. No glue code needed.
- **Backpressure propagates automatically.** If the disk can't keep up with an incoming upload, the Writable signals the Readable to pause — all handled by the Stream base class.
- **Reusable tooling.** Any Transform stream (compression, encryption, parsing) works with any Readable/Writable pair. `req.pipe(zlib.createGunzip()).pipe(parser).pipe(output)` — each piece is interchangeable.

The alternative — standalone APIs with unique interfaces — would mean writing custom glue code for every combination of source and destination. The inheritance chain makes Node.js I/O feel like building with Lego blocks.

</details>

<details>
<summary>3. What are the event loop phases in Node.js (timers, pending callbacks, poll, check, close), how do microtasks and process.nextTick() fit into the execution order, and why does the ordering between setTimeout, setImmediate, and Promise.then() behave differently depending on context?</summary>

**The six phases (in order):**

```
   ┌───────────────────────────┐
┌─>│        timers              │  ← setTimeout, setInterval callbacks
│  └───────────┬───────────────┘
│  ┌───────────┴───────────────┐
│  │     pending callbacks      │  ← deferred OS callbacks (TCP errors, etc.)
│  └───────────┬───────────────┘
│  ┌───────────┴───────────────┐
│  │       idle, prepare        │  ← internal use only
│  └───────────┬───────────────┘
│  ┌───────────┴───────────────┐
│  │          poll              │  ← retrieve new I/O events; execute I/O callbacks
│  └───────────┬───────────────┘
│  ┌───────────┴───────────────┐
│  │          check             │  ← setImmediate callbacks
│  └───────────┬───────────────┘
│  ┌───────────┴───────────────┐
│  │      close callbacks       │  ← socket.on('close'), etc.
│  └───────────────────────────┘
```

**Where microtasks and nextTick fit:** Between every phase transition (and after every individual callback in Node.js 11+), the event loop drains two queues in this order:

1. **process.nextTick queue** — runs first, always
2. **Microtask queue** — Promise `.then()`, `queueMicrotask()`

So the actual execution is: run callback → drain nextTick queue → drain microtask queue → next callback (or next phase).

**Why setTimeout vs setImmediate ordering is context-dependent:**

In the **main module** (top-level code), `setTimeout(fn, 0)` and `setImmediate(fn)` have non-deterministic order. The reason: `setTimeout(fn, 0)` is actually `setTimeout(fn, 1)` internally. If the event loop enters the timers phase before 1ms has elapsed, the timer isn't ready yet — so setImmediate (check phase) fires first. If 1ms has passed, setTimeout fires first. It depends on system clock precision and process startup time.

Inside an **I/O callback** (e.g., inside `fs.readFile`'s callback), `setImmediate` always fires before `setTimeout`. Why: after an I/O callback completes, the event loop is in the poll phase. The next phase is check (setImmediate), which comes before the loop wraps back around to timers (setTimeout).

```typescript
import { readFile } from 'node:fs';

// Non-deterministic in main module
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// Could be either order

// Deterministic inside I/O
readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
  // Always: immediate, then timeout
});
```

**Promise.then() always runs before setTimeout/setImmediate** because microtasks are drained between every phase transition and after every callback — they cut in line ahead of any macrotask.

</details>

## Conceptual Depth

<details>
<summary>4. Why does Node.js need libuv and a thread pool if it's single-threaded — what specific operations get offloaded to the thread pool vs handled by the OS via epoll/kqueue, how does the default thread pool size of 4 become a bottleneck, and what does UV_THREADPOOL_SIZE actually control?</summary>

**Why libuv exists:** Node.js is "single-threaded" in that your JavaScript runs on one thread. But the OS-level reality is messier — not all async I/O operations can be handled by OS notification mechanisms (epoll on Linux, kqueue on macOS). libuv is the cross-platform abstraction layer that provides a unified async I/O interface, using the best OS mechanism available and falling back to a thread pool for operations that can't be done asynchronously at the OS level.

**What the OS handles natively (no thread pool):**

- **Network I/O** — TCP/UDP sockets, HTTP connections. The OS provides efficient async notification via epoll/kqueue/IOCP. libuv registers file descriptors and gets notified when data is ready.
- **Pipes and TTY** — also use OS-native async primitives.

**What gets offloaded to the thread pool:**

- **DNS lookups** — `dns.lookup()` uses `getaddrinfo()` which is a blocking C library call. (Note: `dns.resolve()` uses c-ares and does NOT use the thread pool.)
- **File system operations** — `fs.readFile()`, `fs.stat()`, etc. Most OS async file I/O is unreliable or incomplete, so libuv uses the thread pool.
- **Crypto operations** — `crypto.pbkdf2()`, `crypto.randomBytes()`, `crypto.scrypt()` — CPU-intensive work offloaded to avoid blocking the main thread.
- **Compression** — `zlib.inflate()`, `zlib.deflate()`.

**How size 4 becomes a bottleneck:** Imagine a Node.js API that on every request does a DNS lookup + reads a file + hashes a password. Each operation needs a thread pool thread. With pool size 4, only 4 of these operations can run concurrently. Request #5 waits in a queue even though the main thread is idle. Under load, you see requests stalling with no apparent CPU or I/O saturation — the thread pool is the hidden bottleneck.

**What UV_THREADPOOL_SIZE controls:** Set it as an environment variable before the process starts (it's read once at startup and can't be changed at runtime). It controls the number of threads in libuv's worker pool, applicable to all the operations listed above. Default is 4, max is 1024.

```bash
# Set before starting Node.js
UV_THREADPOOL_SIZE=16 node server.js
```

**Why not just set it to 1024:** Each thread consumes memory (~8MB stack on Linux by default) and more threads means more context switching. The right number depends on your workload — profile with actual load. A common heuristic is: number of CPU cores + number of concurrent I/O-bound thread pool operations you expect. For most apps, 8-16 is reasonable.

</details>

<details>
<summary>5. Why does Node.js have four stream types (Readable, Writable, Duplex, Transform) and what design problem does each solve — how does backpressure work internally (highWaterMark, drain event), and when should you avoid streams entirely?</summary>

**The four types and what each solves:**

| Stream Type | Problem | Examples |
|---|---|---|
| **Readable** | Consume data from a source incrementally, without loading it all into memory | `fs.createReadStream()`, `http.IncomingMessage`, `process.stdin` |
| **Writable** | Send data to a destination incrementally | `fs.createWriteStream()`, `http.ServerResponse`, `process.stdout` |
| **Duplex** | Both read and write independently (two separate channels) | `net.Socket` (send and receive on same connection), TCP sockets |
| **Transform** | Modify data as it passes through (read in, transform, write out) | `zlib.createGzip()`, `crypto.createCipheriv()`, CSV parser |

Duplex is "two independent pipes in one object." Transform is "one pipe where data gets modified in transit." The distinction matters: a TCP socket reads and writes independently (Duplex), while a gzip compressor reads uncompressed and writes compressed (Transform).

**How backpressure works:**

Each stream has a `highWaterMark` — the internal buffer size threshold (default: 16KB for binary, 16 objects for object mode). The mechanism:

1. The Writable's `.write(chunk)` method returns `true` if the internal buffer is below highWaterMark, `false` if it's full.
2. When `.write()` returns `false`, the producer should stop pushing data.
3. When the buffer drains below highWaterMark, the Writable emits a `'drain'` event, signaling the producer to resume.

When using `.pipe()` or `pipeline()`, this protocol is handled automatically — the Readable pauses when the Writable signals backpressure, and resumes on `'drain'`.

```typescript
import { createReadStream, createWriteStream } from 'node:fs';

const readable = createReadStream('large-file.csv');
const writable = createWriteStream('output.json');

// Manual backpressure handling (what pipe does for you):
readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    readable.pause(); // stop reading until writable catches up
  }
});

writable.on('drain', () => {
  readable.resume(); // writable caught up, continue reading
});
```

**When to avoid streams:**

- **Small data that fits in memory.** Reading a 1KB config file with `fs.readFile()` is simpler than creating a stream. Streams add complexity (error handling, state management) that isn't worth it for small payloads.
- **When you need the entire dataset before processing.** JSON.parse requires the complete string. Streaming a JSON file only makes sense with a streaming JSON parser (like `jsonstream`), which adds complexity.
- **Simple request/response APIs.** If your endpoint reads a small DB result and returns JSON, streams add no value. Just `res.json(data)`.
- **When debugging matters more than memory.** Streams make stack traces harder to read and error handling more complex. For non-performance-critical code, the simplicity of `await fs.readFile()` wins.

**Rule of thumb:** Use streams when data size is unknown or large (file uploads, log processing, ETL pipelines). Use simple read-all APIs when data is small and bounded.

</details>

<details>
<summary>6. Why does Node.js provide three different concurrency mechanisms — worker_threads, child_process, and the cluster module — when would you reach for each one, what are the tradeoffs in memory overhead, data sharing, and communication cost, and what goes wrong when you pick the wrong one?</summary>

Each solves a different concurrency problem. They exist separately because the tradeoffs are fundamentally different.

| | **worker_threads** | **child_process** | **cluster** |
|---|---|---|---|
| **What it is** | Threads within the same process | Separate OS processes | Multiple processes sharing a server port |
| **Memory overhead** | ~5-10MB (shares V8 isolate code) | ~30-50MB (full Node.js process) | ~30-50MB per worker |
| **Data sharing** | SharedArrayBuffer, MessagePort (zero-copy), transferable objects | Serialized IPC (JSON over pipe) | Serialized IPC |
| **Isolation** | Crash can affect parent | Full process isolation | Full process isolation |
| **Use case** | CPU-intensive tasks in a server | Run external programs, sandboxed work | Scale HTTP server across CPU cores |

**When to use each:**

**worker_threads** — CPU-bound work that needs to happen alongside your main server without blocking the event loop. Examples: image resizing, hashing, data transformation, heavy computation. You share memory (SharedArrayBuffer) for performance and keep overhead low. Best combined with a worker pool (like `workerpool` or `piscina`) to avoid spawning/destroying threads per request.

**child_process** — Running external programs (`ffmpeg`, `git`, shell scripts) or when you need full process isolation. `spawn()` for long-running processes with streaming I/O, `exec()` for quick one-shot commands. Also useful when you want to run untrusted code — a crash in a child process doesn't take down the parent.

**cluster** — Scaling an HTTP server to use all CPU cores. The primary process forks workers, and the OS distributes incoming connections across them (round-robin on most platforms). Each worker is a full Node.js process with its own event loop.

**What goes wrong with the wrong choice:**

- **Using cluster for CPU work:** Each cluster worker is a full Node.js process (~30-50MB). If you need to parallelize a computation, spawning 8 cluster workers is wasteful — use worker threads instead with ~5MB each.
- **Using worker_threads for running external programs:** Workers run JavaScript. If you need to call `ffmpeg`, you need `child_process.spawn()`.
- **Using child_process for parallel HTTP serving:** You'd have to manage port sharing yourself. Cluster handles this natively with SO_REUSEPORT or the primary distributing connections.
- **Using worker_threads without a pool:** Spawning a new worker per request has high overhead (~50ms startup). Use a pool pattern where workers are pre-spawned and tasks are queued.

```typescript
// Worker pool pattern (using piscina)
import Piscina from 'piscina';

const pool = new Piscina({
  filename: './hash-worker.js',
  maxThreads: 4,
});

// In your request handler:
const result = await pool.run({ data: requestBody });
```

**In Kubernetes/containers:** Cluster is mostly unnecessary — you scale by running multiple single-process containers and letting K8s distribute traffic. Worker threads remain useful for CPU offloading within a single container.

</details>

<details>
<summary>7. How does V8's garbage collector work in Node.js — what is the generational model (young generation Scavenge vs old generation Mark-Sweep-Compact), why do memory leaks happen despite automatic GC, and what are the most common leak patterns in Node.js applications (closures, event listeners, global caches, unbounded collections)?</summary>

**The generational model:**

V8 splits the heap into two generations based on the observation that most objects die young.

**Young Generation (Scavenge / Minor GC):**
- Small space (~1-8MB per semi-space), split into two halves: "from-space" and "to-space."
- New objects are allocated in from-space. When it fills up, a Scavenge runs: live objects are copied to to-space, dead objects are simply abandoned (no deallocation needed — the entire from-space is wiped).
- Objects that survive two Scavenge cycles get promoted to the old generation.
- Very fast (~1-2ms) because it only looks at a small space and most objects are dead.

**Old Generation (Mark-Sweep-Compact / Major GC):**
- Much larger space (default ~1.5GB, controlled by `--max-old-space-size`).
- **Mark phase:** Traverse from root objects (global scope, stack, handles), marking everything reachable as "alive."
- **Sweep phase:** Scan the heap and free unmarked (dead) objects, creating free-list entries.
- **Compact phase:** Optionally moves live objects together to reduce fragmentation. This is the expensive part — it must pause JavaScript execution (stop-the-world).
- V8 mitigates pauses with incremental marking (spreads marking across multiple small steps) and concurrent sweeping (runs on background threads).

**Why leaks happen despite GC:** The GC can only collect objects that are unreachable. A memory leak in a GC language means you're unintentionally keeping references to objects you no longer need. The GC sees them as "still in use."

**The most common leak patterns in Node.js:**

1. **Growing event listeners.** Calling `.on()` in a request handler without `.removeListener()` means every request adds a permanent listener. The emitter holds references to all callback closures, which hold references to their scope.

```typescript
// LEAK: listener added per request, never removed
server.on('request', (req, res) => {
  database.on('error', handleDbError); // accumulates forever
});
```

2. **Closures capturing large scopes.** A closure keeps the entire scope chain alive. If a long-lived callback (timer, event listener) closes over a large variable, that variable can't be collected.

```typescript
function processRequest(largeDataset: Buffer) {
  // This closure keeps largeDataset alive for 1 hour
  setTimeout(() => {
    console.log('done');
    // largeDataset is in scope, even if unused
  }, 3_600_000);
}
```

3. **Unbounded caches / global maps.** A module-level `Map` or object used as a cache without eviction grows forever.

```typescript
const cache = new Map(); // module-level, never cleared
function lookup(key: string) {
  if (!cache.has(key)) {
    cache.set(key, expensiveComputation(key));
  }
  return cache.get(key);
}
// cache grows with every unique key, forever
```

4. **Forgotten timers.** `setInterval()` keeps its callback (and its closure scope) alive until cleared.

5. **Detached DOM-like patterns.** In server contexts: storing references to request/response objects in a module-level structure that outlives the request.

**Detection** is covered in the diagnostics questions later, but the key tool is comparing heap snapshots over time to find objects that grow monotonically.

</details>

<details>
<summary>8. Why does Node.js have two module systems (CommonJS and ES Modules) instead of just one — what are the fundamental differences (synchronous require vs async import, value copies vs live bindings), what tradeoffs does each impose, and why is dual-package publishing necessary but painful?</summary>

**Why two systems exist:** CommonJS (`require`) was invented for Node.js in 2009 — there was no standard. ES Modules (`import/export`) became the official JavaScript standard in ES2015. Node.js couldn't just switch overnight because the npm ecosystem has millions of CJS packages. So both coexist, with interop rules that are workable but imperfect.

**Fundamental differences:**

| | **CommonJS** | **ES Modules** |
|---|---|---|
| **Loading** | Synchronous. `require()` blocks until the file is read and executed. | Asynchronous. Modules are parsed, dependencies resolved, then executed. |
| **Binding** | `module.exports` creates a **value copy**. If the exporting module changes a variable, importers don't see it. | `export` creates a **live binding**. Importers always see the current value. |
| **Evaluation** | Eager — code runs immediately on `require()`. | Deferred — parsed statically, executed after all dependencies are resolved. |
| **Static analysis** | No — `require()` can be called conditionally, with dynamic paths. Bundlers can't reliably tree-shake. | Yes — `import`/`export` are top-level only, statically analyzable. Enables tree-shaking. |
| **`this` at top level** | `this === module.exports` | `this === undefined` (strict mode always) |
| **File extensions** | `.js` (default), `.cjs` | `.mjs`, or `.js` with `"type": "module"` in package.json |

**The live binding distinction matters:**

```typescript
// counter.mjs (ESM)
export let count = 0;
export function increment() { count++; }

// consumer.mjs
import { count, increment } from './counter.mjs';
console.log(count); // 0
increment();
console.log(count); // 1 — live binding sees the change

// In CJS, you'd get 0 both times (value copy of `count` at require time)
```

**Interop rules:**
- ESM can `import` from CJS (Node wraps `module.exports` as the default export).
- CJS cannot `require()` ESM synchronously — you must use dynamic `import()` which returns a Promise.

**Why dual-package publishing is necessary but painful:**

Library authors need to serve both CJS consumers (legacy code, tooling) and ESM consumers (modern code, tree-shaking). The `"exports"` field in package.json enables conditional exports:

```json
{
  "exports": {
    ".": {
      "import": "./dist/esm/index.mjs",
      "require": "./dist/cjs/index.cjs"
    }
  }
}
```

**The pain — the dual package hazard:** If someone uses both `import` and `require()` for the same package in the same app (directly or transitively), Node.js loads it twice — creating two separate instances. Any singleton state, `instanceof` checks, or identity comparisons break silently. This is the core reason dual-package publishing is error-prone: you must ensure both entry points don't create separate stateful instances, typically by having one format be a thin wrapper that re-exports the other.

</details>

<details>
<summary>9. Why is propagating request-scoped context (like trace IDs or user info) difficult in Node.js's async model — how does AsyncLocalStorage solve this without passing context through every function call, what are async_hooks under the hood, and what performance and reliability tradeoffs come with using them?</summary>

**Why it's hard:** In thread-per-request servers (Java, Go), you store context in thread-local storage — each thread handles one request, so the context is naturally isolated. In Node.js, a single thread handles all requests concurrently via async callbacks. There's no "current request" thread to attach context to. Without explicit passing, a logger deep in the call stack has no way to know which request it's serving.

The naive solution — passing `requestId` as a parameter through every function — pollutes every function signature in your codebase:

```typescript
// This is what you're trying to avoid:
async function handleRequest(req: Request, res: Response) {
  const traceId = req.headers['x-trace-id'];
  const user = await getUser(req.userId, traceId);  // pass it
  await saveOrder(user, orderData, traceId);          // pass it again
  logger.info('done', { traceId });                   // and again
}
```

**How AsyncLocalStorage solves it:**

`AsyncLocalStorage` creates a "store" that is automatically propagated through the entire async chain — Promises, async/await, timers, event emitters — without passing it explicitly. It works like thread-local storage, but for async contexts.

```typescript
import { AsyncLocalStorage } from 'node:async_hooks';
import http from 'node:http';
import crypto from 'node:crypto';

interface RequestContext {
  traceId: string;
  userId?: string;
}

const ctxStore = new AsyncLocalStorage<RequestContext>();

// Helper to get context anywhere in the call chain
export function getContext(): RequestContext {
  const ctx = ctxStore.getStore();
  if (!ctx) throw new Error('No request context available');
  return ctx;
}

// Logger automatically includes trace ID
function log(msg: string) {
  const ctx = ctxStore.getStore();
  console.log(`[${ctx?.traceId ?? 'no-ctx'}] ${msg}`);
}

// Deep in the call chain — no traceId parameter needed
async function queryDatabase(userId: string) {
  log(`querying user ${userId}`);
  // ... database call
}

// Wrap each request in a context
http.createServer((req, res) => {
  const context: RequestContext = {
    traceId: req.headers['x-trace-id'] as string ?? crypto.randomUUID(),
  };

  ctxStore.run(context, async () => {
    log('request started');
    await queryDatabase('123');  // log() inside here sees the traceId
    log('request finished');
    res.end('ok');
  });
}).listen(3000);
```

**async_hooks under the hood:**

`async_hooks` is the low-level API that tracks the lifecycle of async resources (Promises, timers, I/O handles). It fires callbacks at four points: `init` (async resource created), `before` (callback about to run), `after` (callback finished), `destroy` (resource garbage collected).

`AsyncLocalStorage` uses async_hooks internally — specifically, it piggybacks on the `init` hook to propagate the store from a parent async context to its children. When a Promise is created inside a `ctxStore.run()` callback, the `init` hook copies the store reference to the new Promise's async context. When that Promise's callback later executes, the store is restored.

**Tradeoffs:**

- **Performance:** AsyncLocalStorage has a measurable but modest overhead (~5-10% in microbenchmarks). In real applications with I/O, the impact is negligible. The raw `async_hooks` API is heavier — enabling all four hooks can add 10-20% overhead.
- **Reliability:** AsyncLocalStorage works correctly with Promises, async/await, timers, and most EventEmitters. It can break with non-context-aware native addons or libraries that use raw callbacks outside the async tracking system (e.g., some connection pool implementations that reuse callbacks across contexts).
- **Debugging:** Context "loss" (getStore() returning undefined) can be hard to debug. It usually means something in the chain escaped the async context — a common culprit is connection pool callbacks or manually constructed Promises in native modules.

AsyncLocalStorage is stable and production-ready (no longer experimental as of Node.js 16). It's the standard approach for request-scoped logging, tracing, and user context in Node.js.

</details>

<details>
<summary>10. How do Express and Fastify differ architecturally — why does Express use a linear middleware chain while Fastify uses an encapsulated plugin system, how does Fastify's schema-based validation and serialization give it a performance advantage, and when would you still choose Express over Fastify?</summary>

**Express architecture — linear middleware chain:**

Express processes every request through a flat array of middleware functions in registration order. Each middleware calls `next()` to pass control to the next one. All middleware shares the same `req`/`res` objects — any middleware can mutate them, and there's no scoping.

```typescript
// Express: everything is global, order matters
app.use(cors());
app.use(helmet());
app.use(authMiddleware);     // runs for ALL routes
app.use('/api/v1', router);  // path-based narrowing is the only scoping
```

This is simple to understand but creates problems at scale: middleware ordering bugs are hard to trace, there's no encapsulation (a plugin can accidentally modify shared state), and tree-shaking middleware per route requires manual discipline.

**Fastify architecture — encapsulated plugin system:**

Fastify organizes code into plugins, each with its own encapsulated context. Decorators (added properties), hooks, and schemas registered in a plugin are only visible to that plugin and its children — not to siblings or parents. This is enforced by Fastify's internal "avvio" boot system.

```typescript
import Fastify from 'fastify';

const app = Fastify({ logger: true });

// Plugin A — its decorators/hooks are isolated
app.register(async function pluginA(instance) {
  instance.decorate('dbConnection', createDbPool());
  instance.addHook('onRequest', authHook);

  instance.get('/users', async (req) => {
    return instance.dbConnection.query('SELECT * FROM users');
  });
});

// Plugin B — cannot see pluginA's dbConnection or authHook
app.register(async function pluginB(instance) {
  instance.get('/health', async () => ({ status: 'ok' }));
});
```

To share something across plugins, you explicitly use `fastify-plugin` (which breaks encapsulation intentionally) — making sharing an opt-in decision rather than the default.

**Schema-based validation and serialization — the performance advantage:**

In Express, you typically validate with a library (Joi, Zod) at runtime and serialize responses with `JSON.stringify()`. Both happen on every request, interpreting schemas dynamically.

Fastify takes JSON Schemas defined per route and compiles them at startup into optimized functions:
- **Validation** uses Ajv, which compiles JSON Schema into a validation function — no schema interpretation at runtime.
- **Serialization** uses `fast-json-stringify`, which compiles a response schema into a serializer that's 2-3x faster than `JSON.stringify()` because it knows the shape ahead of time (no need to inspect types, check for toJSON, handle edge cases).

```typescript
// Fastify: schema compiled at startup, not interpreted per request
app.post('/users', {
  schema: {
    body: {
      type: 'object',
      required: ['name', 'email'],
      properties: {
        name: { type: 'string' },
        email: { type: 'string', format: 'email' },
      },
      additionalProperties: false, // also prevents data leakage
    },
    response: {
      201: {
        type: 'object',
        properties: {
          id: { type: 'integer' },
          name: { type: 'string' },
        },
      },
    },
  },
}, async (request, reply) => {
  const user = await createUser(request.body);
  reply.code(201);
  return user; // serialized with compiled function, not JSON.stringify
});
```

The response schema also acts as a security feature — properties not in the schema are automatically stripped, preventing accidental leakage of internal fields.

**When to choose Express over Fastify:**

- **Existing codebase.** Migrating a large Express app is expensive and risky. Unless performance is a proven bottleneck, the cost usually doesn't justify it.
- **Team familiarity.** Express's middleware model is simpler to onboard new developers to. Fastify's encapsulation model has a steeper learning curve.
- **Ecosystem.** Express has more middleware packages available, though the gap is shrinking. Some niche middleware only exists for Express.
- **Rapid prototyping.** For a quick API that won't see high traffic, Express's simplicity wins.

**When Fastify wins clearly:** High-throughput APIs, microservices where every millisecond of latency matters, projects that benefit from schema-first design (automatic validation, serialization, and documentation generation).

</details>

<details>
<summary>11. Why does graceful shutdown matter in Node.js — what happens if a process just exits on SIGTERM without cleanup, how should you handle SIGTERM vs SIGINT, what's the difference between uncaughtException and unhandledRejection in terms of process safety, and what do different exit codes signal?</summary>

**Why graceful shutdown matters:**

Without cleanup, a hard exit on SIGTERM causes:
- **In-flight HTTP requests** get TCP RST (connection reset) — clients see errors, not responses.
- **Database transactions** are left open — the DB server eventually times out the connection, but data may be in an inconsistent state.
- **Message queue messages** that were dequeued but not acknowledged get redelivered, potentially causing duplicate processing if your consumer isn't idempotent.
- **File writes** may be incomplete or corrupted (partial writes without flush).
- **Connection pools** aren't drained — the database sees connections drop abruptly, consuming cleanup resources.

In Kubernetes, this is especially critical: every rolling deployment sends SIGTERM to old pods. Without graceful shutdown, every deploy causes a brief error spike.

**SIGTERM vs SIGINT:**

| Signal | Source | Convention |
|---|---|---|
| **SIGTERM** | Sent by orchestrators (K8s, Docker, systemd) to request termination | "Please shut down cleanly" — the process should clean up and exit |
| **SIGINT** | Sent by the terminal (Ctrl+C) | "User wants to stop" — same cleanup logic applies |

Both should trigger the same graceful shutdown sequence. The only difference is the source — handle them identically in code:

```typescript
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

**uncaughtException vs unhandledRejection:**

- **`uncaughtException`**: A synchronous throw that nobody caught. The process is in an **undefined state** — you don't know what cleanup ran or didn't. Node.js docs explicitly say: log the error and exit. Do not attempt to resume. The only safe action is a controlled crash.
- **`unhandledRejection`**: A Promise rejected with no `.catch()` handler. Since Node.js 15, this defaults to throwing (crashing the process). In older versions, it was just a warning. The right approach: log it and exit, same as uncaughtException — an unhandled rejection means your error handling has a gap.

```typescript
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception — exiting');
  process.exit(1); // crash immediately, let orchestrator restart
});

process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'Unhandled rejection — exiting');
  process.exit(1);
});
```

**Exit codes:**

| Code | Meaning |
|---|---|
| **0** | Success — clean exit |
| **1** | Generic error (uncaught exception, application error) |
| **2** | Misuse of shell command (rarely used by Node) |
| **12** | Invalid debug argument |
| **128 + N** | Terminated by signal N (e.g., 137 = SIGKILL (128+9), 143 = SIGTERM (128+15)) |

When you handle SIGTERM yourself, exit with code 0 (clean shutdown succeeded) or set `process.exitCode = 0` and let pending operations finish. If shutdown times out, exit with 1.

</details>

<details>
<summary>12. Why does Node.js distinguish between operational errors and programmer errors — how should each be handled differently in production (retry vs crash), what's the "let it crash" philosophy, and where do unhandled promise rejections fit in this model?</summary>

**The distinction:**

- **Operational errors** are expected failures in a correctly written program: network timeouts, DNS resolution failures, database connection refused, file not found, invalid user input, rate limit exceeded. They happen because the outside world is unreliable.
- **Programmer errors** are bugs: `TypeError: Cannot read property of undefined`, out-of-bounds array access, passing a string where a number is expected, forgetting to `await` a Promise. They happen because the code is wrong.

**Why the distinction matters for handling:**

| | Operational | Programmer |
|---|---|---|
| **Can you recover?** | Yes — retry, fallback, return error to client | No — the code path is broken, state may be corrupt |
| **Action** | Handle gracefully: retry with backoff, circuit break, return 4xx/5xx | Crash the process and fix the bug |
| **Logging** | Warn or error level | Fatal level |
| **User impact** | Degraded service, retry possible | Must restart process |

**The "let it crash" philosophy:**

Borrowed from Erlang. The idea: when a programmer error occurs, don't try to recover — you don't know what state the process is in. Instead, crash immediately and let the supervisor (K8s, PM2, systemd) restart a fresh process. A clean restart is safer than continuing with potentially corrupted in-memory state.

This works because:
- Node.js processes are stateless (or should be) — restart is cheap.
- Orchestrators restart crashed containers in seconds.
- Trying to "recover" from a bug often makes things worse (silent data corruption, cascading failures).

**Where unhandled promise rejections fit:**

An unhandled rejection is almost always a programmer error — you forgot a `.catch()` or a `try/catch` around an `await`. Since Node.js 15, unhandled rejections crash the process by default (`--unhandled-rejections=throw`), which is the correct behavior under "let it crash."

The exception: sometimes a library rejects a Promise that you intentionally don't handle (e.g., a race where you discard the loser). In those cases, use explicit handling — don't suppress the global behavior.

**Practical pattern:**

```typescript
// Operational — handle gracefully
async function fetchUserProfile(userId: string) {
  try {
    return await httpClient.get(`/users/${userId}`);
  } catch (err) {
    if (err.code === 'ECONNREFUSED') {
      throw new ServiceUnavailableError('User service is down');
    }
    if (err.response?.status === 404) {
      return null; // user doesn't exist — expected case
    }
    throw err; // unexpected — let it bubble up
  }
}

// Programmer — let it crash
// If someone passes undefined as userId, the TypeError
// should crash the process (in dev) or be caught by the
// top-level error handler and logged as fatal
```

</details>

<details>
<summary>13. What production diagnostics does a Node.js application need — when would you reach for CPU profiling vs heap snapshots vs event loop lag monitoring vs distributed tracing, how do you decide which tool to use based on the symptoms, and what signals tell you an app is healthy vs degrading?</summary>

**Symptom-to-tool mapping:**

| Symptom | Tool | Why |
|---|---|---|
| High CPU usage, slow responses across all endpoints | **CPU profiling** (--prof, Chrome DevTools, 0x) | Identifies hot functions eating CPU cycles |
| Memory growing over time, OOM crashes | **Heap snapshots** (--inspect + DevTools, heapdump) | Shows retained objects causing leaks |
| Intermittent slowness, requests queuing up | **Event loop lag monitoring** (monitorEventLoopDelay) | Detects blocked event loop — something synchronous is holding it |
| Slow responses on specific paths, not all | **Distributed tracing** (OpenTelemetry) | Pinpoints which service/DB call in the chain is slow |
| Memory spikes under load, fine at rest | **process.memoryUsage()** + heap snapshots under load | Differentiates leak from normal allocation patterns |

**When to reach for each:**

**CPU profiling** — When the process is using more CPU than expected, or p50 latency is high across all endpoints (not just specific ones). Generate a flame graph to find functions consuming the most time. Look for wide bars (functions taking a long time) and tall stacks (deep call chains).

**Heap snapshots** — When `process.memoryUsage().heapUsed` trends upward over hours/days, or when you get OOM kills. Take two snapshots minutes apart, compare them in Chrome DevTools to find objects growing in count. Common culprits: Maps without eviction, event listeners accumulating, closures holding references.

**Event loop lag monitoring** — When latency spikes are intermittent and correlate with specific operations. Use `perf_hooks.monitorEventLoopDelay()` to measure how long callbacks wait before executing:

```typescript
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

// Expose as a Prometheus metric periodically
setInterval(() => {
  console.log(`Event loop p99: ${histogram.percentile(99) / 1e6}ms`);
  histogram.reset();
}, 10_000);
```

Healthy: p99 < 10ms. Degraded: p99 > 50ms. Blocked: p99 > 100ms.

**Distributed tracing** — When slowness is in a specific endpoint that calls other services or databases. Tracing shows the waterfall of calls with durations, so you can see if latency is in your code, a downstream service, or a database query.

**Health signals to monitor continuously:**

- **Event loop lag** — the single most important Node.js-specific metric
- **Heap used / heap total** — watch for monotonic growth
- **Active handles and requests** — `process._getActiveHandles().length`, `process._getActiveRequests().length` — growing counts suggest resource leaks
- **GC duration and frequency** — via `--trace-gc` or `perf_hooks` GC observation — frequent long pauses indicate memory pressure
- **Response time percentiles** — p50, p95, p99 at the application level
- **Error rate** — sudden increase signals something broke

</details>

<details>
<summary>14. How can process.nextTick() and recursive microtasks starve the event loop — why do microtasks and nextTick callbacks run to completion before the event loop advances to the next phase, what real-world scenarios cause this (recursive nextTick, unbounded Promise chains), and how do you detect and prevent it?</summary>

**Why starvation happens:**

As covered in the event loop question (Q3), between every phase transition the event loop drains two queues completely: the nextTick queue first, then the microtask queue. "Drain completely" is the key — if draining one queue adds more items to the same queue, the loop keeps draining. It never moves to the next phase until both queues are empty.

This means I/O callbacks, timers, and setImmediate are all stuck waiting. The event loop is alive (not crashed), but it's trapped in an infinite microtask/nextTick loop.

**Starvation example with recursive nextTick:**

```typescript
// This starves the event loop — setTimeout never fires
function recursiveNextTick() {
  process.nextTick(recursiveNextTick);
}
recursiveNextTick();

setTimeout(() => {
  console.log('This never runs');
}, 100);
```

**Starvation with recursive Promise resolution:**

```typescript
// Same problem with Promises
function recursiveMicrotask() {
  Promise.resolve().then(recursiveMicrotask);
}
recursiveMicrotask();

// I/O, timers, and setImmediate are all blocked
```

**Real-world scenarios:**

1. **Unbounded recursive data processing with Promises.** Processing a large list by recursively calling `Promise.resolve().then(() => processNext())` instead of using a loop or `setImmediate()` for batching.

2. **Event handlers that trigger themselves.** An EventEmitter listener that, in its handler, emits the same event synchronously or via `process.nextTick()`.

3. **Library code with recursive nextTick.** Some older stream implementations used recursive `nextTick` for scheduling, which could stall under high throughput.

**How to detect it:**

- **Event loop lag spikes to infinity.** `monitorEventLoopDelay()` shows the event loop isn't advancing — the delay grows unbounded.
- **All I/O stops.** HTTP requests hang, timers don't fire, but the process is using CPU (it's running microtasks, not idle).
- **`--prof` or CPU profile** shows all time spent in microtask/nextTick callbacks.

**How to prevent it:**

Use `setImmediate()` instead of `process.nextTick()` for recursive scheduling. `setImmediate` runs in the check phase, which means the event loop completes a full cycle (including I/O) before executing the callback:

```typescript
// SAFE: yields to the event loop between iterations
function processItems(items: string[], index = 0) {
  if (index >= items.length) return;

  doWork(items[index]);

  // setImmediate lets I/O and timers run between iterations
  setImmediate(() => processItems(items, index + 1));
}
```

**Rule of thumb:** Use `process.nextTick()` only for one-shot scheduling (like emitting an event after construction). For anything recursive or unbounded, use `setImmediate()`.

</details>

<details>
<summary>15. What are the most common ways developers accidentally block the Node.js event loop — how do synchronous API calls (crypto.pbkdf2Sync, fs.readFileSync, zlib.inflateSync), large JSON.stringify/parse operations, and O(n^2) algorithms in request handlers cause latency spikes under load, and how does partitioning work with setImmediate() compare to offloading to worker threads as a mitigation strategy?</summary>

**The common blockers:**

**1. Synchronous API calls:**

`crypto.pbkdf2Sync()` with 100k iterations blocks for ~70-100ms. `fs.readFileSync()` on a large file blocks until the entire read completes. `zlib.inflateSync()` on a large payload blocks during decompression. In a request handler, this means every other request waits.

The trap: these work fine in development with one request at a time. Under concurrent load, they serialize all requests through that synchronous bottleneck.

**2. Large JSON operations:**

`JSON.stringify()` and `JSON.parse()` are synchronous and O(n) in the size of the data. Stringifying a 10MB object can take 50-100ms — blocking the event loop for that duration. This is especially dangerous because it's not an obvious "sync" call — developers don't realize `JSON.stringify()` blocks.

A related attack vector: JSON.parse DoS — sending a deeply nested JSON payload that takes exponentially longer to parse.

**3. O(n^2) algorithms in request handlers:**

Nested loops over request data (e.g., deduplication with `.includes()` inside `.filter()`, naive string matching, sorting with a bad comparator) run fine with 10 items but block for seconds with 10,000 items.

```typescript
// O(n^2) — blocks the event loop with large arrays
function removeDuplicates(items: string[]) {
  return items.filter((item, i) => items.indexOf(item) === i);
}
// Fix: use a Set — O(n)
function removeDuplicates(items: string[]) {
  return [...new Set(items)];
}
```

**Mitigation: setImmediate partitioning vs worker threads:**

**setImmediate partitioning** breaks work into chunks, yielding to the event loop between each chunk. Good for CPU-light work that's just too much in one go (processing 100k items):

```typescript
async function processInBatches(items: string[], batchSize = 1000) {
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    batch.forEach(processItem);

    // Yield to event loop — lets I/O and other requests through
    await new Promise((resolve) => setImmediate(resolve));
  }
}
```

**Worker threads** offload the entire computation to a separate thread. Better for genuinely CPU-intensive work (hashing, compression, image processing) where even partitioning would cause unacceptable total latency:

| | setImmediate partitioning | Worker threads |
|---|---|---|
| **Overhead** | Near zero | ~5-10MB per thread, message serialization |
| **Total time** | Same (or slightly slower due to yields) | Same, but doesn't block main thread at all |
| **Complexity** | Low — just wrap in a loop | Higher — separate worker file, message passing |
| **Best for** | Large iterations, batch processing | Crypto, compression, image/video processing |
| **Latency impact** | Reduces spikes but still uses main thread CPU | Zero impact on main thread |

**Rule of thumb:** If the blocking operation takes < 5ms per chunk, partition with setImmediate. If it takes > 50ms total, offload to a worker thread (as covered in Q6, use a pool like piscina to avoid per-request spawn overhead).

</details>

<details>
<summary>16. Why do lockfiles (package-lock.json) exist alongside package.json when package.json already specifies versions — how does semver resolution work, what problems do peer dependencies solve (and what pain do they cause), how do npm workspaces help in monorepos, and what are the security implications of lockfile integrity and registry configuration?</summary>

**Why lockfiles exist:**

`package.json` specifies version **ranges** (`^1.2.3`, `~1.2.3`, `>=1.0.0`). When you run `npm install`, npm resolves those ranges to specific versions based on what's available in the registry *at that moment*. Without a lockfile, the same `package.json` can produce different `node_modules/` trees on different machines or at different times — leading to "works on my machine" bugs.

`package-lock.json` records the exact resolved version of every package (including transitive dependencies) and their integrity hashes. `npm ci` (the CI-focused install) uses only the lockfile, ignoring `package.json` ranges — guaranteeing identical installs everywhere.

**Semver resolution:**

Semver format: `MAJOR.MINOR.PATCH`.
- `^1.2.3` — allows minor + patch updates: `>=1.2.3 <2.0.0` (most common, default for `npm install`)
- `~1.2.3` — allows patch updates only: `>=1.2.3 <1.3.0`
- `1.2.3` — exact version, no flexibility

The caret (`^`) trusts that library authors follow semver — minor versions add features without breaking, patches fix bugs. In practice, this trust is sometimes misplaced (a "minor" update breaks your code), which is exactly why lockfiles are essential.

**Peer dependencies — problem and pain:**

Peer dependencies declare "I need this package, but the consumer must provide it — don't install a separate copy." They solve the **singleton problem**: plugins that must share a single instance of a host package.

Example: A React component library declares `react` as a peer dependency. If it bundled its own React, you'd have two React instances — hooks would break, context wouldn't propagate.

**The pain:**
- npm 7+ auto-installs peer dependencies, which can cause version conflicts if two packages need different ranges.
- Conflicting peer dependency ranges produce `ERESOLVE` errors that are confusing and require `--legacy-peer-deps` or manual resolution.
- Library authors must keep peer dependency ranges wide enough to be useful but narrow enough to be safe.

**npm workspaces in monorepos:**

Workspaces let multiple packages in a monorepo share a single `node_modules/` at the root, with symlinks for cross-package references:

```json
{
  "workspaces": ["packages/*"]
}
```

Benefits: shared dependencies are deduplicated (one copy of TypeScript, not five), cross-package `import` works via symlinks without publishing, and `npm run --workspaces` executes scripts across all packages.

**Security implications:**

- **Lockfile integrity hashes:** Each dependency in the lockfile has an `integrity` field (SHA-512 hash of the tarball). `npm ci` verifies these hashes — if a package was tampered with on the registry, the hash won't match and the install fails. Always commit lockfiles and use `npm ci` in CI.
- **Registry configuration:** `.npmrc` can point to custom registries (e.g., Artifactory, GitHub Packages). A misconfigured `.npmrc` or a missing scope mapping could silently pull packages from the public registry instead of your private one — a vector for dependency confusion attacks.
- **`npm audit`:** Scans your dependency tree against the GitHub Advisory Database. It catches known vulnerabilities but not zero-days or malicious packages that haven't been reported yet. It's a baseline, not a complete security strategy.
- **Install scripts:** Packages can run arbitrary code during `npm install` via `preinstall`/`postinstall` scripts. This is the primary vector for supply chain attacks. Use `--ignore-scripts` for untrusted packages, and consider tools like `socket.dev` for deeper analysis.

</details>

<details>
<summary>17. How does the Node.js release cycle work (Current vs LTS vs Maintenance), why does it matter which line you run in production, and what notable recent additions (built-in test runner, native fetch, permission model) reduce the need for third-party dependencies — how do you manage multiple Node.js versions across projects with nvm or fnm?</summary>

**Release cycle:**

Node.js releases a new major version every 6 months (April and October). Even-numbered versions become LTS; odd-numbered versions never do.

```
Timeline for a version (e.g., Node.js 22):

April 2024     → "Current" release (6 months)
October 2024   → Enters "Active LTS" (18 months of active support)
April 2026     → Enters "Maintenance LTS" (12 months, security fixes only)
April 2027     → End of life
```

| Phase | Duration | What you get |
|---|---|---|
| **Current** | 6 months | New features, may have breaking changes between minors |
| **Active LTS** | 18 months | Bug fixes, security patches, non-breaking backports |
| **Maintenance** | 12 months | Critical security fixes only |

**Why it matters for production:**

Run Active LTS in production. Current releases are for testing upcoming features — they may have bugs or instability that haven't been caught yet. Maintenance LTS means you're getting only emergency patches and should be planning your upgrade. Running an EOL version means no security patches at all.

**Notable recent additions reducing third-party needs:**

- **Built-in test runner** (stable in Node.js 20): `node:test` module with `describe`, `it`, `beforeEach`, mocking, and code coverage. Replaces Jest/Mocha for many use cases, especially in libraries.
- **Native fetch** (stable in Node.js 21): Built on undici, no need for `node-fetch` or `axios` for basic HTTP requests. Includes `Request`, `Response`, `Headers` globals.
- **Permission model** (experimental, Node.js 20+): `--experimental-permission` flag restricts file system access, child process spawning, and worker creation. Run with `--allow-fs-read=/app/config` to whitelist specific paths.
- **Native watch mode** (`--watch`, stable in Node.js 22): Restarts the process on file changes. Replaces `nodemon` for development.
- **Single-file executables** (experimental): Compile a Node.js app into a standalone binary without requiring Node.js on the target machine.

**Managing versions with nvm/fnm:**

Both tools let you switch Node.js versions per project using a `.nvmrc` or `.node-version` file:

```bash
# .nvmrc in project root
22.12.0

# Switch versions
nvm use           # reads .nvmrc
fnm use           # reads .node-version or .nvmrc
```

**fnm vs nvm:** fnm is written in Rust, significantly faster (instant version switches vs nvm's ~200ms shell overhead), and supports automatic switching via shell hooks. nvm is the established default with wider adoption. For new setups, fnm is the better choice.

</details>

<details>
<summary>18. What are the most dangerous Node.js-specific security vulnerabilities — how does prototype pollution work and why is it especially dangerous in JavaScript, what makes regular expressions vulnerable to ReDoS attacks, why is child_process.exec vulnerable to shell injection while execFile is not, how do dependency supply chain attacks happen (typosquatting, compromised maintainers, install scripts), and what do npm audit and lockfile integrity checks actually protect against?</summary>

**1. Prototype pollution:**

JavaScript objects inherit from `Object.prototype`. If an attacker can set properties on `Object.prototype`, those properties appear on *every* object in the application.

```typescript
// Vulnerable deep merge function
function merge(target: any, source: any) {
  for (const key of Object.keys(source)) {
    if (typeof source[key] === 'object') {
      target[key] = target[key] || {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
}

// Attack: send JSON body with __proto__
const malicious = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, malicious);

// Now EVERY object has isAdmin === true
const user = {};
console.log((user as any).isAdmin); // true
```

**Why it's especially dangerous in JavaScript:** The prototype chain is implicit — developers don't expect `{}` to have properties they didn't set. It can bypass authorization checks, enable property injection in templating engines (leading to RCE), or pollute configuration objects.

**Prevention:** Use `Object.create(null)` for dictionaries, validate/sanitize keys (reject `__proto__`, `constructor`, `prototype`), use `Map` instead of plain objects for dynamic keys, freeze prototypes in security-critical code.

**2. ReDoS (Regular Expression Denial of Service):**

Some regex patterns have catastrophic backtracking — they take exponential time on specific inputs. The pattern `/(a+)+$/` on input `"aaaaaaaaaaaaaaaaaaaX"` causes the regex engine to try every possible combination of `a` groupings before failing.

```typescript
// Vulnerable: nested quantifiers
const emailRegex = /^([a-zA-Z0-9]+\.)+[a-zA-Z]{2,}$/;
// Attack input: 'aaaaaaaaaaaaaaaaaaaaaaaaaaa!'
// This can take seconds to evaluate, blocking the event loop
```

**Prevention:** Avoid nested quantifiers (`(a+)+`, `(a|b)*c*`), use `re2` (Google's regex library that guarantees linear time), set timeouts on regex execution, validate input length before regex matching.

**3. Shell injection — exec vs execFile:**

```typescript
import { exec, execFile } from 'node:child_process';

const userInput = '; rm -rf /'; // malicious

// VULNERABLE: exec passes the entire string to /bin/sh
exec(`ls ${userInput}`); // executes: sh -c "ls ; rm -rf /"

// SAFE: execFile doesn't use a shell — arguments are passed directly
execFile('ls', [userInput]); // passes "; rm -rf /" as a literal filename argument
```

`exec()` spawns a shell and passes the command as a string — shell metacharacters (`;`, `|`, `&&`, `$()`) are interpreted. `execFile()` invokes the binary directly with arguments as an array — no shell interpretation happens. Always use `execFile()` or `spawn()` (without `shell: true`) when user input is involved.

**4. Supply chain attacks:**

- **Typosquatting:** Publishing `expresss` (three s's) hoping developers mistype `npm install express`. The malicious package runs code on install.
- **Compromised maintainers:** An attacker gains access to a maintainer's npm account and publishes a malicious update to a popular package (happened with `event-stream` in 2018).
- **Malicious install scripts:** `preinstall`/`postinstall` scripts run arbitrary code during `npm install` — they can exfiltrate environment variables (including secrets), install backdoors, or modify other packages.
- **Dependency confusion:** A company uses a private package `@company/utils`. An attacker publishes `@company/utils` to the public registry with a higher version number — npm may resolve to the public one.

**What npm audit and lockfile integrity protect against:**

- **npm audit:** Checks your dependency tree against the GitHub Advisory Database for *known* vulnerabilities. It catches CVEs that have been reported and cataloged. It does NOT catch zero-day exploits, malicious packages that haven't been reported, or typosquatting.
- **Lockfile integrity:** The `integrity` field (SHA-512 hash) in `package-lock.json` ensures the exact tarball you resolved is the one installed. If a package on the registry was replaced with a different tarball (compromised registry, MITM), the hash check fails. Use `npm ci` (which enforces integrity checks) in CI pipelines, never `npm install`.

</details>

## Practical — Implementation & Configuration

<details>
<summary>19. Build a stream pipeline that reads a large file, transforms each line (e.g., parse CSV to JSON), and writes the output to another file — use pipeline() instead of pipe(), implement proper backpressure handling, show what happens when the writable side is slower than the readable side, and explain why pipeline() is preferred for error handling and cleanup</summary>

```typescript
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline, Transform } from 'node:stream';
import { pipeline as pipelinePromise } from 'node:stream/promises';
import { createInterface } from 'node:readline';

// Approach 1: Using async generator with stream/promises pipeline
// (modern, clean, recommended)
async function csvToJsonPipeline(inputPath: string, outputPath: string) {
  const readable = createReadStream(inputPath, { encoding: 'utf8' });
  const writable = createWriteStream(outputPath);

  let headers: string[] | null = null;
  let isFirst = true;

  await pipelinePromise(
    readable,
    // Async generator acts as a transform — backpressure is handled automatically
    async function* (source) {
      yield '[\n';
      for await (const chunk of source) {
        const lines = chunk.split('\n').filter((l: string) => l.trim());
        for (const line of lines) {
          const fields = line.split(',');
          if (!headers) {
            headers = fields;
            continue;
          }
          const obj: Record<string, string> = {};
          headers.forEach((h, i) => (obj[h.trim()] = fields[i]?.trim() ?? ''));

          const prefix = isFirst ? '' : ',\n';
          isFirst = false;
          yield `${prefix}  ${JSON.stringify(obj)}`;
        }
      }
      yield '\n]\n';
    },
    writable,
  );
}

// Approach 2: Using a Transform class (traditional, more control)
class CsvToJsonTransform extends Transform {
  private headers: string[] | null = null;
  private isFirst = true;
  private remainder = '';

  constructor() {
    super({ encoding: 'utf8' }); // object mode off, strings in/out
  }

  _transform(chunk: Buffer, _encoding: string, callback: Function) {
    const data = this.remainder + chunk.toString();
    const lines = data.split('\n');
    this.remainder = lines.pop() ?? ''; // last line may be incomplete

    for (const line of lines) {
      if (!line.trim()) continue;
      const fields = line.split(',');

      if (!this.headers) {
        this.headers = fields;
        this.push('[\n');
        continue;
      }

      const obj: Record<string, string> = {};
      this.headers.forEach((h, i) => (obj[h.trim()] = fields[i]?.trim() ?? ''));

      const prefix = this.isFirst ? '' : ',\n';
      this.isFirst = false;
      this.push(`${prefix}  ${JSON.stringify(obj)}`);
    }
    callback(); // signal we're done with this chunk
  }

  _flush(callback: Function) {
    // Handle any remaining data
    if (this.remainder.trim() && this.headers) {
      const fields = this.remainder.split(',');
      const obj: Record<string, string> = {};
      this.headers.forEach((h, i) => (obj[h.trim()] = fields[i]?.trim() ?? ''));
      this.push(`,\n  ${JSON.stringify(obj)}`);
    }
    this.push('\n]\n');
    callback();
  }
}

// Usage with callback-style pipeline
pipeline(
  createReadStream('large-data.csv'),
  new CsvToJsonTransform(),
  createWriteStream('output.json'),
  (err) => {
    if (err) console.error('Pipeline failed:', err);
    else console.log('Pipeline complete');
  },
);
```

**What happens when the writable side is slower:**

Backpressure kicks in automatically within `pipeline()`. When the Writable's internal buffer fills past `highWaterMark`, the Transform's `push()` returns `false`. The pipeline pauses the Readable, stopping data from flowing. When the Writable drains its buffer, it emits `'drain'`, the pipeline resumes the Readable, and data flows again. Memory stays bounded — you never buffer the entire file.

**Why pipeline() over pipe():**

| | `pipe()` | `pipeline()` |
|---|---|---|
| **Error handling** | Must attach `.on('error')` to **every** stream individually. An error on one stream doesn't clean up others. | Single error callback/Promise. Errors from any stream in the chain are propagated. |
| **Cleanup** | Broken streams are NOT automatically destroyed. A failed Readable doesn't close the Writable — you leak file descriptors. | All streams are destroyed on error or completion. No resource leaks. |
| **API** | Returns the destination stream (chainable but awkward for error handling) | Returns void (callback) or Promise (promise version) |

```typescript
// pipe() — error in transform doesn't close the writable
readable.pipe(transform).pipe(writable);
// If transform errors, writable stays open — file descriptor leak

// pipeline() — everything is cleaned up
pipeline(readable, transform, writable, (err) => { /* single handler */ });
```

Always use `pipeline()`. There's no reason to use `pipe()` in new code.

</details>

<details>
<summary>20. Implement a CPU-intensive task (e.g., hashing or data processing) using worker_threads — show the main thread and worker code, demonstrate message passing with postMessage and SharedArrayBuffer, and explain when you'd use a worker thread pool pattern vs spawning workers on demand</summary>

**Basic worker thread — message passing with postMessage:**

```typescript
// main.ts
import { Worker } from 'node:worker_threads';
import { cpus } from 'node:os';

interface HashResult {
  hash: string;
  durationMs: number;
}

function hashInWorker(data: string, iterations: number): Promise<HashResult> {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./hash-worker.ts', {
      workerData: { data, iterations },
    });

    worker.on('message', (result: HashResult) => resolve(result));
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

// Usage in an HTTP handler — main thread stays unblocked
const result = await hashInWorker(requestBody, 100_000);
```

```typescript
// hash-worker.ts
import { parentPort, workerData } from 'node:worker_threads';
import { createHash } from 'node:crypto';

const { data, iterations } = workerData as { data: string; iterations: number };

const start = performance.now();
let hash = data;
for (let i = 0; i < iterations; i++) {
  hash = createHash('sha256').update(hash).digest('hex');
}

parentPort?.postMessage({
  hash,
  durationMs: performance.now() - start,
});
```

**SharedArrayBuffer — zero-copy shared memory:**

```typescript
// main.ts — shared memory for high-throughput data exchange
import { Worker } from 'node:worker_threads';

// Create shared memory visible to both threads
const sharedBuffer = new SharedArrayBuffer(1024 * 1024); // 1MB
const sharedArray = new Int32Array(sharedBuffer);

// Fill with data to process
for (let i = 0; i < 1000; i++) {
  sharedArray[i] = i;
}

const worker = new Worker('./compute-worker.ts', {
  workerData: { sharedBuffer, length: 1000 },
});

worker.on('message', () => {
  // Worker modified sharedArray in place — no serialization cost
  console.log('Sum computed by worker:', sharedArray[0]);
});
```

```typescript
// compute-worker.ts
import { parentPort, workerData } from 'node:worker_threads';

const { sharedBuffer, length } = workerData;
const arr = new Int32Array(sharedBuffer);

// Compute directly on shared memory — no copy
let sum = 0;
for (let i = 0; i < length; i++) {
  sum += arr[i];
}
arr[0] = sum; // write result back to shared memory

// Use Atomics for thread-safe operations when multiple workers share memory
// Atomics.add(arr, 0, localSum);
// Atomics.notify(arr, 0); // wake waiting threads

parentPort?.postMessage('done'); // signal completion
```

**Worker pool pattern — using piscina:**

```typescript
// pool-setup.ts
import Piscina from 'piscina';

const pool = new Piscina({
  filename: './hash-worker.ts',
  minThreads: 2,
  maxThreads: Math.max(1, cpus().length - 1), // leave one core for main thread
  idleTimeout: 30_000, // destroy idle threads after 30s
});

// In request handler — tasks queue automatically when all threads are busy
app.post('/hash', async (req, res) => {
  const result = await pool.run({ data: req.body.data, iterations: 100_000 });
  res.json(result);
});
```

**Pool vs on-demand spawning:**

| | Worker pool | Spawn on demand |
|---|---|---|
| **Startup cost** | Paid once at boot (~50ms per thread) | Paid per request (~50ms latency added) |
| **Memory** | Fixed — predictable resource usage | Grows with concurrency — risk of OOM |
| **Queuing** | Built in — tasks wait for free thread | Must implement yourself or spawn unbounded |
| **Best for** | Server handling repeated CPU tasks | One-off scripts, infrequent operations |

**Use a pool** (piscina, workerpool) for any server-side CPU offloading. **Spawn on demand** only for CLI tools, scripts, or operations that happen once per process lifetime. As covered in Q6, spawning a worker per request is the most common mistake — the thread creation overhead defeats the purpose.

</details>

<details>
<summary>21. Implement AsyncLocalStorage to propagate a request ID through an HTTP request's entire lifecycle (middleware → service → database call → logger) — show the setup code, how to access the store in nested async functions, and what breaks if you use callbacks that aren't async-context-aware</summary>

This builds on the AsyncLocalStorage concepts from Q9 with a complete, production-ready implementation.

**Setup — context module:**

```typescript
// context.ts — single source of truth for request context
import { AsyncLocalStorage } from 'node:async_hooks';

interface RequestContext {
  requestId: string;
  userId?: string;
  startTime: number;
}

export const requestContext = new AsyncLocalStorage<RequestContext>();

export function getRequestId(): string {
  return requestContext.getStore()?.requestId ?? 'no-request-id';
}

export function getContext(): RequestContext {
  const ctx = requestContext.getStore();
  if (!ctx) throw new Error('Request context not available — are you inside a request?');
  return ctx;
}
```

**Logger — automatically includes request ID:**

```typescript
// logger.ts
import { getRequestId } from './context.js';

export const logger = {
  info(msg: string, meta?: Record<string, unknown>) {
    console.log(JSON.stringify({
      level: 'info',
      requestId: getRequestId(),
      msg,
      ...meta,
      timestamp: new Date().toISOString(),
    }));
  },
  error(msg: string, meta?: Record<string, unknown>) {
    console.error(JSON.stringify({
      level: 'error',
      requestId: getRequestId(),
      msg,
      ...meta,
    }));
  },
};
```

**HTTP server — wrapping each request:**

```typescript
// server.ts
import http from 'node:http';
import crypto from 'node:crypto';
import { requestContext } from './context.js';
import { logger } from './logger.js';
import { UserService } from './user-service.js';

const userService = new UserService();

const server = http.createServer((req, res) => {
  const context = {
    requestId: (req.headers['x-request-id'] as string) ?? crypto.randomUUID(),
    startTime: Date.now(),
  };

  // .run() creates the async context — everything inside inherits it
  requestContext.run(context, async () => {
    logger.info(`${req.method} ${req.url}`);

    try {
      const user = await userService.findById('123');
      // logger calls inside findById automatically have the requestId
      res.writeHead(200, { 'content-type': 'application/json' });
      res.end(JSON.stringify(user));
    } catch (err) {
      logger.error('Request failed', { error: (err as Error).message });
      res.writeHead(500);
      res.end('Internal error');
    }

    logger.info(`Completed in ${Date.now() - context.startTime}ms`);
  });
});
```

**Service and database layers — no context passing needed:**

```typescript
// user-service.ts
import { logger } from './logger.js';
import { db } from './database.js';

export class UserService {
  async findById(id: string) {
    logger.info('Looking up user', { userId: id });
    // requestId is in the log automatically
    const user = await db.query('SELECT * FROM users WHERE id = $1', [id]);
    logger.info('User found');
    return user;
  }
}

// database.ts
import { logger } from './logger.js';

export const db = {
  async query(sql: string, params: unknown[]) {
    const start = Date.now();
    // simulate database call
    const result = { id: params[0], name: 'Alice' };
    logger.info('SQL query', { sql, durationMs: Date.now() - start });
    // requestId appears in this log too — 3 layers deep
    return result;
  },
};
```

**What breaks — callbacks that aren't async-context-aware:**

The store propagates through Promises, async/await, timers, and EventEmitters because Node.js tracks the async context chain. It breaks when something creates a callback outside the async tracking system:

```typescript
import { requestContext } from './context.js';

// 1. Connection pool reusing callbacks across requests
// Some pool libraries cache callbacks and reuse them — the callback
// runs in the context of whenever the pool was initialized, not the
// current request.
pool.query('SELECT 1', (err, result) => {
  // requestContext.getStore() might be undefined or wrong request's context
  // if the pool internally reuses callback wrappers
});

// 2. Native addons with C++ callbacks
// If a native addon schedules a callback outside V8's async hooks,
// the context is lost.

// 3. Manual context loss with raw event emitters
const emitter = new EventEmitter();
requestContext.run({ requestId: 'abc' }, () => {
  // This works — listener is registered inside the context
  emitter.on('data', () => {
    console.log(requestContext.getStore()?.requestId); // 'abc'
  });
});

// But if the listener was registered OUTSIDE run(), it has no context
emitter.on('data', () => {
  console.log(requestContext.getStore()?.requestId); // undefined
});
```

**Diagnosing context loss:** If `getStore()` returns `undefined` unexpectedly, the callback was created outside the `run()` scope. Check if the library creating the callback is pooling or caching callbacks across async boundaries. The fix is usually to ensure the `.run()` wrapper is high enough in the call chain to encompass all work for that request.

</details>

<details>
<summary>22. Set up the same REST endpoint in both Express (middleware chain) and Fastify (plugin with JSON schema validation) — show both implementations side by side, demonstrate how Fastify's schema-based serialization avoids JSON.stringify overhead, and explain the performance difference</summary>

This builds on the architectural comparison in Q10 with concrete code.

**The endpoint:** POST `/users` — validate body, create user, return response.

**Express implementation:**

```typescript
import express from 'express';
import { z } from 'zod';

const app = express();
app.use(express.json());

// Validation schema (interpreted at runtime on every request)
const createUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'guest']).default('user'),
});

// Validation middleware
function validate(schema: z.ZodSchema) {
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.flatten().fieldErrors,
      });
    }
    req.body = result.data;
    next();
  };
}

app.post('/users', validate(createUserSchema), async (req, res) => {
  const user = await createUser(req.body);

  // JSON.stringify runs on every response, inspecting every property
  // dynamically — no knowledge of the shape ahead of time
  res.status(201).json({
    id: user.id,
    name: user.name,
    email: user.email,
    role: user.role,
    // Bug risk: if user has a 'passwordHash' field, it leaks unless
    // you manually exclude it. Nothing stops you from sending extra fields.
  });
});
```

**Fastify implementation:**

```typescript
import Fastify from 'fastify';

const app = Fastify({ logger: true });

// Plugin encapsulates the /users routes
app.register(async function userRoutes(instance) {
  instance.post('/users', {
    schema: {
      body: {
        type: 'object',
        required: ['name', 'email'],
        properties: {
          name: { type: 'string', minLength: 1, maxLength: 100 },
          email: { type: 'string', format: 'email' },
          role: { type: 'string', enum: ['admin', 'user', 'guest'], default: 'user' },
        },
        additionalProperties: false, // rejects unknown fields
      },
      response: {
        201: {
          type: 'object',
          properties: {
            id: { type: 'integer' },
            name: { type: 'string' },
            email: { type: 'string' },
            role: { type: 'string' },
          },
        },
      },
    },
  }, async (request, reply) => {
    // request.body is already validated and typed — no middleware needed
    const user = await createUser(request.body);
    reply.code(201);
    return user;
    // Even if user has passwordHash, the response schema strips it
    // Only id, name, email, role are serialized
  });
});
```

**How Fastify's serialization avoids JSON.stringify overhead:**

At startup, Fastify takes the response schema and compiles it into a specialized serialization function using `fast-json-stringify`. Instead of the generic `JSON.stringify` algorithm (which must inspect each property's type, check for `toJSON()`, handle circular references, etc.), the compiled function knows the exact shape:

```typescript
// What JSON.stringify does (simplified, every call):
// 1. Is it null? undefined? a number? string? boolean? object? array?
// 2. Does it have toJSON()? Call it.
// 3. For each property: what type? Recurse. Handle special chars in strings.
// 4. Handle circular references.

// What fast-json-stringify generates at startup (pseudocode):
function compiledSerializer(obj) {
  return `{"id":${obj.id},"name":"${escapeString(obj.name)}","email":"${escapeString(obj.email)}","role":"${escapeString(obj.role)}"}`;
}
// No type checking, no property discovery, no toJSON — it knows the shape.
```

This compiled serializer is 2-3x faster than `JSON.stringify` for typical API responses. The bigger the response object, the larger the speedup.

**Performance difference summary:**

| Aspect | Express | Fastify |
|---|---|---|
| Validation | Runtime interpretation per request (Zod/Joi) | Compiled Ajv function (once at startup) |
| Serialization | `JSON.stringify` (generic, per-request) | `fast-json-stringify` (compiled, shape-aware) |
| Data leakage protection | Manual — must pick fields explicitly | Automatic — response schema strips extra fields |
| Throughput | ~15k req/s (hello world benchmark) | ~45k req/s (hello world benchmark) |
| Real-world difference | Matters at high throughput; negligible for low-traffic APIs | The gap narrows when DB calls dominate response time |

**When the performance gap matters:** For most CRUD APIs where 90% of response time is database/network I/O, the serialization difference is noise. It matters for high-throughput services (API gateways, proxies, real-time feeds) where you're serializing thousands of responses per second and the framework overhead becomes a meaningful percentage of total latency.

</details>

<details>
<summary>23. Set up a Node.js package that supports both CommonJS (require) and ES Modules (import) consumers — show the package.json exports field, the directory structure with .cjs/.mjs files or dual builds, and explain the common pitfalls (dual package hazard, conditional exports resolution)</summary>

This is the practical implementation of the dual-package concepts from Q8.

**Directory structure:**

```
my-lib/
├── src/
│   └── index.ts              # Single TypeScript source
├── dist/
│   ├── esm/
│   │   ├── index.js           # Compiled ESM (import/export)
│   │   └── index.d.ts         # Type declarations
│   └── cjs/
│       ├── index.js           # Compiled CJS (require/module.exports)
│       └── index.d.ts
├── package.json
└── tsconfig.json
```

**package.json — the exports field:**

```json
{
  "name": "my-lib",
  "version": "1.0.0",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/esm/index.d.ts",
        "default": "./dist/esm/index.js"
      },
      "require": {
        "types": "./dist/cjs/index.d.ts",
        "default": "./dist/cjs/index.js"
      }
    }
  },
  "main": "./dist/cjs/index.js",
  "module": "./dist/esm/index.js",
  "types": "./dist/esm/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "npm run build:esm && npm run build:cjs",
    "build:esm": "tsc --project tsconfig.esm.json",
    "build:cjs": "tsc --project tsconfig.cjs.json"
  }
}
```

Key points about `exports`:
- `"import"` condition matches when the consumer uses `import` or dynamic `import()`.
- `"require"` condition matches when the consumer uses `require()`.
- `"types"` must come first within each condition (TypeScript resolves top-down).
- `"main"` and `"module"` are fallbacks for older Node.js versions and bundlers that don't support `exports`.

**TypeScript configs:**

```json
// tsconfig.esm.json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2022",
    "outDir": "./dist/esm",
    "declaration": true,
    "declarationDir": "./dist/esm"
  },
  "include": ["src"]
}

// tsconfig.cjs.json
{
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "target": "ES2022",
    "outDir": "./dist/cjs",
    "declaration": true,
    "declarationDir": "./dist/cjs"
  },
  "include": ["src"]
}
```

**Critical: mark each dist folder's module type** with a package.json stub:

```json
// dist/esm/package.json
{ "type": "module" }

// dist/cjs/package.json
{ "type": "commonjs" }
```

Without these, Node.js uses the root `package.json`'s `"type"` field for both directories, and one format will fail to load.

**The common pitfalls:**

**1. Dual package hazard (covered conceptually in Q8):**

If both ESM and CJS entry points are loaded in the same application (directly or via transitive dependencies), Node.js creates two separate module instances. Any module-level state exists twice:

```typescript
// src/index.ts
const cache = new Map(); // singleton pattern

export function getCacheSize() { return cache.size; }
export function cacheSet(k: string, v: string) { cache.set(k, v); }
```

If one dependency `require()`s your library and another `import`s it, they get separate `cache` Maps. Calling `cacheSet` from one doesn't affect `getCacheSize` from the other.

**Mitigation — the wrapper pattern:** Ship one format as the "real" implementation, the other as a thin wrapper:

```javascript
// dist/cjs/index.js — CJS wrapper that delegates to ESM
module.exports = async function loadLib() {
  return import('../esm/index.js');
};
// Or for synchronous compat, re-export only stateless functions
```

A simpler approach used by many libraries: ship only ESM and let consumers handle CJS compatibility, since modern tooling (bundlers, Node.js 22+) handles ESM well.

**2. Conditional exports resolution order:**

Node.js evaluates conditions top-to-bottom and uses the first match. Order matters:

```json
{
  "exports": {
    ".": {
      "types": "...",    // Must be first for TypeScript
      "import": "...",
      "require": "...",
      "default": "..."   // Fallback — must be last
    }
  }
}
```

**3. Subpath exports:**

If your package has multiple entry points, each needs explicit exports:

```json
{
  "exports": {
    ".": { "import": "./dist/esm/index.js", "require": "./dist/cjs/index.js" },
    "./utils": { "import": "./dist/esm/utils.js", "require": "./dist/cjs/utils.js" }
  }
}
```

Without the `"./utils"` entry, `import { something } from 'my-lib/utils'` throws `ERR_PACKAGE_PATH_NOT_EXPORTED`, even if the file exists. The `exports` field acts as an allowlist.

</details>

<details>
<summary>24. Show how to use Node.js core APIs in a realistic scenario — read a file with fs/promises, hash its contents with crypto, resolve paths cross-platform with path.join/path.resolve, and work with the raw binary data using Buffer. Explain why Buffer exists (Node.js handles binary data that JavaScript strings can't represent), when you'd use util.promisify vs the built-in promise APIs, and what common mistakes developers make with path manipulation across operating systems.</summary>

**Realistic scenario — file integrity checker:**

```typescript
import { readFile, readdir, stat } from 'node:fs/promises';
import { createHash } from 'node:crypto';
import path from 'node:path';

interface FileChecksum {
  filePath: string;
  sha256: string;
  sizeBytes: number;
}

async function checksumDirectory(dirPath: string): Promise<FileChecksum[]> {
  // path.resolve converts relative paths to absolute using cwd
  const absoluteDir = path.resolve(dirPath);
  const entries = await readdir(absoluteDir);

  const results: FileChecksum[] = [];

  for (const entry of entries) {
    // path.join handles OS separators — / on Unix, \ on Windows
    const filePath = path.join(absoluteDir, entry);
    const fileStat = await stat(filePath);

    if (!fileStat.isFile()) continue;

    // readFile returns a Buffer by default (no encoding argument)
    const buffer: Buffer = await readFile(filePath);

    // crypto works directly with Buffers — no string conversion needed
    const hash = createHash('sha256').update(buffer).digest('hex');

    results.push({
      filePath: path.relative(process.cwd(), filePath), // relative for display
      sha256: hash,
      sizeBytes: buffer.byteLength,
    });
  }

  return results;
}

// Working with Buffer directly
function inspectBinaryFile(filePath: string): void {
  const buf = Buffer.from([0x89, 0x50, 0x4e, 0x47]); // PNG magic bytes

  // Buffer gives you byte-level access
  console.log(buf[0]); // 137 (0x89)
  console.log(buf.toString('hex')); // '89504e47'
  console.log(buf.subarray(0, 2)); // Buffer containing first 2 bytes

  // Allocate fixed-size buffer (e.g., for protocol headers)
  const header = Buffer.alloc(12); // 12 zero-filled bytes
  header.writeUInt32BE(0x01, 0);   // write version at offset 0
  header.writeUInt32BE(1024, 4);   // write payload length at offset 4
  header.writeUInt32BE(0x02, 8);   // write message type at offset 8
}
```

**Why Buffer exists:**

JavaScript strings are UTF-16 encoded — they represent text. But Node.js handles raw binary data constantly: file contents, network packets, cryptographic output, image data, protocol headers. You can't represent a JPEG file as a JavaScript string without corruption (invalid UTF-16 sequences get mangled). Buffer is a fixed-size chunk of raw memory outside the V8 heap that gives you direct byte-level access.

Key differences from strings:
- Buffer is mutable; strings are immutable
- Buffer has a fixed length; strings can be concatenated freely
- Buffer stores raw bytes; strings store UTF-16 code units
- Buffer can represent any binary data; strings only represent valid text

**util.promisify vs built-in promise APIs:**

```typescript
import { readFile } from 'node:fs';            // callback-based
import { readFile as readFileAsync } from 'node:fs/promises'; // built-in promise
import { promisify } from 'node:util';

// Modern approach — use the built-in promise API directly
const content = await readFileAsync('/tmp/file.txt', 'utf8');

// util.promisify — for older APIs that only have callback versions
import { exec } from 'node:child_process';
const execAsync = promisify(exec);
const { stdout } = await execAsync('ls -la');
```

**When to use which:**
- **Built-in promise APIs** (`fs/promises`, `dns/promises`, `timers/promises`): Always prefer these when available. They're native, well-typed, and maintained by the Node.js team.
- **util.promisify**: Only for callback-based APIs that don't have a promise variant — third-party libraries, older Node.js APIs, or custom callback functions. It works with any function following the `(err, result) => void` callback convention.

**Common path manipulation mistakes:**

```typescript
// WRONG: string concatenation — breaks on Windows
const bad = '/home/user' + '/' + 'file.txt';

// CORRECT: path.join handles OS separators
const good = path.join('/home/user', 'file.txt');
// Unix: /home/user/file.txt
// Windows: \home\user\file.txt

// WRONG: assuming cwd
const risky = path.join('config', 'app.json');
// Resolves relative to wherever the process started

// CORRECT: resolve from a known base
const safe = path.join(__dirname, 'config', 'app.json');
// Or in ESM (no __dirname):
import { fileURLToPath } from 'node:url';
const __dirname = path.dirname(fileURLToPath(import.meta.url));

// WRONG: not normalizing user input (path traversal)
const userInput = '../../etc/passwd';
const dangerous = path.join('/uploads', userInput);
// Result: /etc/passwd — path traversal attack

// CORRECT: validate the resolved path stays within bounds
const resolved = path.resolve('/uploads', userInput);
if (!resolved.startsWith('/uploads/')) {
  throw new Error('Path traversal detected');
}

// GOTCHA: path.extname behavior
path.extname('archive.tar.gz');  // '.gz' — only the last extension
path.extname('.gitignore');       // '' — dotfiles have no extension
```

</details>

## Practical — Production Patterns

<details>
<summary>25. Implement graceful shutdown for a Node.js HTTP server — show the SIGTERM/SIGINT handlers that stop accepting new connections, drain in-flight requests with a timeout, clean up resources (database connections, open file handles), and exit with the correct code. What happens to long-running requests that exceed the grace period, and how does terminationGracePeriodSeconds in Kubernetes interact with your application-level grace period?</summary>

This is the implementation of the graceful shutdown concepts from Q11.

```typescript
import http from 'node:http';

interface ShutdownOptions {
  server: http.Server;
  gracePeriodMs: number;
  cleanup: () => Promise<void>; // close DB pools, flush logs, etc.
}

// Module-level shutdown state — accessible to both setup function and request handler
let isShuttingDown = false;

function setupGracefulShutdown({ server, gracePeriodMs, cleanup }: ShutdownOptions) {
  async function shutdown(signal: string) {
    if (isShuttingDown) return; // prevent double-shutdown
    isShuttingDown = true;
    console.log(`Received ${signal} — starting graceful shutdown`);

    // 1. Stop accepting new connections
    // server.close() stops listening but lets existing connections finish
    server.close(async () => {
      console.log('All connections drained');
      try {
        // 3. Clean up resources
        await cleanup();
        console.log('Cleanup complete');
        process.exit(0);
      } catch (err) {
        console.error('Cleanup failed:', err);
        process.exit(1);
      }
    });

    // 2. Force-close connections that are idle (keep-alive with no active request)
    // Prevents server.close() from hanging on keep-alive connections
    server.closeIdleConnections();

    // 4. Hard timeout — kill the process if draining takes too long
    setTimeout(() => {
      console.error(`Graceful shutdown timed out after ${gracePeriodMs}ms — forcing exit`);
      process.exit(1);
    }, gracePeriodMs).unref(); // .unref() so this timer doesn't keep the process alive
  }

  // Handle both signals identically
  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
}

// Usage
const server = http.createServer((req, res) => {
  // Optionally reject new requests during shutdown
  if (isShuttingDown) {
    res.writeHead(503, { 'connection': 'close' });
    res.end('Service shutting down');
    return;
  }
  // ... handle request
});

setupGracefulShutdown({
  server,
  gracePeriodMs: 25_000, // 25 seconds — see K8s interaction below
  async cleanup() {
    await dbPool.end();          // close database connections
    await redisClient.quit();    // close Redis connection
    await logger.flush();        // flush buffered logs
  },
});

server.listen(3000);
```

**What happens to long-running requests that exceed the grace period:**

`server.close()` waits for all active connections to finish. If a request takes longer than the grace period (the `setTimeout` above), the `process.exit(1)` kills the process mid-response. Those clients get a TCP RST (connection reset) — the response is lost.

To handle this gracefully, you can track active requests and forcefully close them before the hard timeout:

```typescript
// Track connections for forced shutdown
const connections = new Set<import('net').Socket>();
server.on('connection', (conn) => {
  connections.add(conn);
  conn.on('close', () => connections.delete(conn));
});

// In the timeout handler, destroy remaining connections
setTimeout(() => {
  console.error('Force-closing remaining connections');
  for (const conn of connections) {
    conn.destroy(); // immediate TCP RST
  }
  process.exit(1);
}, gracePeriodMs).unref();
```

**Kubernetes terminationGracePeriodSeconds interaction:**

```
K8s sends SIGTERM
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  terminationGracePeriodSeconds (default: 30s)       │
│                                                      │
│  Your app's grace period must fit INSIDE this window │
│  ┌──────────────────────────────────┐               │
│  │  App gracePeriodMs (e.g., 25s)   │               │
│  └──────────────────────────────────┘               │
│                                        5s buffer     │
└─────────────────────────────────────────────────────┘
        │
        ▼ (if still running)
   K8s sends SIGKILL — process is killed immediately, no handler possible
```

**Key rules:**
- Set your app's grace period to `terminationGracePeriodSeconds - 5s` (buffer for K8s overhead and cleanup).
- If `terminationGracePeriodSeconds` is 30s (default), your app should aim to exit within 25s.
- If your app needs more time (e.g., long-running WebSocket connections), increase `terminationGracePeriodSeconds` in the pod spec — don't try to fight SIGKILL.
- SIGKILL cannot be caught or handled — the process dies instantly. Any in-flight work is lost.

**Additional detail:** K8s also removes the pod from the Service endpoints when it sends SIGTERM. But there's a propagation delay — new traffic may still arrive for a few seconds. Adding a small `sleep` at the start of your shutdown handler (or using a `preStop` hook) ensures the pod stops receiving traffic before it stops accepting connections:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

</details>

<details>
<summary>26. Implement a production error handling strategy that distinguishes between operational errors (network timeout, file not found, validation failure) and programmer errors (TypeError, null reference) — show how to structure try/catch boundaries, when to let the process crash vs recover, and how to handle unhandledRejection with proper logging</summary>

This is the implementation of the error philosophy from Q12.

**Step 1 — Define an error hierarchy:**

```typescript
// errors.ts — typed operational errors
export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly isOperational: boolean = true,
    public readonly code?: string,
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} '${id}' not found`, 404, true, 'NOT_FOUND');
  }
}

export class ValidationError extends AppError {
  constructor(message: string, public readonly fields?: Record<string, string>) {
    super(message, 400, true, 'VALIDATION_ERROR');
  }
}

export class ServiceUnavailableError extends AppError {
  constructor(service: string) {
    super(`${service} is temporarily unavailable`, 503, true, 'SERVICE_UNAVAILABLE');
  }
}
```

**Step 2 — Structure try/catch at the right boundaries:**

```typescript
// user-service.ts — service layer catches and translates
import { NotFoundError, ServiceUnavailableError } from './errors.js';

export class UserService {
  async findById(id: string) {
    try {
      const user = await this.db.query('SELECT * FROM users WHERE id = $1', [id]);
      if (!user) throw new NotFoundError('User', id);
      return user;
    } catch (err) {
      // Operational: DB connection refused → wrap in a meaningful error
      if ((err as NodeJS.ErrnoException).code === 'ECONNREFUSED') {
        throw new ServiceUnavailableError('database');
      }
      // If it's already an AppError, re-throw as-is
      if (err instanceof AppError) throw err;
      // Unknown error — don't wrap it, let it bubble as a programmer error
      throw err;
    }
  }
}
```

**Step 3 — Top-level error handler (Express example):**

```typescript
// error-handler.ts — the final catch-all
import { AppError } from './errors.js';
import { logger } from './logger.js';
import type { Request, Response, NextFunction } from 'express';

export function errorHandler(err: Error, req: Request, res: Response, _next: NextFunction) {
  // Operational error — return a structured response, do NOT crash
  if (err instanceof AppError && err.isOperational) {
    logger.warn('Operational error', {
      code: err.code,
      message: err.message,
      statusCode: err.statusCode,
      path: req.path,
    });

    return res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        ...(err instanceof ValidationError && err.fields ? { fields: err.fields } : {}),
      },
    });
  }

  // Programmer error — log as fatal, return generic 500
  // Do NOT expose internal error details to the client
  logger.fatal('Programmer error in request handler', {
    error: err.message,
    stack: err.stack,
    path: req.path,
  });

  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
    },
  });

  // In strict "let it crash" mode, you'd exit here:
  // process.exit(1);
  // But most teams accept the risk of continuing for non-fatal programmer errors
  // in HTTP handlers, since each request is independent.
}

// Register as the last middleware
app.use(errorHandler);
```

**Step 4 — Global handlers for unhandled errors:**

```typescript
// process-handlers.ts — last line of defense
import { logger } from './logger.js';

// Unhandled rejection = missing .catch() = programmer error
process.on('unhandledRejection', (reason: unknown) => {
  logger.fatal('Unhandled promise rejection', {
    reason: reason instanceof Error ? reason.message : String(reason),
    stack: reason instanceof Error ? reason.stack : undefined,
  });
  // Crash — state is unknown. Orchestrator will restart.
  process.exit(1);
});

// Uncaught exception = synchronous throw nobody caught = programmer error
process.on('uncaughtException', (err: Error) => {
  logger.fatal('Uncaught exception', {
    error: err.message,
    stack: err.stack,
  });
  // MUST exit — process is in undefined state (as covered in Q11)
  process.exit(1);
});
```

**The decision framework:**

```
Error occurs
  │
  ├── Is it an AppError with isOperational=true?
  │     YES → Log at warn/error level, return meaningful HTTP response, continue
  │
  ├── Is it a known external failure (ECONNREFUSED, ETIMEDOUT)?
  │     YES → Wrap in operational error, apply retry/circuit breaker
  │
  └── Unknown error (TypeError, ReferenceError, unexpected state)
        → Log at fatal level, return generic 500
        → In strict mode: exit process
        → In pragmatic mode: continue if error was in isolated request handler
```

**Key principle:** Never swallow errors silently. Every error is either explicitly handled (operational) or explicitly crashes the process (programmer). The worst pattern is `catch (err) { /* ignore */ }` — it hides bugs and leads to silent data corruption.

</details>

<details>
<summary>27. Given a code snippet that mixes setTimeout, setImmediate, process.nextTick(), Promise.then(), and queueMicrotask() — predict the exact execution order, explain why each callback runs when it does, and show how wrapping the same code inside an I/O callback (e.g., fs.readFile) changes the order</summary>

This applies the event loop theory from Q3 and Q14 to a concrete prediction exercise.

**The snippet:**

```typescript
console.log('1: script start');

setTimeout(() => console.log('2: setTimeout'), 0);

setImmediate(() => console.log('3: setImmediate'));

Promise.resolve().then(() => console.log('4: promise.then'));

queueMicrotask(() => console.log('5: queueMicrotask'));

process.nextTick(() => console.log('6: nextTick'));

console.log('7: script end');
```

**Output (in main module):**

```
1: script start
7: script end
6: nextTick
4: promise.then
5: queueMicrotask
2: setTimeout      ← OR 3: setImmediate (non-deterministic)
3: setImmediate    ← OR 2: setTimeout
```

**Step-by-step explanation:**

1. `"1: script start"` — synchronous, runs immediately.
2. `setTimeout` — schedules callback in the **timers** phase (next event loop iteration).
3. `setImmediate` — schedules callback in the **check** phase.
4. `Promise.resolve().then()` — queues callback in the **microtask queue**.
5. `queueMicrotask()` — also queues in the **microtask queue** (same queue as Promise.then).
6. `process.nextTick()` — queues in the **nextTick queue** (separate from microtasks, higher priority).
7. `"7: script end"` — synchronous, runs immediately.

After all synchronous code completes, before the event loop enters any phase:
- **nextTick queue drains first** → `"6: nextTick"`
- **Microtask queue drains next** → `"4: promise.then"`, then `"5: queueMicrotask"` (FIFO order within the queue)

Then the event loop starts its phases:
- **timers phase**: Is the 1ms timeout ready? Maybe — depends on how fast we got here.
- **check phase**: setImmediate callback is ready.

The order of setTimeout vs setImmediate is **non-deterministic** in the main module because `setTimeout(fn, 0)` is internally `setTimeout(fn, 1)`. If the event loop reaches the timers phase before 1ms has elapsed, the timer isn't ready — so it proceeds to check (setImmediate fires first). If 1ms has passed, setTimeout fires first.

**Now wrap the same code inside an I/O callback:**

```typescript
import { readFile } from 'node:fs';

readFile(__filename, () => {
  console.log('1: I/O callback start');

  setTimeout(() => console.log('2: setTimeout'), 0);

  setImmediate(() => console.log('3: setImmediate'));

  Promise.resolve().then(() => console.log('4: promise.then'));

  queueMicrotask(() => console.log('5: queueMicrotask'));

  process.nextTick(() => console.log('6: nextTick'));

  console.log('7: I/O callback end');
});
```

**Output (inside I/O callback — deterministic):**

```
1: I/O callback start
7: I/O callback end
6: nextTick
4: promise.then
5: queueMicrotask
3: setImmediate      ← ALWAYS before setTimeout
2: setTimeout        ← ALWAYS after setImmediate
```

**Why the order is now deterministic:** The I/O callback runs in the **poll phase**. After it completes:
- nextTick and microtask queues drain (same as before).
- The event loop moves to the **check phase** (next after poll) → `setImmediate` fires.
- Then it wraps around to the **timers phase** → `setTimeout` fires.

Since check always comes after poll and before the next timers phase, `setImmediate` always beats `setTimeout` when scheduled from inside an I/O callback.

**Bonus — nested nextTick vs Promise ordering:**

```typescript
process.nextTick(() => {
  console.log('nextTick 1');
  Promise.resolve().then(() => console.log('promise inside nextTick'));
  process.nextTick(() => console.log('nextTick 2'));
});

// Output:
// nextTick 1
// nextTick 2      ← nextTick queue drains fully before microtasks
// promise inside nextTick
```

The nextTick queue is drained completely (including new entries added during draining) before the microtask queue gets a turn. This is why recursive nextTick can starve microtasks (as covered in Q14).

</details>

<details>
<summary>28. You're running a Node.js app that makes many DNS lookups and file system calls, and you notice requests stalling under load — demonstrate how to diagnose that the default libuv thread pool (size 4) is the bottleneck, show how to set UV_THREADPOOL_SIZE correctly, and explain why setting it too high is also harmful</summary>

This is the practical debugging scenario for the thread pool concepts from Q4.

**Symptoms that point to thread pool saturation:**

- Request latency spikes under load, but CPU usage stays low (the main thread is idle, waiting for thread pool work to complete).
- Event loop lag is low (the event loop itself isn't blocked — it's just waiting for callbacks from the thread pool).
- Latency is worst on endpoints that do DNS lookups + file I/O, not on endpoints that only do network I/O (since network I/O doesn't use the thread pool).
- Adding more instances helps linearly (each instance gets its own pool of 4 threads).

**Diagnosing the bottleneck:**

```typescript
// 1. Measure DNS lookup time vs network time
import { performance } from 'node:perf_hooks';
import dns from 'node:dns';

const start = performance.now();
dns.lookup('api.example.com', (err, address) => {
  const lookupMs = performance.now() - start;
  console.log(`DNS lookup took ${lookupMs}ms`); // If >50ms under load, pool is saturated
});

// 2. Monitor thread pool queue depth with UV_THREADPOOL_SIZE diagnostic
// There's no direct API for queue depth, but you can infer it:

// Simple probe: schedule a trivial fs operation and measure how long
// it takes to START executing (not complete — just start)
import { stat } from 'node:fs';

function measureThreadPoolLatency(): Promise<number> {
  const start = performance.now();
  return new Promise((resolve) => {
    // stat on a tiny file is fast — any delay is from waiting for a thread
    stat('/dev/null', () => resolve(performance.now() - start));
  });
}

// Expose as a Prometheus metric
setInterval(async () => {
  const latencyMs = await measureThreadPoolLatency();
  threadPoolLatencyGauge.set(latencyMs);
  if (latencyMs > 10) {
    console.warn(`Thread pool latency: ${latencyMs.toFixed(1)}ms — pool may be saturated`);
  }
}, 5000);

// 3. Use 'active-handles' to see what's queued
// In a diagnostic endpoint:
console.log('Active handles:', process._getActiveHandles().length);
console.log('Active requests:', process._getActiveRequests().length);
// High numbers suggest many pending thread pool operations
```

**Proving it's the thread pool — the definitive test:**

```bash
# Run your load test with default pool size
UV_THREADPOOL_SIZE=4 node server.js
# Measure p99 latency under load: e.g., 500ms

# Increase pool size and re-run identical load test
UV_THREADPOOL_SIZE=16 node server.js
# Measure p99 latency: e.g., 50ms

# If latency drops significantly, the thread pool was the bottleneck
```

**Setting UV_THREADPOOL_SIZE correctly:**

```bash
# Must be set as env var BEFORE process starts — read once, cannot change at runtime
UV_THREADPOOL_SIZE=16 node server.js
```

**Sizing heuristic:**

```
UV_THREADPOOL_SIZE = number of concurrent thread-pool operations you expect

Typical operations per request that use the thread pool:
- dns.lookup()     → 1 thread
- fs.readFile()    → 1 thread
- crypto.pbkdf2()  → 1 thread
- zlib.inflate()   → 1 thread

If each request does dns.lookup + fs.readFile = 2 thread pool ops,
and you want to handle 8 concurrent requests without queuing:
UV_THREADPOOL_SIZE = 8 × 2 = 16
```

Common production values: 8-64, depending on workload. Most apps do well with 16-32.

**Why setting it too high is harmful:**

1. **Memory overhead:** Each thread has a stack (default ~8MB on Linux, though only pages actually used are allocated). 128 threads × 8MB = 1GB of virtual memory reserved for stacks alone.

2. **Context switching:** With more threads than CPU cores, the OS spends time switching between threads rather than doing work. Beyond ~2x CPU cores, you get diminishing returns and increasing overhead.

3. **Thundering herd on startup:** If you set `UV_THREADPOOL_SIZE=1024` and your app does a burst of file operations at startup, you spawn 1024 threads simultaneously — the OS scheduler struggles and startup time increases.

4. **Hides the real problem:** A very large thread pool masks architectural issues. If every request needs 3 thread pool operations, the real fix is to reduce thread pool usage:
   - Replace `dns.lookup()` with `dns.resolve()` (uses c-ares, not the thread pool)
   - Cache DNS results
   - Use an HTTP client that reuses connections (avoids repeated DNS lookups)
   - Buffer file reads in memory if the data is small and read frequently

**Rule of thumb:** Start with `UV_THREADPOOL_SIZE` equal to the number of CPU cores × 2. Load test. If thread pool latency is still high, increase gradually. If it's already low, don't increase — you're paying memory overhead for nothing.

</details>

<details>
<summary>29. Create a custom EventEmitter subclass that properly handles the 'error' event, sets appropriate maxListeners, and prevents memory leaks from unremoved listeners — show what happens when you emit 'error' without a listener, how to debug the "MaxListenersExceededWarning," and the pattern for cleanup in long-lived emitters</summary>

**Custom EventEmitter subclass with proper error handling:**

```typescript
import { EventEmitter } from 'node:events';

interface JobEvents {
  progress: [percent: number];
  complete: [result: unknown];
  error: [error: Error];
}

class JobProcessor extends EventEmitter {
  private cleanupFns: Array<() => void> = [];

  constructor() {
    super();

    // Set maxListeners based on expected usage
    // Default is 10 — increase only if you know you need more
    this.setMaxListeners(20);

    // ALWAYS have a default error handler to prevent process crash
    this.on('error', (err: Error) => {
      console.error(`[JobProcessor] Unhandled error: ${err.message}`);
    });
  }

  async process(data: unknown) {
    try {
      this.emit('progress', 0);
      const result = await this.doWork(data);
      this.emit('progress', 100);
      this.emit('complete', result);
    } catch (err) {
      this.emit('error', err instanceof Error ? err : new Error(String(err)));
    }
  }

  // Pattern for registering listeners with automatic cleanup
  onWithCleanup(event: string, listener: (...args: unknown[]) => void): () => void {
    this.on(event, listener);
    const cleanup = () => this.removeListener(event, listener);
    this.cleanupFns.push(cleanup);
    return cleanup; // caller can also trigger cleanup manually
  }

  destroy() {
    // Remove all registered listeners
    for (const fn of this.cleanupFns) fn();
    this.cleanupFns = [];
    this.removeAllListeners();
  }

  private async doWork(data: unknown): Promise<unknown> {
    // simulate work
    return { processed: true };
  }
}
```

**What happens when you emit 'error' without a listener:**

```typescript
const emitter = new EventEmitter();

// This CRASHES the process with an uncaught exception
emitter.emit('error', new Error('something failed'));
// Error: something failed
//     at ...
// The process exits with code 1
```

This is by design in Node.js — the `'error'` event is special. If emitted with no listeners, Node.js treats it as an unhandled exception and throws it. This forces developers to handle errors explicitly rather than silently swallowing them.

**Prevention strategies:**

```typescript
// Option 1: Always register an error listener
emitter.on('error', (err) => { /* handle it */ });

// Option 2: Use events.errorMonitor (Node.js 13+) for observability
// without affecting the throw behavior
import { errorMonitor } from 'node:events';
emitter.on(errorMonitor, (err) => {
  logger.error('Error observed', { error: err.message });
  // This doesn't prevent the throw — it's for monitoring only
});
```

**Debugging "MaxListenersExceededWarning":**

This warning fires when more than `maxListeners` (default 10) listeners are added to a single event. It usually indicates a memory leak — listeners being added repeatedly without removal.

```typescript
// LEAK: adding a listener on every request, never removing
server.on('request', (req, res) => {
  const handler = () => console.log('db error');
  database.on('error', handler);
  // handler is never removed — accumulates with each request
});

// After 11 requests:
// MaxListenersExceededWarning: Possible EventEmitter memory leak detected.
// 11 error listeners added to [Database].
// Use emitter.setMaxListeners() to increase limit
```

**Debugging steps:**

```typescript
// 1. Find what's adding listeners — use the trace
process.on('warning', (warning) => {
  console.warn(warning.name);    // MaxListenersExceededWarning
  console.warn(warning.message); // tells you which event and emitter
  console.warn(warning.stack);   // stack trace showing WHERE listeners are added
});

// 2. Or run with the trace-warnings flag
// node --trace-warnings server.js

// 3. Inspect listener counts programmatically
console.log(emitter.listenerCount('error'));  // how many?
console.log(emitter.listeners('error'));       // which functions?
console.log(emitter.eventNames());             // which events have listeners?
```

**Cleanup patterns for long-lived emitters:**

```typescript
// Pattern 1: Use AbortController for scoped listener lifetime
import { addAbortListener } from 'node:events';

function handleRequest(req: Request, dbEmitter: EventEmitter) {
  const ac = new AbortController();

  const handler = (err: Error) => {
    console.error('DB error during request:', err);
  };
  dbEmitter.on('error', handler);

  // addAbortListener safely removes the listener when ac.abort() is called
  addAbortListener(ac.signal, () => dbEmitter.removeListener('error', handler));

  // After request completes, clean up
  req.on('close', () => ac.abort());
}

// Pattern 2: Use 'once' for one-shot listeners
emitter.once('ready', () => {
  // Automatically removed after first invocation — no leak possible
});

// Pattern 3: Track and clean up manually
class RequestHandler {
  private listeners: Array<{ emitter: EventEmitter; event: string; fn: Function }> = [];

  addListener(emitter: EventEmitter, event: string, fn: (...args: unknown[]) => void) {
    emitter.on(event, fn);
    this.listeners.push({ emitter, event, fn });
  }

  cleanup() {
    for (const { emitter, event, fn } of this.listeners) {
      emitter.removeListener(event, fn as any);
    }
    this.listeners = [];
  }
}
```

**When to increase maxListeners vs fix the leak:**

- **Increase** when you legitimately have many listeners (e.g., a pub/sub hub with many subscribers, a connection pool with many health monitors).
- **Fix the leak** when listeners are added per-request or per-event without cleanup — this is always a bug, not a legitimate use case. Use `setMaxListeners(0)` (unlimited) only if you've verified it's not a leak.

</details>

## Practical — Debugging & Profiling

<details>
<summary>30. Walk through diagnosing a memory leak in a Node.js application — show how to take heap snapshots with --inspect and Chrome DevTools (or the heapdump module), how to compare two snapshots to find retained objects, how to identify common culprits (growing arrays, closures, event listeners), how process.memoryUsage() and v8.getHeapStatistics() help you monitor memory programmatically, and what --max-old-space-size controls</summary>

This is the practical diagnostic workflow for the GC concepts from Q7 and the diagnostics overview from Q13.

**Step 1 — Confirm the leak with programmatic monitoring:**

```typescript
import v8 from 'node:v8';

// Log memory usage periodically
setInterval(() => {
  const mem = process.memoryUsage();
  const heap = v8.getHeapStatistics();

  console.log({
    // process.memoryUsage() — what to watch
    heapUsedMB: (mem.heapUsed / 1024 / 1024).toFixed(1),   // JS heap in use
    heapTotalMB: (mem.heapTotal / 1024 / 1024).toFixed(1),  // heap allocated by V8
    rssMB: (mem.rss / 1024 / 1024).toFixed(1),              // total process memory
    externalMB: (mem.external / 1024 / 1024).toFixed(1),    // C++ objects (Buffers)

    // v8.getHeapStatistics() — deeper detail
    heapSizeLimitMB: (heap.heap_size_limit / 1024 / 1024).toFixed(1),
    usedHeapPct: ((heap.used_heap_size / heap.heap_size_limit) * 100).toFixed(1) + '%',
  });
}, 30_000);

// Key indicator of a leak:
// heapUsedMB grows monotonically over hours, never coming back down after GC
// Healthy apps show a sawtooth pattern (grow → GC → drop → grow → GC → drop)
```

**Step 2 — Take heap snapshots:**

```bash
# Start the app with --inspect
node --inspect server.js
# Output: Debugger listening on ws://127.0.0.1:9229/...
```

Open `chrome://inspect` in Chrome → click "inspect" on your Node.js process → go to the Memory tab.

**Snapshot workflow:**
1. Take **Snapshot 1** after the app has been running and handling requests for a few minutes (let the initial allocations settle).
2. Trigger the workload you suspect is leaking (run a batch of requests).
3. Force GC: click the trash can icon in DevTools (or send `global.gc()` if running with `--expose-gc`).
4. Take **Snapshot 2** a few minutes later.
5. Take **Snapshot 3** after another round of load + GC.

**Alternatively, trigger snapshots programmatically:**

```typescript
import v8 from 'node:v8';
import { writeFileSync } from 'node:fs';

// Expose an endpoint to take snapshots
app.get('/debug/heapdump', (req, res) => {
  const snapshotPath = `/tmp/heap-${Date.now()}.heapsnapshot`;
  const stream = v8.writeHeapSnapshot(snapshotPath);
  res.json({ path: snapshotPath });
});

// Load the .heapsnapshot files in Chrome DevTools Memory tab
```

**Step 3 — Compare snapshots to find the leak:**

In Chrome DevTools Memory tab:
1. Select Snapshot 3, change the view dropdown from "Summary" to **"Comparison"**.
2. Compare against Snapshot 1.
3. Sort by **"# Delta"** (objects added) or **"Size Delta"** (memory added).
4. Look for object types with a **large positive delta** — these are objects that were allocated but never freed.

**What to look for:**

| Culprit | What you see in the snapshot | Common cause |
|---|---|---|
| **Growing arrays/Maps** | `(array)` or `Map` with large delta in count and size | Module-level cache without eviction (Q7 pattern #3) |
| **Closures** | `(closure)` entries growing, retaining large scopes | Callbacks registered per request but never removed |
| **Event listeners** | `(closure)` held by `EventEmitter` entries | `.on()` in request handlers without `.removeListener()` |
| **Strings** | `(string)` with large retained size | Log buffers, template accumulation, string concatenation |
| **Detached DOM trees** | N/A for Node.js, but analogous: objects referenced only by a forgotten collection | Objects in a Set/Map that should have been removed |

**Step 4 — Drill into retainers:**

Click on a suspicious object → look at the **"Retainers"** panel at the bottom. This shows the chain of references keeping it alive. Follow the chain to find what's holding the reference:

```
Object @12345
  └── retained by: entries (property) of Map @67890
        └── retained by: cache (variable) in someModule.js:15
```

This tells you: a `Map` called `cache` in `someModule.js` line 15 is holding references to objects that should have been collected.

**--max-old-space-size:**

Controls the maximum size of V8's old generation heap (where long-lived objects go after surviving young generation GC, as explained in Q7).

```bash
# Default: ~1.5GB on 64-bit systems (varies by Node.js version)
# Increase for memory-intensive apps
node --max-old-space-size=4096 server.js  # 4GB

# Decrease for container environments with strict memory limits
node --max-old-space-size=512 server.js   # 512MB
```

**Important:** Set this to ~75% of your container's memory limit. If your container has 1GB, set `--max-old-space-size=768`. This leaves room for:
- V8's young generation and code space
- Native memory (Buffers, C++ objects)
- OS overhead

Without this, the default 1.5GB exceeds a 1GB container and the OOM killer terminates the process before V8's GC gets a chance to reclaim memory — you get a mysterious `SIGKILL` instead of a JavaScript `heap out of memory` error.

**Quick checklist for memory leak diagnosis:**

1. Is `heapUsed` growing monotonically? → Likely a JS leak
2. Is `rss` growing but `heapUsed` stable? → Likely a native/Buffer leak
3. Is `external` growing? → Buffer or native addon leak
4. Take 3 snapshots, compare 1 vs 3 → find objects with growing count
5. Check retainers → find the root reference keeping objects alive
6. Common fixes: add eviction to caches, use `WeakMap`/`WeakRef` for caches that should allow GC, remove listeners on cleanup, use `once()` instead of `on()` for one-shot listeners

</details>

<details>
<summary>31. A Node.js API has intermittent slow responses — walk through generating a CPU profile (--prof or Chrome DevTools), reading a flame graph to identify hot functions, and determining whether the bottleneck is synchronous code blocking the event loop, excessive GC pauses, or something else entirely</summary>

This builds on the diagnostics overview from Q13 with a step-by-step profiling walkthrough.

**Step 1 — Confirm the symptom and narrow the scope:**

Before profiling, check whether slowness is across all endpoints or specific ones. If specific, it's likely a slow dependency (database, downstream service) — use distributed tracing. If all endpoints are slow intermittently, the event loop is likely blocked or GC is pausing execution.

```bash
# Quick check: is event loop lag high?
# Add this to your app or check your metrics dashboard
```

```typescript
import { monitorEventLoopDelay } from 'node:perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  console.log(`EL p99: ${(h.percentile(99) / 1e6).toFixed(1)}ms`);
  h.reset();
}, 5000);
// Healthy: <10ms. If >50ms, something is blocking.
```

**Step 2 — Generate a CPU profile:**

**Option A: Chrome DevTools (interactive, best for development):**

```bash
node --inspect server.js
# Open chrome://inspect → click "inspect" → go to "Profiler" tab
# Click "Start", send requests to reproduce the slowness, click "Stop"
# DevTools shows a flame chart with timing per function
```

**Option B: --prof (production-safe, low overhead):**

```bash
# Generate a V8 profiling log
node --prof server.js
# Send load, then stop the process
# Process the log into human-readable format
node --prof-process isolate-0x*.log > profile.txt
```

The processed output shows a breakdown by category (JavaScript, C++, GC, etc.) and a ranked list of functions by "ticks" (samples where the CPU was in that function).

**Option C: 0x for flame graphs (best visualization):**

```bash
npx 0x server.js
# Send load, then Ctrl+C to stop
# Opens an interactive flame graph in your browser
```

**Step 3 — Read the flame graph:**

A flame graph shows the call stack on the Y-axis and time (or samples) on the X-axis. Key reading techniques:

- **Wide bars at the top** = functions that consume the most CPU directly. These are your hot functions.
- **Wide bars in the middle** = functions that call expensive sub-functions. Trace down to find the actual bottleneck.
- **Plateaus** (wide flat sections) = synchronous operations holding the CPU for extended periods.
- **Color coding** in 0x: red = hot (many samples), blue = cold. Look for red bars.

```
// Example flame graph (simplified ASCII):
// Width = time spent

|============= JSON.stringify ===============|     ← hot! 40% of samples
|===== serializeUser =====|== formatDate ==|
|====== handleRequest ======|
|========== httpCallback ==========|

// This tells you: JSON.stringify is the bottleneck,
// called by serializeUser inside handleRequest
```

**Step 4 — Determine the bottleneck category:**

**Synchronous code blocking the event loop:**
- Flame graph shows wide bars in your application code (not V8 internals or GC).
- Common culprits: `JSON.stringify` on large objects, regex operations, synchronous crypto, tight loops.
- Event loop lag metric confirms: high p99 correlating with the profiled hot function.
- Fix: offload to worker threads or optimize the algorithm (as covered in Q15).

**Excessive GC pauses:**
- The `--prof-process` output shows a high percentage under "GC" (>10% is concerning, >20% is a problem).
- In the flame graph, look for `v8::internal::Heap::CollectGarbage` or `Scavenge` taking significant width.
- Correlate with `--trace-gc` output:

```bash
node --trace-gc server.js
# Output: [12345:0x...] 45 ms: Scavenge 15.2 (20.0) -> 14.8 (20.0) MB, 2.1 ms
# [12345:0x...] 50 ms: Mark-sweep 250.3 (280.0) -> 200.1 (280.0) MB, 120.5 ms
#                                                                      ^^^^^^
#                                                        120ms pause — this is your problem
```

- Long Mark-Sweep-Compact pauses indicate old generation pressure — likely a memory leak or objects living longer than necessary. See Q30 for heap snapshot diagnosis.

**Something else entirely:**
- **Thread pool saturation:** Low event loop lag, low CPU, but high latency on endpoints doing DNS/file I/O. Profile won't show it because the main thread is idle. See Q28.
- **Downstream latency:** Flame graph shows the app spending most time in `await` (which doesn't appear as CPU time). Use distributed tracing instead.
- **OS-level issues:** File descriptor exhaustion, network congestion, container CPU throttling. Check with `lsof`, `dmesg`, and container metrics.

**Decision tree summary:**

```
Intermittent slow responses
├── Event loop lag high?
│   ├── YES → CPU profile → find hot function → optimize or offload
│   └── NO → Not an event loop problem
│       ├── Thread pool latency high? → UV_THREADPOOL_SIZE (Q28)
│       ├── Specific endpoints only? → Distributed tracing → slow downstream
│       └── Memory growing? → GC pauses → heap snapshots (Q30)
```

</details>

<details>
<summary>32. Your Node.js service in production starts throwing ECONNRESET errors on outbound HTTP requests and EMFILE errors when opening files — explain what each error means at the OS level, walk through the debugging steps (checking open file descriptors with lsof, ulimit settings, connection pool configuration), and show the fixes</summary>

**ECONNRESET — what it means at the OS level:**

The remote side (server, proxy, or load balancer) sent a TCP RST packet, abruptly closing the connection. Your process was mid-read/write on a socket that no longer exists. Common causes:

- **The remote server crashed or restarted** while your request was in flight.
- **A load balancer or proxy timed out** and killed the idle connection, but your HTTP client still had it in its connection pool and tried to reuse it (stale connection).
- **Keep-alive mismatch:** Your client holds connections open longer than the server's keep-alive timeout. The server closes the connection, your client sends a request on it, and gets RST.
- **The remote server hit its connection limit** and is dropping connections.

**Debugging ECONNRESET:**

```bash
# 1. Check if it's specific to one downstream service
# Add structured logging with the target host/port

# 2. Check connection pool health
# Log pool statistics: active connections, idle connections, pending requests

# 3. Check the remote server's keep-alive settings
curl -v https://api.example.com/health 2>&1 | grep -i keep-alive
# Look for: Keep-Alive: timeout=5, max=100
# Your client's keepAlive idle timeout must be SHORTER than this
```

**Fixing ECONNRESET:**

```typescript
import { Agent } from 'node:http';

// Fix 1: Set keepAlive timeout shorter than the server's
const agent = new Agent({
  keepAlive: true,
  keepAliveMsecs: 4000,  // send keep-alive probes every 4s
  maxSockets: 50,         // limit concurrent connections per host
  maxFreeSockets: 10,     // limit idle pooled connections
  timeout: 30_000,        // socket-level timeout
});

// Fix 2: Retry on ECONNRESET (it's a transient error)
async function fetchWithRetry(url: string, retries = 3): Promise<Response> {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await fetch(url, { signal: AbortSignal.timeout(10_000) });
    } catch (err: any) {
      if (err.code === 'ECONNRESET' && attempt < retries) {
        await new Promise((r) => setTimeout(r, 100 * attempt)); // backoff
        continue;
      }
      throw err;
    }
  }
  throw new Error('Unreachable');
}
```

**EMFILE — what it means at the OS level:**

"Too many open files." Every open file, socket, pipe, and directory watcher consumes a file descriptor (fd). The OS enforces a per-process limit (`ulimit -n`). When you hit it, any call that opens an fd (`open()`, `socket()`, `connect()`) fails with EMFILE.

**Debugging EMFILE:**

```bash
# 1. Check the current limit
ulimit -n
# Typical default: 1024 (way too low for a server)

# 2. Check how many fds the process has open
lsof -p $(pgrep -f "node server.js") | wc -l
# Or inside the container:
ls /proc/$(pgrep -f "node server.js")/fd | wc -l

# 3. See WHAT is open
lsof -p $(pgrep -f "node server.js") | head -50
# Look for patterns:
# - Hundreds of TCP connections to the same host → connection pool leak
# - Hundreds of open files → file handles not being closed
# - Hundreds of pipes → child processes not being cleaned up

# 4. Check programmatically
import { opendir } from 'node:fs/promises';
const fdDir = await opendir('/proc/self/fd');
let count = 0;
for await (const _ of fdDir) count++;
console.log(`Open file descriptors: ${count}`);
```

**Common causes and fixes:**

```typescript
// Cause 1: File handles not closed after use
// BAD — opens a handle but never closes it on error
const stream = createReadStream('file.txt');
stream.on('data', process);
// If an error occurs before 'end', the fd leaks

// GOOD — use pipeline (auto-closes on error or completion, as covered in Q19)
import { pipeline } from 'node:stream/promises';
await pipeline(createReadStream('file.txt'), transformStream, writable);

// Cause 2: Too many concurrent file operations
// BAD — opens 10,000 files simultaneously
const results = await Promise.all(
  files.map((f) => readFile(f)) // 10,000 concurrent fds
);

// GOOD — limit concurrency
async function mapWithConcurrency<T, R>(
  items: T[], concurrency: number, fn: (item: T) => Promise<R>
): Promise<R[]> {
  const results: R[] = [];
  const executing = new Set<Promise<void>>();
  for (const item of items) {
    const p = fn(item).then((r) => { results.push(r); });
    executing.add(p);
    p.finally(() => executing.delete(p));
    if (executing.size >= concurrency) await Promise.race(executing);
  }
  await Promise.all(executing);
  return results;
}

await mapWithConcurrency(files, 50, (f) => readFile(f));

// Cause 3: Connection pool not limiting connections
// Fix: configure maxSockets on HTTP agents (shown above)
```

**Fixing the ulimit:**

```bash
# Temporary (current session)
ulimit -n 65536

# Permanent (Linux — in /etc/security/limits.conf)
# node    soft    nofile    65536
# node    hard    nofile    65536

# In Docker (Dockerfile or docker-compose)
# docker run --ulimit nofile=65536:65536 ...

# In Kubernetes (pod spec)
# securityContext:
#   capabilities:
#     add: ["SYS_RESOURCE"]
# Or better: set it in the container image's entrypoint
```

Increasing `ulimit` is a band-aid — it raises the ceiling but doesn't fix the leak. Always find and fix the root cause (unclosed handles, unbounded concurrency, connection pool misconfiguration) and then set a reasonable limit as a safety net.

</details>

<details>
<summary>33. Your Node.js API's p99 latency has doubled over the past week with no code changes — use clinic.js (Doctor, Flame, Bubbleprof) to diagnose the regression. Explain what each clinic.js tool measures, walk through the workflow of running clinic doctor for an initial diagnosis then drilling down with clinic flame or bubbleprof based on the findings, and how to interpret the output to identify the root cause</summary>

**The three clinic.js tools and what each measures:**

| Tool | What it measures | When to use |
|---|---|---|
| **clinic doctor** | CPU usage, memory, event loop delay, active handles — all on one dashboard | First step — triage. Identifies the category of problem. |
| **clinic flame** | CPU profiling as an interactive flame graph (built on 0x) | When doctor says "event loop blocked" or CPU is high |
| **clinic bubbleprof** | Async operation timing — visualizes time spent waiting in async operations | When doctor says "I/O issue" or latency is high but CPU is low |

**Step 1 — Run clinic doctor for triage:**

```bash
# Install globally or per-project
npm install -g clinic

# Run doctor with your server + a load test
clinic doctor -- node server.js
# In another terminal, send load:
autocannon -d 30 -c 100 http://localhost:3000/api/endpoint
# Stop the server (Ctrl+C) — doctor opens a browser dashboard
```

**Interpreting doctor's output:**

Doctor shows four charts over time: CPU %, RSS memory, event loop delay, and active handles. It also gives a text recommendation:

- **"Event loop is blocked"** → CPU chart is high, event loop delay spikes. Something synchronous is hogging the thread. → Use **clinic flame** next.
- **"I/O issue detected"** → CPU is low, event loop delay is low, but latency is high. Time is spent waiting for I/O (database, file system, network). → Use **clinic bubbleprof** next.
- **"Memory issue detected"** → RSS grows monotonically. Likely a memory leak causing GC pressure. → Use heap snapshots (Q30).
- **"Active handles growing"** → File descriptors or connections are leaking. → Check for unclosed handles (Q32).

**Step 2a — Drill down with clinic flame (event loop blocked):**

```bash
clinic flame -- node server.js
# Send load, stop, flame graph opens in browser
```

**Reading the clinic flame graph:**

- The visualization is an interactive flame chart — click to zoom into call stacks.
- **Hot (red/orange) bars at the top** are functions consuming the most CPU. Click them to see the full call stack.
- **Look for your application code**, not V8 internals. Filter by clicking the "app only" toggle.
- **Common findings for "no code changes" regressions:**
  - A dependency updated (transitive dep via lockfile changes) that introduced a slower code path.
  - Data volume increased — a `JSON.stringify` on a growing dataset, or an O(n^2) operation hitting larger n.
  - A cache expired or external service got slower, causing more fallback computation.

Example finding: `JSON.stringify` takes 35% of CPU time, called from `serializeResponse` → the response payload grew because a database query started returning more data.

**Step 2b — Drill down with clinic bubbleprof (I/O issue):**

```bash
clinic bubbleprof -- node server.js
# Send load, stop, visualization opens in browser
```

**Reading bubbleprof output:**

Bubbleprof shows async operations as bubbles. Size = time spent. Color = type (network, file, etc.). Lines between bubbles show the async flow.

- **Large bubbles** = operations taking a long time. Click to see what they are.
- **Clusters of bubbles** on the same async resource = a bottleneck (e.g., all requests waiting on the same database pool).
- **Common findings for "no code changes" regressions:**
  - Database query latency increased (larger tables, missing index, connection pool exhaustion).
  - A downstream service got slower (new version deployed, increased load).
  - DNS resolution got slower (DNS server change, cache eviction).
  - Thread pool saturation (as covered in Q28) — file/DNS operations waiting for threads.

**Step 3 — Identify the root cause for "no code changes" regression:**

If there were truly no code changes, the regression comes from the environment changing:

```
Checklist for "no code changes" latency regression:
1. Did a transitive dependency update? → Check lockfile diff
2. Did data volume grow? → Check database table sizes, response payload sizes
3. Did a downstream service change? → Check dependency service deploy history
4. Did infrastructure change? → New node type, different CPU throttling, network config
5. Did DNS/certificate changes happen? → Check DNS TTLs, cert renewal
6. Did the database get slower? → Check slow query logs, index usage, connection pool metrics
7. Did container resources change? → CPU limits, memory limits, cgroup throttling
```

The clinic.js workflow (doctor → flame or bubbleprof) narrows the search space quickly: doctor tells you *what kind* of problem it is, and flame/bubbleprof tells you *where* in the code it's happening.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>34. Tell me about a time you debugged a memory leak or performance issue in a Node.js application — what were the symptoms, what tools did you use, and what was the root cause?</summary>

**What the interviewer is looking for:**

- Systematic debugging approach, not random guessing.
- Familiarity with real diagnostic tools (heap snapshots, CPU profiles, metrics dashboards).
- Understanding of Node.js-specific causes (event loop blocking, GC pressure, thread pool saturation).
- Post-fix actions: monitoring, prevention, documentation.

**Suggested structure (STAR):**

1. **Situation:** What service, what scale, what was normal behavior.
2. **Task:** What symptom triggered the investigation (OOM kills, latency spikes, growing memory).
3. **Action:** Step-by-step debugging — what tools you used, what each told you, how you narrowed down.
4. **Result:** What the root cause was, how you fixed it, what you added to prevent recurrence.

**Example outline to personalize:**

- **Situation:** "We had a Node.js API service handling ~500 req/s. After a deploy, we noticed the pods were getting OOM-killed every 6-8 hours."
- **Action path:**
  - Checked Grafana dashboards — `heapUsed` was growing linearly, never dropping after GC (sawtooth pattern gone).
  - Connected with `--inspect` to a staging replica, took heap snapshots 5 minutes apart.
  - Compared snapshots in Chrome DevTools — found thousands of `(closure)` objects retained by a `Map` in a caching module.
  - Root cause: A new feature added a per-request cache entry keyed by user ID, but never evicted entries. With 500 req/s from unique users, the Map grew unboundedly.
- **Fix:** Added LRU eviction with a max size, or switched to Redis for caching. Added a Prometheus metric for cache size.
- **Prevention:** Added a memory growth alert (if heapUsed increases >50% over 1 hour without dropping).

**Key points to hit:**

- Show you know the difference between a JS heap leak and a native memory leak (heapUsed vs rss vs external).
- Mention the specific tool output that pointed you to the root cause — not just "I used DevTools."
- Explain why you chose that debugging approach over alternatives.
- Show the preventive measures you added.

</details>

<details>
<summary>35. Describe a time you had to choose between worker threads, child processes, or the cluster module for scaling a Node.js service — what was the workload, what did you choose and why, and what were the results?</summary>

**What the interviewer is looking for:**

- That you understand the tradeoffs between all three options (covered in Q6).
- A decision grounded in the specific workload, not a default choice.
- Awareness of the Kubernetes/container context — cluster is often unnecessary there.
- Ability to articulate why you rejected the other options.

**Suggested structure (STAR):**

1. **Situation:** What service, what workload was causing problems.
2. **Task:** Why the current approach wasn't working (blocking event loop, underutilizing cores, etc.).
3. **Action:** Which option you chose, why you picked it over the others, how you implemented it.
4. **Result:** Performance improvement, resource impact, lessons learned.

**Example outline to personalize:**

- **Situation:** "Our API service needed to generate PDF reports from templates — each report took 200-500ms of CPU time, blocking the event loop and causing p99 latency spikes for all other endpoints."
- **Options considered:**
  - **Cluster module** — rejected because we ran in Kubernetes, which already scaled horizontally. Adding cluster workers inside containers would double memory usage without solving the event loop blocking problem per process.
  - **Child process** — considered for full isolation, but the overhead of spawning a process per report (~50ms + 30-50MB) was too high at our volume (100+ reports/hour).
  - **Worker threads with piscina** — chosen because: low memory overhead (~5-10MB per thread), pre-spawned pool eliminates per-request startup cost, tasks queue automatically when all threads are busy.
- **Implementation:** Created a piscina pool with `maxThreads` set to CPU cores minus 1, moved the PDF generation logic to a worker file, main thread stays responsive.
- **Result:** p99 latency for non-PDF endpoints dropped from 800ms back to 50ms. PDF generation throughput improved because multiple reports could run in parallel.

**Key points to hit:**

- Show you evaluated all three options, not just picked the one you were familiar with.
- Mention the specific characteristics of the workload that drove the decision (CPU-bound vs I/O-bound, frequency, isolation needs).
- If you chose cluster, explain why the simpler K8s horizontal scaling wasn't sufficient.
- Mention the pool pattern if you used worker threads — spawning on demand is the common mistake.

</details>

<details>
<summary>36. Tell me about a time you had to deal with a Node.js security issue in production or during development — what was the vulnerability (dependency issue, prototype pollution, ReDoS, supply chain attack), how did you discover it, and what did you do to prevent it from recurring?</summary>

**What the interviewer is looking for:**

- Awareness of the Node.js-specific attack surface (supply chain, prototype pollution, ReDoS — covered in Q18).
- A structured incident response: discovery, assessment, remediation, prevention.
- Understanding of the difference between a vulnerability existing and it being exploitable.
- Proactive security practices, not just reactive firefighting.

**Suggested structure (STAR):**

1. **Situation:** What service, what the security context was (public-facing, internal, handling PII, etc.).
2. **Task:** How the issue was discovered (npm audit, Snyk alert, pen test, incident).
3. **Action:** How you assessed severity, what you did to remediate, timeline.
4. **Result:** What you changed in the process to prevent recurrence.

**Example outline to personalize:**

- **Situation:** "Our CI pipeline ran `npm audit` on every PR. A critical severity alert flagged a prototype pollution vulnerability in a deep transitive dependency used by our request body parser."
- **Assessment:** Checked if the vulnerability was reachable in our code — could an attacker send a crafted JSON body that would hit the vulnerable merge function? Yes, because the parser passed raw user input through the vulnerable path.
- **Remediation:**
  - Immediate: Upgraded the direct dependency that pulled in the vulnerable transitive dep. Verified the fix with `npm audit` and manual testing with a PoC payload.
  - If no fix was available: Applied input sanitization to reject `__proto__` and `constructor` keys at the middleware level as a temporary mitigation.
- **Prevention:**
  - Added `npm audit --audit-level=critical` as a blocking CI step (PR can't merge with critical vulnerabilities).
  - Set up Dependabot/Renovate for automated dependency update PRs.
  - Added a security review checklist for any dependency changes.

**Key points to hit:**

- Show you assessed exploitability, not just blindly patched. Not every CVE is actually reachable in your code.
- Mention the specific tool that caught it (npm audit, Snyk, Socket.dev, Dependabot).
- Demonstrate defense-in-depth thinking: the fix + the process improvement.
- If it was a supply chain issue, mention lockfile integrity, `npm ci`, and registry configuration.

</details>

<details>
<summary>37. Describe a time you handled a production Node.js incident — what went wrong (OOM, unhandled rejections, event loop blocking, cascading failures), how did you diagnose and resolve it, and what changes did you make to prevent it from recurring?</summary>

**What the interviewer is looking for:**

- Calm, structured incident response under pressure.
- Ability to quickly narrow the problem using metrics and logs.
- Understanding of Node.js-specific failure modes (not generic "the server crashed").
- The postmortem: what you changed to prevent recurrence, not just how you fixed it.

**Suggested structure (STAR):**

1. **Situation:** What service, what time, how it was detected (alert, customer report, on-call page).
2. **Task:** What the immediate impact was (error rate, latency, downtime).
3. **Action:** Timeline of diagnosis and resolution — what you checked, in what order, what each clue told you.
4. **Result:** Root cause, fix, and systemic changes.

**Example outline to personalize:**

- **Situation:** "At 3 AM, PagerDuty fired: our order processing service's error rate jumped from 0.1% to 40%. Pods were restarting every 2-3 minutes."
- **Diagnosis timeline:**
  - Checked pod logs — saw `FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory` followed by `SIGKILL` (exit code 137).
  - Checked Grafana — `heapUsed` was hitting the container memory limit. Container had 1GB, `--max-old-space-size` wasn't set (defaulting to 1.5GB), so V8 tried to grow past the container limit and the OOM killer terminated it.
  - But why was memory growing? Checked recent deployments — no code changes. Checked data volume — order volume had spiked 3x due to a flash sale.
  - Root cause: The service loaded all pending orders into memory for batch processing. At 3x volume, the batch exceeded available heap.
- **Immediate fix:** Increased pod memory limit to 2GB and set `--max-old-space-size=1536` to give V8 a proper ceiling.
- **Permanent fix:** Refactored batch processing to use streaming/pagination — process 100 orders at a time instead of loading all pending orders.
- **Prevention:**
  - Added `--max-old-space-size` to all Node.js services (set to 75% of container memory).
  - Added memory usage alerts at 70% and 85% of the limit.
  - Added load testing with 3x expected volume as part of the release process.
  - Documented the incident in a postmortem shared with the team.

**Key points to hit:**

- Show the diagnostic order: alerts → logs → metrics → code. Not "I looked at the code and found the bug."
- Distinguish between the immediate mitigation (keep the service running) and the permanent fix (prevent recurrence).
- Mention the postmortem and systemic changes — this is what separates senior from mid-level incident handling.
- If it was cascading (one service failing caused others to fail), explain how you'd add circuit breakers, timeouts, or bulkheads.

</details>
