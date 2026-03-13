# Performance & Optimization

> **26 questions** — 13 theory, 10 practical, 3 experience

- Measure first, optimize second — establishing baselines, profiling before changing code, validating fixes with before/after data
- Request lifecycle bottlenecks — where latency hides across DNS, TLS, load balancer, app server, database, and external APIs
- Key performance metrics — latency percentiles (p95, p99 over averages), throughput, saturation, error rate
- Multi-layer caching — CDN, reverse proxy, Redis, database query cache; cache-aside with stampede prevention, what belongs at each layer
- Cache invalidation tradeoffs — TTL-based vs event-based vs version-based, when not to cache
- HTTP caching headers — Cache-Control, ETag, Vary for static assets, API responses, user-specific data
- N+1 query problem — detection, ORM patterns (Prisma/TypeORM), eager loading, DataLoader/batching, denormalization
- Query optimization — EXPLAIN ANALYZE interpretation (Seq Scan, Index Scan, Bitmap Heap Scan), index strategies
- Connection pooling — PostgreSQL (PgBouncer, ORM pool) and Redis, pool sizing for Node.js, exhaustion symptoms
- Async processing — moving work out of the request path with queues/events, latency vs consistency tradeoffs
- Performance antipatterns — chatty I/O, retry storms, thundering herd, noisy neighbor, premature optimization; recognizing when optimization effort is justified vs wasted
- Node.js profiling — flame graphs (clinic.js, --prof), event loop lag detection (perf_hooks), --max-old-space-size, common causes (CPU-bound code, large JSON, sync I/O)
- Payload and serialization optimization — pagination strategies (cursor vs offset), field selection, avoiding over-fetching, JSON serialization cost for large payloads

---

## Foundational

<details>
<summary>1. Why is "measure first, optimize second" a non-negotiable principle for performance work — what goes wrong when engineers skip establishing baselines, how do you profile effectively before changing code, and what does validating a fix with before/after data actually look like in practice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Walk through the full lifecycle of an HTTP request from the client to the database and back — where does latency hide at each layer (DNS resolution, TLS handshake, load balancer, application server, database query, external API calls), and how do you determine which layer is the actual bottleneck rather than guessing?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Why are latency percentiles (p95, p99) far more useful than averages for understanding real user experience — how do throughput, saturation, and error rate complete the picture, and what dangerous performance problems does an average-only view hide?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. Why does caching need to happen at multiple layers (CDN, reverse proxy, Redis, database query cache) instead of just one — what type of data belongs at each layer, how do the layers interact, and what are the tradeoffs of caching too aggressively vs too conservatively?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What is cache stampede (thundering herd on cache miss) and why does naive cache-aside fail under high concurrency — what prevention strategies exist (locking, probabilistic early expiration, request coalescing) and what are the tradeoffs of each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What are the tradeoffs between TTL-based, event-based, and version-based cache invalidation — when is each approach appropriate, what consistency guarantees does each provide, and when should you avoid caching entirely?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do HTTP caching headers (Cache-Control, ETag, Vary) work together to control caching behavior — why do static assets, API responses, and user-specific data each need different caching strategies, and what breaks when you get Vary wrong or set overly aggressive max-age on mutable content?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why does the N+1 query problem occur so frequently with ORMs, how do you detect it in a running application, and what are the tradeoffs between eager loading, DataLoader/batching, and denormalization as solutions — when does each approach make sense vs cause new problems?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why do PostgreSQL and Redis need connection pooling, what problems does pool exhaustion cause, how does pool sizing work for Node.js specifically (single-threaded event loop vs traditional thread-per-request), and what symptoms tell you your pool is misconfigured?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why is moving work out of the request path (via queues or events) one of the most impactful performance techniques — what types of work are good candidates for async processing, what are the latency vs consistency tradeoffs, and when does async processing introduce more problems than it solves?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Explain the chatty I/O and noisy neighbor antipatterns -- why does each one emerge in production systems, how do you recognize each from symptoms and metrics alone, and what makes them particularly dangerous in microservice architectures?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Explain retry storms and thundering herd -- why does each one emerge after deployments or partial failures, how do you distinguish between them from metrics and logs, and why is premature optimization itself an antipattern? How do you decide when optimization effort is actually warranted vs wasted?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why is Node.js particularly vulnerable to certain performance problems (CPU-bound code blocking the event loop, large JSON serialization, synchronous I/O) — how does the single-threaded event loop model make these issues more severe than in multi-threaded runtimes, and what role does --max-old-space-size play in memory-related performance?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Caching & Query Optimization

<details>
<summary>14. Set up a multi-layer caching strategy for a read-heavy API endpoint — show how you'd configure CDN caching headers, a Redis cache-aside layer with stampede prevention (locking or early recomputation), and explain what cache-control directives you'd set at each layer and why</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Given an API endpoint that lists orders with their items and customer details, show how you'd detect an N+1 query problem in Prisma or TypeORM, fix it with eager loading, then show an alternative fix using DataLoader for a GraphQL resolver — explain the performance difference and when you'd choose denormalization instead</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. You have a slow PostgreSQL query — walk through interpreting the EXPLAIN ANALYZE output step by step, showing what Seq Scan, Index Scan, and Bitmap Heap Scan mean, how to read the actual vs estimated rows and cost, and demonstrate choosing the right index strategy (B-tree, partial index, composite index) to fix a specific slow query pattern</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Configure connection pooling for a Node.js application connecting to PostgreSQL — show the setup with PgBouncer in front of PostgreSQL plus the ORM-level pool configuration, explain how to size the pool for a Node.js process (why it's different from Java/Python), and show how you'd also configure Redis connection pooling with ioredis</summary>

<!-- Answer will be added later -->

</details>

## Practical — Performance Engineering

<details>
<summary>18. Refactor a synchronous API endpoint that sends an email, generates a PDF, and updates an analytics service into an async queue-based architecture — show the before and after code using BullMQ, explain what guarantees you lose by going async, and how you communicate the eventual result back to the client</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. You have a service that makes 15 sequential HTTP calls to another microservice to build a single response — show how you'd diagnose this chatty I/O antipattern (what the traces and latency profile look like), refactor it using batching or a composite endpoint, and explain what to do when you can't change the downstream service's API</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. An API endpoint returns a list of 10,000 records with all fields -- show how you'd optimize this with cursor-based pagination (explain why cursor beats offset at scale), field selection (sparse fieldsets or GraphQL), and response size reduction. Demonstrate the before/after payload sizes and explain the JSON serialization cost that makes large responses a latency problem even when the database query is fast.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Profiling & Debugging<details>
<summary>21. An API endpoint that was fast last month is now consistently slow (p99 went from 200ms to 2s) — walk through the systematic diagnosis process across every layer of the request lifecycle: how you'd isolate whether the problem is in DNS/TLS, the load balancer, application code, database queries, or an external dependency, showing the exact tools and commands at each step</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. After a deployment, your service starts showing exponentially increasing latency and eventually cascading failures across dependent services — diagnose whether this is a retry storm or thundering herd, show how each pattern manifests differently in metrics and logs, and demonstrate the specific fixes (exponential backoff with jitter, circuit breaking, request coalescing) with code examples</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Your Node.js service is intermittently timing out on database calls even though the database server shows low CPU and memory — walk through diagnosing connection pool exhaustion: what symptoms to look for in application metrics, how to confirm the pool is the bottleneck (vs slow queries or network issues), the exact PgBouncer and ORM pool settings to inspect, and how to fix it without just blindly increasing the pool size</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you diagnosed and fixed a significant production performance issue — what were the symptoms, how did you identify the root cause (not just the surface-level fix), and how did you validate that your fix actually worked under real traffic?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Describe a time you designed and implemented a caching strategy for a system — what drove the decision, what layer(s) did you cache at, how did you handle invalidation, and what unexpected problems did caching introduce?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>26. Describe a time you had to make a deliberate decision NOT to optimize something — what was the performance problem, why did you decide the optimization wasn't worth it, and how did you communicate that tradeoff to stakeholders?</summary>

<!-- Answer framework will be added later -->

</details>
