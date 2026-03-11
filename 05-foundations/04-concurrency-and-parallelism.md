# Concurrency & Parallelism

> **34 questions** — 21 theory, 13 practical

- Concurrency vs parallelism: definitions, single-core interleaving vs multi-core execution, how Node.js achieves concurrency without parallelism
- Concurrency hazards: race conditions, deadlocks, starvation, livelocks
- Node.js event loop: phases (timers, pending, poll, check, close), microtask queue starvation, CPU-bound blocking symptoms, starvation causes
- Threading models: single-threaded event loop vs thread-per-request vs goroutines — cooperative vs preemptive scheduling, how each handles long-running tasks
- Race conditions in async/await: check-then-act, concurrent API calls, lost updates
- Deadlocks: four conditions, database transactions, distributed locks, Promise chains
- Mutexes, semaphores, and locks: multi-threaded synchronization, Node.js single-threaded equivalents (async mutex, queue-per-key)
- Async patterns: callbacks, Promises, async/await — evolution, subtle bugs (lost stack traces, unhandled rejections), error propagation differences
- Async iterators and generators: for-await-of, async function*, producing/consuming async sequences, relationship to streams and backpressure
- I/O-bound vs CPU-bound work: why Node.js excels at I/O, offloading options for CPU (worker threads, child processes, separate services), and how to identify which category a workload falls into
- libuv thread pool: which operations use it, default size, saturation behavior
- Worker threads: main thread/worker communication, SharedArrayBuffer vs message passing
- Process-level concurrency: cluster module for multi-core HTTP servers, child_process (spawn, exec, fork), when processes beat threads
- Backpressure: fast producer / slow consumer problem, how Node.js streams handle it automatically, memory exhaustion from ignoring highWaterMark signals
- Concurrency limiting: unbounded async operations (Promise.all over thousands of items), connection pool saturation, patterns like p-limit and async queue
- Shared mutable state: why it causes bugs, immutability as defense, message passing between workers, actor model concept
- Distributed locks: Redis, advisory locks, TTL expiry, fencing tokens, Redlock
- Promise combinators: all, allSettled, race, any — behavior differences, error handling, and when each fits
- Event loop diagnostics: monitorEventLoopDelay, clinic.js, blocked-at, Chrome DevTools
- Cancellation patterns: AbortController/AbortSignal, draining in-flight requests, cooperative cancellation in async workflows

---

## Foundational

<details>
<summary>1. What is the difference between concurrency and parallelism — why is the distinction important, how does single-core interleaving differ from multi-core simultaneous execution, and why does Node.js achieve concurrency without parallelism on the main thread?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How do the three major threading models — single-threaded event loop (Node.js), thread-per-request (Java/Spring), and lightweight coroutines (Go goroutines) — each handle concurrent workloads, what is the difference between cooperative and preemptive scheduling, and what happens in each model when a task takes a long time to complete?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Why does Node.js excel at I/O-bound work but struggle with CPU-bound work — what is the fundamental architectural reason, what symptoms appear when CPU-bound work runs on the main thread, and what are the options for handling CPU-intensive tasks in a Node.js application?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. How does the Node.js event loop actually work — what happens in each phase (timers, pending callbacks, poll, check, close), how does the microtask queue (Promise callbacks, process.nextTick) interact with the phases, and why can microtask queue starvation starve I/O even when the event loop isn't blocked by CPU work?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What are the four main concurrency hazards — race conditions, deadlocks, starvation, and livelocks — how does each one manifest differently, and why are some of these still possible in single-threaded Node.js even without shared memory?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How do race conditions occur in async/await code even though Node.js is single-threaded — explain the check-then-act pattern, how concurrent API handler invocations lead to lost updates, and why the absence of threads doesn't protect you from logical races across await boundaries?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What are the four necessary conditions for a deadlock (Coffman conditions), and how do deadlocks manifest in different contexts — database transactions acquiring locks in different orders, distributed systems with multiple lock acquisitions, and even Promise chains that wait on each other — what strategies prevent each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What are mutexes, semaphores, and locks in multi-threaded systems, why do they exist, and what are the Node.js single-threaded equivalents — how do async mutexes and queue-per-key patterns solve the same coordination problems without OS-level threading primitives?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why does shared mutable state cause so many concurrency bugs, why is this problem fundamental rather than accidental, and what defenses exist — how do immutability, message passing between workers, and the actor model each address the root cause differently?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How did JavaScript's async patterns evolve from callbacks to Promises to async/await -- what specific problem did each generation solve that the previous couldn't, and why wasn't each new pattern a strict upgrade (what did you lose or trade away with each transition)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. What are the subtle bugs and error propagation differences across callbacks, Promises, and async/await -- how do lost stack traces with Promises, unhandled rejections with async/await, and swallowed errors in callbacks each manifest differently in production, and what defensive patterns prevent each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What is the libuv thread pool, which Node.js operations use it (and which don't), what is the default pool size, and what happens when the pool is saturated — why does DNS resolution use the thread pool while TCP sockets don't, and how does saturation cause seemingly unrelated operations to slow down?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. When should you use Worker threads in Node.js instead of other concurrency options — how does main thread/worker communication work, what is the difference between transferring data via structured clone (message passing) vs SharedArrayBuffer, and what are the tradeoffs and pitfalls of each approach?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. How does process-level concurrency work in Node.js — what does the cluster module do for multi-core HTTP servers, what are the differences between child_process.spawn, exec, and fork, and when do separate processes beat threads for concurrency?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. What is backpressure, why does the fast producer / slow consumer problem cause memory exhaustion, how do Node.js streams handle backpressure automatically, and what happens when you ignore backpressure signals — why is this one of the most common causes of Node.js memory issues in production?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. How do async iterators and generators work in Node.js -- what do `async function*` and `for-await-of` enable that regular iterators and Promises alone cannot, how do they relate to Node.js streams and backpressure handling, and when would you use an async generator to produce a sequence of values vs using a readable stream?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Why is unbounded concurrency dangerous when running many async operations simultaneously — what happens when you fire off thousands of concurrent HTTP requests or database queries without a concurrency limit, how does this differ from backpressure in streams, and what patterns exist to control the level of concurrency?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. How do distributed locks work and why are they harder than in-process locks — explain Redis-based locking (SETNX), the problem TTL expiry introduces, what fencing tokens solve, and why the Redlock algorithm exists — when should you use advisory database locks instead?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. What are the four Promise combinators (all, allSettled, race, any), how does each one behave when some promises resolve and others reject, and when does each one fit — why does the difference between all and allSettled matter for production error handling, and when would you choose race over any?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. How do cancellation patterns work in Node.js — what are AbortController and AbortSignal, how do you use them to cancel fetch requests, database queries, and custom async workflows, what does "cooperative cancellation" mean, and how do you drain in-flight requests gracefully during server shutdown?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Why do you need specialized tools to diagnose event loop problems instead of just using console.log or standard profiling — what do monitorEventLoopDelay, clinic.js, blocked-at, and Chrome DevTools each reveal that the others don't, and when would you reach for each one?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Implementation & Patterns

<details>
<summary>22. Given two concurrent Express request handlers that both read a user's balance, check if it's sufficient, and then deduct an amount — show the TypeScript code that demonstrates the race condition (check-then-act), explain exactly how two requests interleave across await boundaries to produce a lost update, and then show how to fix it using an async mutex or database-level locking</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Implement an async mutex and a queue-per-key pattern in TypeScript — show how the mutex serializes access to a shared resource, show how queue-per-key ensures operations on the same entity (e.g., same user ID) run sequentially while different entities run concurrently, and explain why this is the Node.js equivalent of fine-grained locking</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Write TypeScript examples that demonstrate when to use each Promise combinator (all, allSettled, race, any) — show a real scenario for each one, demonstrate how each handles partial failures differently, and show the error handling patterns that prevent silent failures in production</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Implement a concurrency limiter in TypeScript that processes an array of async tasks with a maximum number of concurrent operations — show the implementation, demonstrate what happens without the limiter (memory exhaustion, connection pool saturation), and explain how libraries like p-limit solve this in production</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Show how to offload CPU-intensive work to a Worker thread in Node.js — write the TypeScript code for both the main thread and the worker, demonstrate data transfer via message passing (postMessage) and via SharedArrayBuffer, and explain the serialization overhead tradeoff between the two approaches</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Implement cooperative cancellation using AbortController/AbortSignal in a TypeScript async workflow that involves multiple steps (HTTP fetch, database query, file write) — show how each step checks the signal, how to clean up partial work when cancelled, and how to wire this into graceful server shutdown to drain in-flight requests</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Set up a multi-core HTTP server using the Node.js cluster module — show the TypeScript code for the primary process and workers, explain how incoming connections are distributed across workers, what happens when a worker crashes, and compare this approach to using a reverse proxy (Nginx) with multiple Node.js processes</summary>

<!-- Answer will be added later -->

</details>

## Practical — Diagnostics & Debugging

<details>
<summary>29. Your Node.js API's response times have degraded and you suspect event loop blocking — walk through the exact diagnostic process: how to detect the blockage using monitorEventLoopDelay and clinic.js, how to identify the offending code using blocked-at or Chrome DevTools CPU profiling, and how to interpret the results to distinguish between CPU-bound blocking and microtask queue starvation</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Your Node.js application starts timing out on DNS lookups and crypto operations even though CPU usage is low — diagnose libuv thread pool saturation: show how to confirm the pool is the bottleneck, explain what operations are competing for the default 4 threads, demonstrate how to increase the pool size and restructure code to reduce thread pool pressure, and explain why simply increasing UV_THREADPOOL_SIZE isn't always the right fix</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>31. Tell me about a time you encountered a concurrency bug in production — a race condition, deadlock, or data inconsistency caused by concurrent operations — how did you discover it, how did you reproduce and diagnose the root cause, and what did you change to prevent it from happening again?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>32. Describe a time you had to handle CPU-bound work in a Node.js application — what was the workload, what symptoms told you the event loop was being blocked, and what solution did you choose (worker threads, child processes, separate service) and why?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>33. Tell me about a time you dealt with backpressure, memory exhaustion, or uncontrolled concurrency in a Node.js service — what caused the problem, how did you detect it, and what changes did you make to bring the system under control?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you implemented distributed locking or coordination across multiple services or instances — what mechanism did you use (Redis locks, database advisory locks, etc.), what failure modes did you account for, and what would you do differently knowing what you know now?</summary>

<!-- Answer framework will be added later -->

</details>
