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

**What goes wrong without baselines:**

- You optimize the wrong thing. Intuition about bottlenecks is wrong more often than not -- engineers frequently guess "the database is slow" when the actual problem is a chatty downstream HTTP call or serialization overhead.
- You can't prove your fix worked. Without before numbers, a "50% improvement" is meaningless -- 50% of what?
- You introduce regressions unknowingly. An optimization in one area can shift the bottleneck or add overhead elsewhere.

**How to profile effectively before changing code:**

1. **Establish baseline metrics** -- capture p50, p95, p99 latency, throughput (req/s), error rate, and resource utilization (CPU, memory, event loop lag) under realistic load.
2. **Use distributed tracing** (Jaeger, Datadog APM) to break down where time is spent per request -- database, external calls, application logic.
3. **Profile the hot path** -- use Node.js `--prof` or clinic.js flame graphs for CPU-bound issues, heap snapshots for memory, and `perf_hooks` for event loop lag.
4. **Reproduce under load** -- single-request profiling misses concurrency problems. Use load testing (k6, Artillery) to expose issues that only appear at scale.

**What before/after validation looks like in practice:**

```
Before fix:
  p50: 120ms | p95: 450ms | p99: 1200ms | throughput: 200 req/s

After fix (added Redis cache for product lookups):
  p50: 45ms  | p95: 110ms | p99: 350ms  | throughput: 800 req/s
  Cache hit rate: 92%
```

You run the same load test with the same parameters, compare the distributions (not just averages), and monitor for at least a few days in production to catch edge cases the load test missed. If you can't show this kind of comparison, you don't actually know whether your optimization helped.

</details>

<details>
<summary>2. Walk through the full lifecycle of an HTTP request from the client to the database and back — where does latency hide at each layer (DNS resolution, TLS handshake, load balancer, application server, database query, external API calls), and how do you determine which layer is the actual bottleneck rather than guessing?</summary>

**Request lifecycle and where latency hides:**

| Layer | Typical Latency | Where Latency Hides |
|-------|----------------|---------------------|
| **DNS resolution** | 1-100ms | Uncached DNS lookups, slow resolvers, DNS propagation during migrations |
| **TLS handshake** | 10-100ms | Full handshake (2 round trips for TLS 1.2), expired session tickets forcing renegotiation, missing OCSP stapling |
| **Load balancer** | 1-10ms | Connection draining during deploys, health check misconfigurations routing to unhealthy instances, SSL termination overhead |
| **Application server** | Variable | Middleware chains (auth, validation, logging), synchronous blocking in Node.js event loop, memory pressure triggering GC pauses |
| **Database** | 1-1000ms+ | Missing indexes (sequential scans), lock contention, connection pool exhaustion (waiting for a connection, not the query itself), N+1 queries |
| **External APIs** | 50-5000ms+ | No timeouts set (waiting forever), sequential calls that should be parallel, DNS resolution to external hosts, cold starts on serverless dependencies |
| **Response serialization** | 1-500ms | Large JSON payloads (serializing 10K objects), unnecessary fields, no compression |

**How to identify the actual bottleneck:**

1. **Distributed tracing** is the primary tool. Each span in a trace (Jaeger, Datadog) shows exactly how much time was spent at each layer. If your trace shows 80% of time in a single database span, that's your bottleneck.
2. **Server-Timing header** -- add timing breakdowns to the response header so you can see from the client side:
   ```
   Server-Timing: db;dur=45.2, cache;dur=2.1, auth;dur=12.0, render;dur=8.5
   ```
3. **Network-level tools** -- `curl -w` with timing variables isolates DNS, TLS, and TTFB:
   ```bash
   curl -w "dns: %{time_namelookup}s\ntls: %{time_appconnect}s\nttfb: %{time_starttransfer}s\ntotal: %{time_total}s\n" -o /dev/null -s https://api.example.com/orders
   ```
4. **Compare internal vs external latency** -- if your application metrics show 50ms internal latency but clients see 500ms, the problem is between the client and your servers (CDN, DNS, TLS, load balancer). If internal latency matches external, the problem is in your code or dependencies.

</details>

<details>
<summary>3. Why are latency percentiles (p95, p99) far more useful than averages for understanding real user experience — how do throughput, saturation, and error rate complete the picture, and what dangerous performance problems does an average-only view hide?</summary>

**Why percentiles beat averages:**

Averages hide the worst experiences. If 95% of requests take 50ms and 5% take 5000ms, the average is ~300ms -- which describes nobody's actual experience. The p95 (5000ms) tells you that 1 in 20 users is having a terrible time. For high-traffic services, 5% means thousands of users per hour.

Averages are particularly dangerous because:
- A bimodal distribution (fast cache hits + slow cache misses) looks "fine" as an average
- A gradually degrading tail (p99 creeping from 500ms to 3s) is invisible in averages until it's catastrophic
- Averages can actually *improve* during an outage if slow requests start timing out and getting dropped

**The four key metrics (USE/RED hybrid):**

| Metric | What it tells you | Example danger signal |
|--------|-------------------|----------------------|
| **Latency percentiles** (p50, p95, p99) | How fast individual requests are, including the tail | p99 spiking while p50 stays flat = some requests hitting a slow path (cache miss, lock contention, GC pause) |
| **Throughput** (req/s) | How much work the system is doing | Throughput dropping while latency stays flat = requests being rejected or queued upstream before reaching your service |
| **Saturation** | How full your resources are (CPU, memory, connections, queue depth) | 85% connection pool utilization = you're one traffic spike away from exhaustion, even though everything looks fine right now |
| **Error rate** | What percentage of work is failing | Error rate climbing with stable latency = fast failures (timeouts, circuit breaker trips), not slowness |

**What averages hide -- a concrete example:**

A service has average latency of 100ms. Looks healthy. But the p99 is 8 seconds because 1% of requests trigger a missing index path that does a sequential scan. Those 1% of users retry, adding more load. The average barely moves. By the time the average visibly degrades, the system is already in a cascading failure. The p99 would have caught this days earlier.

</details>

## Conceptual Depth

<details>
<summary>4. Why does caching need to happen at multiple layers (CDN, reverse proxy, Redis, database query cache) instead of just one — what type of data belongs at each layer, how do the layers interact, and what are the tradeoffs of caching too aggressively vs too conservatively?</summary>

**Why multiple layers:** Each layer serves a different purpose and intercepts requests at a different point. A CDN prevents requests from ever reaching your infrastructure. Redis prevents your application from hitting the database. A database query cache prevents re-parsing and re-planning identical queries. No single layer can do all of this efficiently.

**What belongs at each layer:**

| Layer | Best for | TTL range | Example |
|-------|----------|-----------|---------|
| **CDN** (CloudFront, Fastly) | Static assets, public API responses that rarely change | Minutes to days | Product images, public catalog pages, JS/CSS bundles |
| **Reverse proxy** (Nginx, Varnish) | Response-level caching for repeated identical requests | Seconds to minutes | API responses for popular product pages, common search queries |
| **Application cache** (Redis/Memcached) | Computed data, session state, frequently accessed business objects | Seconds to hours | User permissions, shopping cart, aggregated product data from multiple DB tables |
| **Database query cache** (PG `pg_stat_statements`, materialized views) | Expensive analytical queries, read replicas | Minutes to hours | Dashboard aggregations, report data |

**How layers interact:** Requests flow through layers in order -- CDN first, then reverse proxy, then app-level cache, then database. A cache hit at any layer short-circuits everything below it. The key design decision is consistency: a CDN cache hit means the user might see stale data even if you've already invalidated the Redis cache. You need to think about invalidation at every layer, not just one.

**Caching too aggressively vs too conservatively:**

- **Too aggressive:** Users see stale data (prices, inventory, permissions), invalidation becomes a nightmare, debugging is harder because you can't tell if you're looking at cached or fresh data, and cache stampedes on expiry become severe because everything expires at once.
- **Too conservative:** Every request hits the database, response times are unnecessarily high, infrastructure costs more to handle the load, and you miss the 80/20 opportunity where caching the top 20% of hot keys eliminates 80% of database load.

**The practical rule:** Cache immutable data aggressively (content-addressed assets, historical records). Cache mutable data with short TTLs plus event-based invalidation. Never cache data where staleness causes correctness problems (payment amounts, auth tokens mid-revocation).

</details>

<details>
<summary>5. What is cache stampede (thundering herd on cache miss) and why does naive cache-aside fail under high concurrency — what prevention strategies exist (locking, probabilistic early expiration, request coalescing) and what are the tradeoffs of each?</summary>

**Cache stampede** happens when a popular cache key expires and hundreds of concurrent requests all see the miss simultaneously, all hit the database to recompute the value, and all write back to cache. The database gets hammered with identical expensive queries at the exact same moment.

**Why naive cache-aside fails:** The standard pattern -- check cache, miss, query DB, write cache -- has no coordination between concurrent requests. With a hot key serving 1000 req/s and a TTL of 60s, the moment it expires, potentially dozens of requests hit the DB before the first one finishes recomputing and writing back.

**Prevention strategies:**

**1. Distributed locking (mutex)**
```typescript
async function getWithLock(key: string, computeFn: () => Promise<string>) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const acquired = await redis.set(lockKey, '1', 'EX', 5, 'NX'); // 5s lock TTL

  if (acquired) {
    const value = await computeFn();
    await redis.set(key, JSON.stringify(value), 'EX', 300);
    await redis.del(lockKey);
    return value;
  }

  // Didn't get lock — wait briefly and retry from cache
  await new Promise((r) => setTimeout(r, 100));
  return getWithLock(key, computeFn); // retry
}
```
- **Tradeoff:** Only one request recomputes, but losers must wait/retry. Adds complexity around lock TTL (too short = multiple holders; too long = stale on crash). Retry loops can add latency for waiting requests.

**2. Probabilistic early expiration (stale-while-revalidate)**
Each cached value includes its expiry time. As the TTL approaches, requests have an increasing probability of triggering a background refresh *before* the key actually expires.
- **Tradeoff:** No coordination needed, no lock contention. But the recomputation happens while the old value is still valid, so you do extra work preemptively. Works best for keys with predictable access patterns.

**3. Request coalescing (singleflight)**
```typescript
const inFlight = new Map<string, Promise<any>>();

async function coalesced(key: string, computeFn: () => Promise<any>) {
  if (inFlight.has(key)) return inFlight.get(key);

  const promise = computeFn().finally(() => inFlight.delete(key));
  inFlight.set(key, promise);
  return promise;
}
```
- **Tradeoff:** Dead simple, works within a single process. But only coalesces within one Node.js instance -- in a multi-instance deployment, you still get N concurrent recomputations (one per instance). Combine with locking for full protection.

**Which to use:** For most cases, request coalescing + short distributed lock is the best combination. Probabilistic early refresh is great for very hot keys where you can afford the extra compute cost to avoid any miss at all.

</details>

<details>
<summary>6. What are the tradeoffs between TTL-based, event-based, and version-based cache invalidation — when is each approach appropriate, what consistency guarantees does each provide, and when should you avoid caching entirely?</summary>

| Strategy | How it works | Consistency | Complexity | Best for |
|----------|-------------|-------------|------------|----------|
| **TTL-based** | Cache entries expire after a fixed duration | Eventual (stale up to TTL window) | Low | Data where short staleness is acceptable -- product catalog, search results, public API responses |
| **Event-based** | Cache is explicitly invalidated when the underlying data changes (via pub/sub, webhooks, CDC) | Near-real-time | High | Data that must be fresh when updated -- user permissions, inventory counts, pricing |
| **Version-based** | Cache key includes a version number or hash; changing the data bumps the version, making old keys orphans | Strong (reads always get correct version) | Medium | Immutable-by-design data -- content-addressed assets, compiled templates, config snapshots |

**TTL-based** is the default starting point. Simple to implement, no coupling between write and cache paths. The tradeoff is a staleness window -- if TTL is 60s, users might see data up to 60s old. Making TTL shorter reduces staleness but increases cache miss rate and DB load.

**Event-based** gives you fresh data almost immediately but couples your write path to your cache layer. If the event delivery fails or is delayed, you serve stale data with no TTL safety net. You often combine this with a TTL as a fallback: event-based invalidation for fast refresh, TTL as a safety net to prevent stale data if an event is lost.

**Version-based** sidesteps invalidation entirely -- you never update a cache key, you create a new one. This works beautifully for things like `product:v42` or `bundle-a3f8c2.js`. The downside is you need a way to know the current version (which itself might need to be cached), and orphaned old keys waste memory until they're evicted.

**When NOT to cache:**

- **Transactional data mid-flight** -- order totals during checkout, payment amounts, anything where staleness causes monetary errors
- **Highly personalized data with low reuse** -- if every user gets a unique response, cache hit rates will be near zero and you're just wasting memory
- **Data that changes faster than your TTL** -- caching stock tickers with a 30s TTL gives you wrong data most of the time
- **Write-heavy workloads** -- if you invalidate more often than you read, caching adds overhead without benefit

</details>

<details>
<summary>7. How do HTTP caching headers (Cache-Control, ETag, Vary) work together to control caching behavior — why do static assets, API responses, and user-specific data each need different caching strategies, and what breaks when you get Vary wrong or set overly aggressive max-age on mutable content?</summary>

**How the headers work together:**

- **Cache-Control** tells caches (browser, CDN, proxy) *whether* to cache and *for how long*. Key directives: `max-age=3600` (cache for 1 hour), `no-cache` (cache it but revalidate every time), `no-store` (don't cache at all), `private` (only browser can cache, not CDN/proxy), `public` (any cache can store it), `s-maxage` (override max-age for shared caches like CDNs).
- **ETag** is a fingerprint of the response content. On subsequent requests, the client sends `If-None-Match: <etag>` and the server returns 304 Not Modified if unchanged -- saving bandwidth but not the round trip.
- **Vary** tells caches that the response differs based on specific request headers. `Vary: Accept-Encoding` means the gzipped and uncompressed versions are different cache entries. `Vary: Authorization` means each user's response is cached separately.

**Different strategies for different content:**

**Static assets** (JS, CSS, images with content hashes in filenames):
```
Cache-Control: public, max-age=31536000, immutable
```
Cache forever. The filename changes when content changes (`bundle-a3f8c2.js`), so stale cache is impossible.

**Public API responses** (product listings, search results):
```
Cache-Control: public, s-maxage=60, stale-while-revalidate=30
ETag: "v42-products"
```
CDN caches for 60s, can serve stale for 30s more while revalidating in background.

**User-specific data** (account details, cart):
```
Cache-Control: private, no-cache
ETag: "user-123-v7"
```
Only browser caches, must revalidate every time. The CDN never stores this.

**What breaks with wrong Vary:** If you omit `Vary: Authorization` on user-specific responses, a CDN caches User A's data and serves it to User B -- a security breach. If you set `Vary: *`, nothing is ever cached because every request is treated as unique. If you set `Vary: Cookie` on a response where cookie values vary per user, you effectively disable CDN caching since every user creates a separate cache entry.

**What breaks with aggressive max-age on mutable content:** If you set `Cache-Control: max-age=86400` on an API response and then the data changes, users see stale data for up to 24 hours with *no way to force a refresh*. Unlike server-side caches you control, you cannot invalidate browser caches or intermediate proxy caches. This is why mutable content should use short `max-age` with `stale-while-revalidate`, or `no-cache` with ETags.

</details>

<details>
<summary>8. Why does the N+1 query problem occur so frequently with ORMs, how do you detect it in a running application, and what are the tradeoffs between eager loading, DataLoader/batching, and denormalization as solutions — when does each approach make sense vs cause new problems?</summary>

**Why ORMs cause N+1 so frequently:** ORMs default to lazy loading -- they fetch the parent entity, then fetch each related entity only when accessed. This is a sensible default for single-entity operations, but when you iterate over a list, each iteration triggers a separate query. Fetching 50 orders with their items becomes 1 query for orders + 50 queries for items = 51 queries instead of 2.

**How to detect it:**

- **Query logging** -- enable ORM query logging and look for repeated identical query patterns with different IDs. In Prisma, set `log: ['query']` in the client constructor.
- **Distributed tracing** -- a trace for a single request showing 50+ database spans with near-identical queries is a dead giveaway.
- **APM tools** -- Datadog, New Relic, etc. flag "N+1 detected" automatically by recognizing repeated similar queries within a single request.
- **pg_stat_statements** -- if a simple SELECT by ID has dramatically more calls than expected, it's likely being called in a loop.

**Solutions and tradeoffs:**

**Eager loading (JOINs / includes):**
```typescript
// Prisma: single query with JOIN
const orders = await prisma.order.findMany({
  include: { items: true, customer: true },
});
```
- **Pro:** Simple, single round trip (or 2-3 queries for Prisma which uses separate queries per relation).
- **Con:** Can over-fetch if you don't always need the related data. Large JOINs can produce cartesian explosions -- 50 orders x 20 items each = 1000 rows transferred.
- **Best when:** You always need the related data and the result set is bounded.

**DataLoader / batching:**
```typescript
// Collects all IDs in a single tick, batches into one query
const itemLoader = new DataLoader(async (orderIds: string[]) => {
  const items = await prisma.orderItem.findMany({
    where: { orderId: { in: orderIds } },
  });
  // Group by orderId to return in correct order
  return orderIds.map((id) => items.filter((i) => i.orderId === id));
});
```
- **Pro:** Only fetches what's actually accessed, batches automatically across resolvers. Perfect for GraphQL where different clients request different fields.
- **Con:** Adds a layer of indirection, requires per-request DataLoader instances (they cache within a request). Debugging is less obvious than a JOIN.
- **Best when:** GraphQL, or when different code paths access different relations unpredictably.

**Denormalization:**
Store pre-computed data (e.g., embed item summaries directly in the order document/row) so no join is needed at all.
- **Pro:** Single query, no joins, fastest reads possible.
- **Con:** Write complexity increases (must update denormalized data when source changes), risk of data inconsistency, storage increases.
- **Best when:** Read-heavy, write-rare data where the read pattern is predictable and consistency can be eventually handled.

</details>

<details>
<summary>9. Why do PostgreSQL and Redis need connection pooling, what problems does pool exhaustion cause, how does pool sizing work for Node.js specifically (single-threaded event loop vs traditional thread-per-request), and what symptoms tell you your pool is misconfigured?</summary>

**Why pooling is needed:**

**PostgreSQL** -- each connection spawns a separate OS process (~10MB RAM). Creating a connection involves TCP handshake + authentication + TLS negotiation (~20-50ms). Without pooling, every query pays this cost, and under load, hundreds of connections overwhelm the server. PostgreSQL has a hard `max_connections` limit (default 100), after which new connections are refused.

**Redis** -- while Redis is single-threaded and connections are cheaper than PostgreSQL, the TCP connection setup overhead still matters at high throughput. More importantly, connection pools provide connection reuse, automatic reconnection on failure, and pipeline batching.

**What pool exhaustion causes:**

Requests queue up waiting for an available connection. The application appears to hang -- database queries that normally take 5ms now take 30 seconds (not because the query is slow, but because it waited 29.995 seconds for a connection). Eventually, request timeouts cascade: upstream services timeout, retry, and make the problem worse.

**Pool sizing for Node.js:**

Node.js is fundamentally different from Java/Python here. In Java, each request runs on its own thread and holds a connection for the duration. You need pool size >= concurrent threads. Node.js has a single event loop -- it fires off async queries and releases the connection back to the pool as soon as the query is sent, picking up the result via callback. This means:

- A Node.js process can handle hundreds of concurrent requests with a pool of **5-20 connections**
- The formula is roughly: `pool_size = number_of_concurrent_queries_in_flight` (not number of concurrent requests)
- With 4 Node.js instances, each with a pool of 10, you use 40 PostgreSQL connections total
- The PgBouncer layer in front of PostgreSQL can multiplex these further, allowing hundreds of application-side connections to share a smaller number of real PostgreSQL connections

**Symptoms of misconfigured pool:**

| Symptom | Likely cause |
|---------|-------------|
| Intermittent timeouts on DB calls, DB server itself shows low load | Pool too small -- requests queuing for connections |
| Memory usage on DB server climbing steadily | Pool too large -- too many idle connections consuming PostgreSQL process memory |
| Latency spikes correlating with deploy/restart | No connection pre-warming -- all connections created under load |
| "too many connections" errors | Total pool size across all instances exceeds `max_connections` (don't forget connections from migrations, monitoring, admin tools) |
| Connections stuck in "idle in transaction" | Application code opens a transaction but doesn't commit/rollback promptly, holding the connection hostage |

</details>

<details>
<summary>10. Why is moving work out of the request path (via queues or events) one of the most impactful performance techniques — what types of work are good candidates for async processing, what are the latency vs consistency tradeoffs, and when does async processing introduce more problems than it solves?</summary>

**Why it's so impactful:** The critical path of an HTTP request determines response time. If your endpoint sends an email (200ms), generates a PDF (500ms), and updates analytics (100ms) synchronously, your response takes 800ms+ even though the user only cares about the core operation (e.g., "order placed"). Moving non-critical work to a queue can cut response time by 80%+ in many cases.

**Good candidates for async processing:**

- **Notifications** -- email, SMS, push notifications. Users don't need to wait for the email to be sent to see "order confirmed."
- **Document generation** -- PDFs, reports, exports. These are CPU-intensive and often take seconds.
- **Analytics and audit logging** -- recording events, updating counters, syncing to data warehouses.
- **Image/video processing** -- resizing, transcoding, thumbnail generation.
- **Third-party integrations** -- syncing data to external systems (CRM, ERP) that have unreliable latency.
- **Expensive computations** -- recommendations, search index updates, cache warming.

**The rule of thumb:** If the user doesn't need the result immediately to continue their workflow, it's a candidate.

**Latency vs consistency tradeoffs:**

Synchronous processing gives you **immediate consistency** -- the response includes the complete result of all operations. Async processing gives you **eventual consistency** -- the response confirms the primary operation, but side effects complete later.

This means:
- The user might see "order placed" before the confirmation email arrives
- If the async job fails, the user doesn't get immediate feedback
- You need mechanisms to handle failures: retries, dead letter queues, idempotency
- The client may need to poll or subscribe (WebSocket) for completion status

**When async processing makes things worse:**

- **When the user actually needs the result** -- generating a receipt the user needs to download right now cannot be deferred
- **When ordering matters** -- if operations must happen in sequence and later steps depend on earlier ones, queues add complexity to maintain ordering
- **When the failure handling cost exceeds the latency gain** -- if you need to build retry logic, DLQ monitoring, idempotency, compensation logic, and status tracking, the operational overhead may outweigh the latency improvement for low-traffic endpoints
- **When the queue itself becomes a bottleneck** -- if Redis (the queue backend) goes down, all async work stops. You've traded a slow endpoint for a single point of failure
- **Simple, fast side effects** -- if the side effect takes 10ms, the overhead of enqueue + dequeue + serialization may be comparable. Don't queue work that's already fast

</details>

<details>
<summary>11. Explain the chatty I/O and noisy neighbor antipatterns -- why does each one emerge in production systems, how do you recognize each from symptoms and metrics alone, and what makes them particularly dangerous in microservice architectures?</summary>

**Chatty I/O:**

A service makes many small, sequential network calls where fewer batched calls would suffice. This emerges naturally when developers write code that fetches related data in loops -- the code reads clearly but produces terrible network behavior.

**Why it emerges:** It's the network equivalent of N+1 queries (covered in question 8). A developer writes `for (const id of ids) { await fetch(serviceB/items/${id}) }` because it's the obvious code. It works in development with 3 items and localhost latency. In production with 50 items and 5ms network latency per call, that's 250ms of pure network overhead -- before the downstream service even processes anything.

**How to recognize it from metrics:**
- Distributed traces show many short, sequential spans to the same downstream service within a single request
- Latency scales linearly with the number of related entities (10 items = 50ms, 100 items = 500ms)
- The downstream service sees high request rates but each request is tiny and fast
- Network bandwidth utilization is low but connection count is high

**Noisy neighbor:**

One tenant, customer, or workload on shared infrastructure consumes disproportionate resources, degrading performance for everyone else. This is inherent to any multi-tenant or shared-resource system.

**Why it emerges:** Shared infrastructure (databases, Redis clusters, Kubernetes nodes, connection pools) is provisioned for average load. One tenant running a massive export query, one pod with a memory leak, or one customer triggering an expensive API pattern can saturate a shared resource.

**How to recognize it from metrics:**
- Performance degradation correlates with a specific tenant's activity patterns (their batch jobs, peak hours)
- Resource utilization (CPU, memory, I/O) spikes on specific nodes but not uniformly across the cluster
- Latency increases for all tenants simultaneously without a deployment or traffic change
- Database slow query logs show expensive queries from a single source

**Why both are especially dangerous in microservices:**

- **Chatty I/O** gets amplified across service hops. Service A chatters to B, which chatters to C -- the multiplicative effect creates exponential request amplification. A single user request can generate hundreds of internal network calls.
- **Noisy neighbor** is harder to isolate in microservices because resources are shared at multiple levels (shared database, shared Kubernetes node, shared network). The blast radius is larger and the source is harder to trace -- the noisy pod might be 3 hops away from the affected service.

</details>

<details>
<summary>12. Explain retry storms and thundering herd -- why does each one emerge after deployments or partial failures, how do you distinguish between them from metrics and logs, and why is premature optimization itself an antipattern? How do you decide when optimization effort is actually warranted vs wasted?</summary>

**Retry storms:**

When a downstream service fails or slows down, clients retry their requests. If each client retries 3 times with no backoff, a service receiving 1000 req/s suddenly receives 3000 req/s -- all hitting an already-struggling dependency. The retries make the failure worse, not better.

**Why it emerges after deployments:** A new deployment introduces a bug or slow path. Requests start failing. Clients retry. The increased load from retries overwhelms the service, causing more failures, causing more retries -- a positive feedback loop. Even after a rollback, the retry backlog can keep the system overloaded for minutes.

**Thundering herd:**

Many clients simultaneously attempt the same action -- reconnecting after an outage, cache stampede on expiry (covered in question 5), or all cron jobs firing at the same second. Unlike retry storms, this isn't caused by retries -- it's caused by synchronized behavior.

**Why it emerges after failures:** A database goes down for 30 seconds. When it comes back, every service instance simultaneously opens new connections, runs health checks, and replays queued queries. The database, barely recovered, gets hit with 10x normal load and goes down again.

**How to distinguish them from metrics:**

| Signal | Retry storm | Thundering herd |
|--------|-------------|-----------------|
| **Request pattern** | Repeated requests with same IDs/parameters from same clients | Many unique requests arriving simultaneously |
| **Error rates** | Downstream errors spike first, then upstream errors follow | Errors spike simultaneously across all clients |
| **Traffic shape** | Exponential growth in request volume to the failing service | Sudden spike followed by gradual decay |
| **Logs** | Same request ID appearing 3-5 times with increasing timestamps | Many different request IDs all with the same timestamp |
| **Trigger** | Partial failure or slow response from one service | Recovery from outage, cache expiry, scheduled event |

**Premature optimization as an antipattern:**

Premature optimization means spending engineering time improving performance before you have evidence that it matters. It's an antipattern because:
- It adds complexity that slows future development (clever caching, denormalized schemas, custom serializers)
- Intuition about bottlenecks is unreliable -- you optimize the wrong thing (as covered in question 1)
- The "optimized" code is harder to maintain, debug, and change
- Business requirements may change, making the optimization irrelevant

**When optimization IS warranted:**
- You have measured data showing a specific bottleneck (not a hunch)
- The performance problem affects user experience or business metrics (SLA violations, customer complaints, revenue impact)
- The cost of optimization (engineering time, added complexity) is proportional to the impact
- You've exhausted simpler solutions first (is there a missing index? an N+1 query? an unnecessary synchronous call?)

**The practical test:** Can you answer "what metric will improve, by how much, and how will we verify it?" If you can't, it's premature.

</details>

<details>
<summary>13. Why is Node.js particularly vulnerable to certain performance problems (CPU-bound code blocking the event loop, large JSON serialization, synchronous I/O) — how does the single-threaded event loop model make these issues more severe than in multi-threaded runtimes, and what role does --max-old-space-size play in memory-related performance?</summary>

**The core vulnerability:** Node.js runs all JavaScript on a single thread. In Java or Go, a CPU-bound operation blocks one thread while other threads continue serving requests. In Node.js, a CPU-bound operation blocks the *only* thread -- every concurrent request freezes until it finishes. A 100ms synchronous computation means 100ms of zero throughput for the entire process.

**The three main problem categories:**

**1. CPU-bound code blocking the event loop:**
- Cryptographic operations (`crypto.pbkdf2Sync`), complex regex on large strings, tight computational loops
- Sorting/transforming large in-memory datasets
- Template rendering with large data sets

In a multi-threaded runtime, this blocks one thread out of 200. In Node.js, it blocks 1 out of 1. If your event loop is blocked for 200ms, a service handling 500 req/s has 100 requests queued.

**Detection:** Use `perf_hooks` monitorEventLoopDelay or the `blocked-at` package:
```typescript
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

// Check periodically
setInterval(() => {
  console.log(`Event loop p99: ${histogram.percentile(99) / 1e6}ms`);
  histogram.reset();
}, 5000);
```

**2. Large JSON serialization:**
`JSON.stringify()` and `JSON.parse()` are synchronous and run on the main thread. Serializing a 10MB object can block the event loop for 50-200ms. This is invisible in profiling tools that only measure async I/O -- the CPU is busy, not waiting.

**Mitigation:** Stream large responses instead of building them in memory, paginate results (covered in question 20), or use `JSON.stringify` on smaller chunks. For extreme cases, offload to a worker thread.

**3. Synchronous I/O:**
`fs.readFileSync`, `execSync`, or any `*Sync` API blocks the event loop during the entire I/O operation. A 50ms disk read blocks all requests for 50ms. This is especially dangerous in request handlers -- it's fine at startup (loading config), but catastrophic in hot paths.

**`--max-old-space-size` and memory performance:**

V8's default heap limit is ~1.5-2GB (varies by platform). When your application approaches this limit:
1. The garbage collector runs more frequently and aggressively, causing GC pauses that block the event loop (same problem as CPU-bound code)
2. Eventually, V8 throws an out-of-memory error and the process crashes

`--max-old-space-size=4096` raises the limit to 4GB. But this is a double-edged sword:
- **Too low:** Frequent GC pauses under normal load, OOM crashes during traffic spikes
- **Too high:** The GC has more memory to scan, so individual GC pauses can be *longer*. Also masks memory leaks that would crash earlier and be detected sooner
- **Right approach:** Set it based on your container's memory limit (leave ~25% for the OS and non-heap memory), then investigate *why* you need more heap rather than just raising the ceiling

</details>

## Practical — Caching & Query Optimization

<details>
<summary>14. Set up a multi-layer caching strategy for a read-heavy API endpoint — show how you'd configure CDN caching headers, a Redis cache-aside layer with stampede prevention (locking or early recomputation), and explain what cache-control directives you'd set at each layer and why</summary>

**Scenario:** A product catalog endpoint (`GET /products/:slug`) that gets 5000 req/s, updates a few times per day, and serves the same data to all users.

**Layer 1: CDN (CloudFront/Fastly)**

Set response headers so the CDN caches the response and serves it without hitting your origin:

```typescript
// In the response handler
res.setHeader('Cache-Control', 'public, s-maxage=30, stale-while-revalidate=60');
res.setHeader('Vary', 'Accept-Encoding');
// No Vary on Authorization — this is public data
```

- `s-maxage=30` -- CDN caches for 30 seconds. Short enough that updates propagate quickly, long enough to absorb most traffic.
- `stale-while-revalidate=60` -- CDN can serve stale content for 60s while fetching a fresh copy in the background. Users never see a slow response due to revalidation.
- `Vary: Accept-Encoding` -- separate cache entries for gzip vs uncompressed. No other Vary headers since this is the same for all users.

**Layer 2: Redis cache-aside with stampede prevention**

```typescript
import Redis from 'ioredis';

const redis = new Redis();
const inFlight = new Map<string, Promise<any>>(); // per-process coalescing

async function getProduct(slug: string) {
  const cacheKey = `product:${slug}`;

  // Check Redis first
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // Request coalescing — only one in-flight computation per key per process
  if (inFlight.has(cacheKey)) return inFlight.get(cacheKey);

  const promise = recomputeWithLock(cacheKey, slug);
  inFlight.set(cacheKey, promise);
  promise.finally(() => inFlight.delete(cacheKey));
  return promise;
}

async function recomputeWithLock(cacheKey: string, slug: string) {
  const lockKey = `lock:${cacheKey}`;
  const acquired = await redis.set(lockKey, '1', 'EX', 5, 'NX');

  if (!acquired) {
    // Another instance is recomputing — wait briefly and check cache again
    await new Promise((r) => setTimeout(r, 50));
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);
    // Fallback: compute anyway rather than failing
  }

  try {
    const product = await prisma.product.findUnique({
      where: { slug },
      include: { variants: true, images: true },
    });

    if (product) {
      await redis.set(cacheKey, JSON.stringify(product), 'EX', 300); // 5 min TTL
    }
    return product;
  } finally {
    if (acquired) await redis.del(lockKey);
  }
}
```

This combines two stampede prevention techniques from question 5: per-process request coalescing (free, handles same-instance deduplication) and distributed locking (handles cross-instance deduplication).

**Layer 3: Event-based invalidation on writes**

When a product is updated, proactively invalidate rather than waiting for TTL:

```typescript
async function updateProduct(slug: string, data: UpdateProductInput) {
  const product = await prisma.product.update({ where: { slug }, data });
  // Invalidate Redis — next read will recompute
  await redis.del(`product:${slug}`);
  // Optionally: purge CDN cache via API (CloudFront invalidation, Fastly purge)
  await cdn.purge(`/products/${slug}`);
  return product;
}
```

**Why this layering works:** The CDN handles 95%+ of traffic without touching your servers. Redis handles the remaining cache-miss traffic without touching the database. The database only gets hit on genuine cache misses (once every 5 minutes per product, or immediately after updates). The stampede prevention ensures that even a popular product's cache miss only triggers a single database query.

</details>

<details>
<summary>15. Given an API endpoint that lists orders with their items and customer details, show how you'd detect an N+1 query problem in Prisma or TypeORM, fix it with eager loading, then show an alternative fix using DataLoader for a GraphQL resolver — explain the performance difference and when you'd choose denormalization instead</summary>

**The problematic code:**

```typescript
// REST endpoint — looks innocent, produces N+1
app.get('/orders', async (req, res) => {
  const orders = await prisma.order.findMany({ take: 50 }); // 1 query

  const result = await Promise.all(
    orders.map(async (order) => ({
      ...order,
      items: await prisma.orderItem.findMany({ where: { orderId: order.id } }), // 50 queries
      customer: await prisma.customer.findUnique({ where: { id: order.customerId } }), // 50 queries
    })),
  );
  // Total: 101 queries for 50 orders
  res.json(result);
});
```

**Detection:**

Enable Prisma query logging to see the problem:
```typescript
const prisma = new PrismaClient({ log: ['query'] });
// Output shows 101 separate SELECT statements for a single request
```

In distributed tracing (Datadog/Jaeger), you'd see 101 database spans fanned out within a single request trace -- an unmistakable pattern.

**Fix 1: Eager loading with Prisma `include`**

```typescript
app.get('/orders', async (req, res) => {
  const orders = await prisma.order.findMany({
    take: 50,
    include: {
      items: true,      // JOIN or separate batched query
      customer: true,    // JOIN or separate batched query
    },
  });
  // Prisma generates 2-3 queries total (one per relation), not 101
  res.json(orders);
});
```

Prisma's `include` doesn't use SQL JOINs -- it runs separate `SELECT ... WHERE id IN (...)` queries for each relation, which avoids cartesian explosion. This turns 101 queries into 3.

**Fix 2: DataLoader for GraphQL resolvers**

In GraphQL, the client controls which fields are requested, so you can't eagerly load everything (you'd over-fetch). DataLoader batches by collecting all IDs within a single event loop tick:

```typescript
import DataLoader from 'dataloader';

// Create per-request loaders (important: new instance per request to avoid cross-request caching)
function createLoaders() {
  return {
    orderItems: new DataLoader<string, OrderItem[]>(async (orderIds) => {
      const items = await prisma.orderItem.findMany({
        where: { orderId: { in: [...orderIds] } },
      });
      // Must return results in same order as input IDs
      const grouped = new Map<string, OrderItem[]>();
      items.forEach((item) => {
        const list = grouped.get(item.orderId) || [];
        list.push(item);
        grouped.set(item.orderId, list);
      });
      return orderIds.map((id) => grouped.get(id) || []);
    }),

    customers: new DataLoader<string, Customer>(async (customerIds) => {
      const customers = await prisma.customer.findMany({
        where: { id: { in: [...customerIds] } },
      });
      const map = new Map(customers.map((c) => [c.id, c]));
      return customerIds.map((id) => map.get(id)!);
    }),
  };
}

// GraphQL resolvers
const resolvers = {
  Order: {
    items: (order: Order, _args: unknown, ctx: Context) =>
      ctx.loaders.orderItems.load(order.id),
    customer: (order: Order, _args: unknown, ctx: Context) =>
      ctx.loaders.customers.load(order.customerId),
  },
};
```

If the client requests 50 orders with items and customers, DataLoader collects all 50 order IDs and all 50 customer IDs, then fires 2 batched queries. Same result as eager loading (3 queries total), but only for fields the client actually requested.

**Performance comparison:**

| Approach | Queries for 50 orders | Over-fetching risk | Complexity |
|----------|----------------------|-------------------|------------|
| N+1 (broken) | 101 | None | Low |
| Eager loading | 2-3 | Yes (loads relations even if unused) | Low |
| DataLoader | 2-3 (only for requested fields) | None | Medium |

**When to choose denormalization instead:**

Denormalization (storing `itemCount`, `totalAmount`, `customerName` directly on the order row) makes sense when:
- The read pattern is always the same (list view always needs the same summary fields)
- The related data rarely changes after creation (order items are immutable once placed)
- You need sub-millisecond reads at extreme scale and even 3 queries is too many
- The consistency tradeoff is acceptable (customer name change doesn't retroactively update old orders -- which may actually be correct behavior)

Don't denormalize when the source data changes frequently, when you need many different views of the data, or when the write-side complexity of keeping denormalized fields in sync outweighs the read-side benefit.

</details>

<details>
<summary>16. You have a slow PostgreSQL query — walk through interpreting the EXPLAIN ANALYZE output step by step, showing what Seq Scan, Index Scan, and Bitmap Heap Scan mean, how to read the actual vs estimated rows and cost, and demonstrate choosing the right index strategy (B-tree, partial index, composite index) to fix a specific slow query pattern</summary>

**Scenario:** A slow query fetching recent orders for a specific customer:

```sql
SELECT * FROM orders WHERE customer_id = 'abc-123' AND status = 'completed' ORDER BY created_at DESC LIMIT 20;
```

**Step 1: Run EXPLAIN ANALYZE**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 'abc-123' AND status = 'completed' ORDER BY created_at DESC LIMIT 20;
```

**Step 2: Interpret the output**

```
Limit  (cost=45000.12..45000.17 rows=20 width=250) (actual time=1245.332..1245.340 rows=20 loops=1)
  -> Sort  (cost=45000.12..45125.45 rows=50132 width=250) (actual time=1245.330..1245.335 rows=20 loops=1)
        Sort Key: created_at DESC
        Sort Method: top-N heapsort  Memory: 30kB
        -> Seq Scan on orders  (cost=0.00..42500.00 rows=50132 width=250) (actual time=0.025..1180.456 rows=48750 loops=1)
              Filter: ((customer_id = 'abc-123') AND (status = 'completed'))
              Rows Removed by Filter: 4951250
              Buffers: shared hit=12000 read=18500
Planning Time: 0.15 ms
Execution Time: 1245.52 ms
```

**Reading the key fields:**

- **Seq Scan** -- PostgreSQL is reading the entire `orders` table (5M rows) sequentially. This is the main problem. It scanned all rows and filtered out 4.95M of them to find 48,750 matches.
- **cost=0.00..42500.00** -- estimated startup cost..total cost in arbitrary units. Higher = more expensive.
- **rows=50132 (actual rows=48750)** -- PostgreSQL estimated 50K rows but found 48.7K. Close enough -- statistics are accurate. A big mismatch here (estimated 100, actual 50000) means stale statistics (`ANALYZE` needed).
- **Buffers: shared hit=12000 read=18500** -- 12K pages from cache, 18.5K from disk. Heavy disk I/O.
- **Sort** -- after finding 48.7K matching rows, it sorts them all to find the top 20. Wasteful when an index could provide pre-sorted results.

**Scan types explained:**

| Scan Type | What it does | When PostgreSQL uses it |
|-----------|-------------|----------------------|
| **Seq Scan** | Reads every row in the table | No useful index exists, or the query matches a large percentage of rows (>5-10%) where an index wouldn't help |
| **Index Scan** | Looks up the index, then fetches each matching row from the heap (table) | Highly selective queries returning few rows, the index covers the filter columns |
| **Bitmap Heap Scan** | Builds a bitmap of matching pages from the index, then reads those pages in order | Medium selectivity (too many rows for index scan, too few for seq scan). Combines multiple indexes efficiently via BitmapAnd/BitmapOr |
| **Index Only Scan** | Reads only from the index, never touches the table | All requested columns are in the index (covering index) and the visibility map is up to date |

**Step 3: Choose the right index**

**Option A: Simple B-tree on the filter column**
```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
```
Helps find rows for one customer, but PostgreSQL still needs to filter by status and sort by created_at. Partial improvement.

**Option B: Composite index matching the query pattern**
```sql
CREATE INDEX idx_orders_customer_status_created
  ON orders (customer_id, status, created_at DESC);
```
PostgreSQL can seek directly to `customer_id = 'abc-123' AND status = 'completed'`, then read rows already sorted by `created_at DESC`. The `LIMIT 20` means it stops after 20 rows -- no sorting, no scanning thousands of matches. This is the optimal index for this query.

**Option C: Partial index (if most queries filter on the same status)**
```sql
CREATE INDEX idx_orders_completed
  ON orders (customer_id, created_at DESC)
  WHERE status = 'completed';
```
Smaller index (only includes completed orders), faster to scan, less storage. Perfect if `status = 'completed'` is the dominant query pattern. Useless for queries filtering on other statuses.

**After adding the composite index, the new EXPLAIN ANALYZE:**

```
Limit  (cost=0.56..25.30 rows=20 width=250) (actual time=0.045..0.082 rows=20 loops=1)
  -> Index Scan using idx_orders_customer_status_created on orders
       (cost=0.56..62000.00 rows=50132 width=250) (actual time=0.043..0.078 rows=20 loops=1)
        Index Cond: ((customer_id = 'abc-123') AND (status = 'completed'))
        Buffers: shared hit=4
Execution Time: 0.10 ms
```

From 1245ms to 0.1ms. The index scan reads only 4 pages (the 20 matching rows) instead of 30,500 pages (the entire table).

</details>

<details>
<summary>17. Configure connection pooling for a Node.js application connecting to PostgreSQL — show the setup with PgBouncer in front of PostgreSQL plus the ORM-level pool configuration, explain how to size the pool for a Node.js process (why it's different from Java/Python), and show how you'd also configure Redis connection pooling with ioredis</summary>

**Architecture:** Application pool (ORM-level) -> PgBouncer -> PostgreSQL. Two layers of pooling, each solving a different problem.

**PgBouncer configuration (`pgbouncer.ini`):**

```ini
[databases]
mydb = host=postgres-primary port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0

# Pool mode: transaction is best for Node.js
# - session: holds connection for entire client session (wastes connections)
# - transaction: returns connection to pool after each transaction (ideal)
# - statement: returns after each statement (breaks multi-statement transactions)
pool_mode = transaction

# Max connections PgBouncer opens to PostgreSQL
default_pool_size = 20
# Extra connections allowed during temporary spikes
reserve_pool_size = 5
reserve_pool_timeout = 3

# Max total client connections PgBouncer accepts
max_client_conn = 400

# Kill idle server connections after 10 minutes
server_idle_timeout = 600
```

**Why `transaction` mode:** Node.js doesn't hold connections during "thinking time" like Java threads do. It sends a query, releases the connection, and picks it up again for the next query. Transaction mode matches this behavior -- the connection returns to the pool between transactions, maximizing utilization.

**ORM-level pool (Prisma):**

```typescript
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = "postgresql://user:pass@pgbouncer-host:6432/mydb"
}
```

```typescript
// Prisma manages its own connection pool internally
// Configure via connection string parameters or environment variables
// DATABASE_URL="postgresql://user:pass@pgbouncer:6432/mydb?connection_limit=10&pool_timeout=10"
```

**Pool sizing for Node.js (why it's different):**

In Java with 200 threads, each thread holds a connection for the duration of a request. You need pool_size >= 200 or threads block.

Node.js has one thread. It can have 500 concurrent requests in flight, but only ~5-15 queries *actively executing* at any moment (the rest are waiting for I/O callbacks). So:

- **ORM pool per process:** 5-15 connections. Start with 10, measure.
- **PgBouncer `default_pool_size`:** Set to PostgreSQL's `max_connections` minus headroom (monitoring, migrations, admin). If `max_connections = 100`, set PgBouncer to 60-80.
- **Total math:** 4 Node.js instances x 10 connections each = 40 connections to PgBouncer. PgBouncer multiplexes these onto 20 real PostgreSQL connections. You serve 4 instances with only 20 database connections.

**Redis connection pooling with ioredis:**

ioredis uses a single connection by default (Redis is single-threaded, so pipelining on one connection is efficient). For high-throughput scenarios, use a cluster of connections:

```typescript
import Redis from 'ioredis';

// Single connection with automatic pipelining (sufficient for most cases)
const redis = new Redis({
  host: 'redis-host',
  port: 6379,
  maxRetriesPerRequest: 3,
  retryStrategy(times) {
    return Math.min(times * 50, 2000); // exponential backoff, max 2s
  },
  // Enable auto-pipelining: batches commands sent in the same event loop tick
  enableAutoPipelining: true,
});

// For truly high-throughput: connection pool using generic-pool or multiple instances
import { Cluster } from 'ioredis';

// Redis Cluster handles pooling across shards automatically
const cluster = new Cluster(
  [{ host: 'redis-node-1', port: 6379 }],
  {
    scaleReads: 'replica',       // read from replicas
    enableAutoPipelining: true,
    redisOptions: {
      maxRetriesPerRequest: 3,
    },
  },
);
```

**Key difference from PostgreSQL:** Redis rarely needs explicit connection pooling because auto-pipelining on a single connection handles enormous throughput. You add connections only when you need to parallelize across multiple CPU cores or separate blocking commands (like `BLPOP`) from normal commands.

</details>

## Practical — Performance Engineering

<details>
<summary>18. Refactor a synchronous API endpoint that sends an email, generates a PDF, and updates an analytics service into an async queue-based architecture — show the before and after code using BullMQ, explain what guarantees you lose by going async, and how you communicate the eventual result back to the client</summary>

**Before: Synchronous (800ms+ response time)**

```typescript
app.post('/orders/:id/confirm', async (req, res) => {
  const order = await prisma.order.update({
    where: { id: req.params.id },
    data: { status: 'confirmed' },
  }); // 50ms

  await emailService.sendConfirmation(order);        // 200ms
  const pdfUrl = await pdfService.generateInvoice(order); // 400ms
  await analyticsService.trackOrderConfirmed(order);  // 150ms

  res.json({ order, invoiceUrl: pdfUrl }); // Total: ~800ms
});
```

The user waits 800ms even though only the database update matters for the core operation.

**After: Async with BullMQ**

```typescript
import { Queue, Worker } from 'bullmq';
import Redis from 'ioredis';

const connection = new Redis({ host: 'redis-host', maxRetriesPerRequest: null });

// Define queues
const emailQueue = new Queue('email', { connection });
const pdfQueue = new Queue('pdf', { connection });
const analyticsQueue = new Queue('analytics', { connection });

// Endpoint — only does the core operation, enqueues the rest
app.post('/orders/:id/confirm', async (req, res) => {
  const order = await prisma.order.update({
    where: { id: req.params.id },
    data: { status: 'confirmed' },
  }); // 50ms

  // Enqueue side effects — each takes <5ms to enqueue
  await Promise.all([
    emailQueue.add('order-confirmation', { orderId: order.id }, {
      attempts: 3,
      backoff: { type: 'exponential', delay: 1000 },
    }),
    pdfQueue.add('invoice', { orderId: order.id }, {
      attempts: 3,
      backoff: { type: 'exponential', delay: 1000 },
    }),
    analyticsQueue.add('order-confirmed', { orderId: order.id }, {
      attempts: 3,
      backoff: { type: 'exponential', delay: 1000 },
    }),
  ]);

  res.json({ order, invoiceStatus: 'processing' }); // Total: ~60ms
});
```

**Workers process jobs independently:**

```typescript
// PDF worker — runs in a separate process or container
const pdfWorker = new Worker(
  'pdf',
  async (job) => {
    const order = await prisma.order.findUnique({
      where: { id: job.data.orderId },
      include: { items: true, customer: true },
    });
    const pdfUrl = await pdfService.generateInvoice(order!);
    // Store the result so the client can retrieve it later
    await prisma.order.update({
      where: { id: order!.id },
      data: { invoiceUrl: pdfUrl },
    });
  },
  {
    connection,
    concurrency: 5, // process up to 5 PDFs in parallel
  },
);

pdfWorker.on('failed', (job, err) => {
  logger.error(`PDF generation failed for order ${job?.data.orderId}`, err);
  // After all retries exhausted, job moves to failed state (inspect via BullMQ dashboard)
});
```

**What guarantees you lose:**

| Guarantee | Sync | Async |
|-----------|------|-------|
| **Atomicity** | All-or-nothing within the request | Core operation succeeds but side effects may fail independently |
| **Immediate feedback** | Response includes PDF URL | Response says "processing" -- client doesn't have the PDF yet |
| **Error visibility** | Client sees the error immediately (500 response) | Client gets 200, failure happens silently in background. Needs monitoring/alerting on failed jobs |
| **Ordering** | Operations execute in defined order | Jobs may execute in any order, or concurrently |

**Communicating the eventual result back to the client:**

**Option 1: Polling endpoint**
```typescript
app.get('/orders/:id/invoice', async (req, res) => {
  const order = await prisma.order.findUnique({ where: { id: req.params.id } });
  if (order?.invoiceUrl) {
    res.json({ status: 'ready', url: order.invoiceUrl });
  } else {
    res.json({ status: 'processing', retryAfter: 5 }); // retry in 5 seconds
  }
});
```

**Option 2: WebSocket/SSE notification** -- push a message to the client when the job completes. More complex but better UX for long-running jobs.

**Option 3: Webhook callback** -- for server-to-server integrations, call back a URL when processing completes. Common in B2B APIs.

</details>

<details>
<summary>19. You have a service that makes 15 sequential HTTP calls to another microservice to build a single response — show how you'd diagnose this chatty I/O antipattern (what the traces and latency profile look like), refactor it using batching or a composite endpoint, and explain what to do when you can't change the downstream service's API</summary>

**Diagnosing from traces and metrics:**

A distributed trace for this endpoint shows a waterfall of 15 sequential spans, each to the same downstream service:

```
[Request Handler]  ─────────────────────────────────────── 780ms
  [GET /items/1]   ──── 48ms
  [GET /items/2]        ──── 52ms
  [GET /items/3]             ──── 45ms
  ...
  [GET /items/15]                                    ──── 51ms
```

Each individual call is fast (~50ms), but 15 x 50ms = 750ms of sequential network overhead. The latency profile shows:
- High p50 latency that scales linearly with the number of items
- Network time dominates total request time (even though each downstream call is fast)
- The downstream service shows high request count but low latency per request
- CPU utilization on your service is low (it's just waiting on network I/O)

**The problematic code:**

```typescript
app.get('/order-summary/:id', async (req, res) => {
  const order = await getOrder(req.params.id);

  // Chatty: 15 sequential HTTP calls
  const enrichedItems = [];
  for (const item of order.items) {
    const details = await fetch(`http://product-service/items/${item.productId}`);
    enrichedItems.push({ ...item, ...(await details.json()) });
  }

  res.json({ ...order, items: enrichedItems });
});
```

**Fix 1: Batch endpoint on downstream service (best solution)**

If you can change the downstream API, add a batch endpoint:

```typescript
// Downstream service adds:
app.post('/items/batch', async (req, res) => {
  const items = await prisma.item.findMany({
    where: { id: { in: req.body.ids } },
  });
  res.json(items);
});

// Your service now makes 1 call instead of 15:
app.get('/order-summary/:id', async (req, res) => {
  const order = await getOrder(req.params.id);

  const productIds = order.items.map((i) => i.productId);
  const response = await fetch('http://product-service/items/batch', {
    method: 'POST',
    body: JSON.stringify({ ids: productIds }),
  });
  const products = await response.json();

  const productMap = new Map(products.map((p: any) => [p.id, p]));
  const enrichedItems = order.items.map((item) => ({
    ...item,
    ...productMap.get(item.productId),
  }));

  res.json({ ...order, items: enrichedItems });
});
// Total: 2 calls (getOrder + batch) instead of 16. ~100ms instead of 780ms.
```

**Fix 2: Parallelize (when you can't change the downstream API)**

If the downstream service can't add a batch endpoint, at minimum run the calls concurrently:

```typescript
app.get('/order-summary/:id', async (req, res) => {
  const order = await getOrder(req.params.id);

  // Parallel: all 15 calls at once
  const enrichedItems = await Promise.all(
    order.items.map(async (item) => {
      const response = await fetch(
        `http://product-service/items/${item.productId}`,
      );
      return { ...item, ...(await response.json()) };
    }),
  );

  res.json({ ...order, items: enrichedItems });
});
// Now 15 calls take ~50ms (max of all calls) instead of 750ms (sum of all calls)
```

This cuts latency from 750ms to ~50ms but still puts 15x request load on the downstream service per request. Add concurrency control for safety:

```typescript
// Limit concurrent requests to avoid overwhelming the downstream service
import pLimit from 'p-limit';

const limit = pLimit(5); // max 5 concurrent requests

const enrichedItems = await Promise.all(
  order.items.map((item) =>
    limit(() =>
      fetch(`http://product-service/items/${item.productId}`).then((r) =>
        r.json(),
      ),
    ),
  ),
);
```

**Fix 3: Cache + parallel (for data that doesn't change often)**

Add a Redis cache in front of the downstream calls. For product details that change infrequently, even a 60s TTL eliminates most of the network overhead:

```typescript
async function getProductDetails(productId: string) {
  const cached = await redis.get(`product:${productId}`);
  if (cached) return JSON.parse(cached);

  const response = await fetch(`http://product-service/items/${productId}`);
  const product = await response.json();
  await redis.set(`product:${productId}`, JSON.stringify(product), 'EX', 60);
  return product;
}
```

**Which to choose:** Batch endpoint > parallel with cache > parallel with concurrency limit > sequential (never). Push for the batch endpoint -- it's a one-time investment that benefits all consumers of that API.

</details>

<details>
<summary>20. An API endpoint returns a list of 10,000 records with all fields -- show how you'd optimize this with cursor-based pagination (explain why cursor beats offset at scale), field selection (sparse fieldsets or GraphQL), and response size reduction. Demonstrate the before/after payload sizes and explain the JSON serialization cost that makes large responses a latency problem even when the database query is fast.</summary>

**The problem quantified:**

```
10,000 records x ~500 bytes each = ~5MB JSON response
- Database query: 80ms (fast — proper indexes)
- JSON.stringify: 150-300ms (blocks the event loop!)
- Network transfer: 200-500ms (even with gzip)
- Client JSON.parse: 100-200ms
Total: 530-1080ms, and most of it is NOT the database
```

**Optimization 1: Cursor-based pagination**

**Why cursor beats offset at scale:**

```sql
-- Offset: PostgreSQL must scan and discard 9,000 rows to return rows 9,001-9,020
SELECT * FROM orders ORDER BY created_at DESC OFFSET 9000 LIMIT 20;
-- Gets slower as offset increases: O(offset + limit)

-- Cursor: PostgreSQL seeks directly to the position using the index
SELECT * FROM orders WHERE created_at < '2024-01-15T10:30:00Z'
  ORDER BY created_at DESC LIMIT 20;
-- Constant time regardless of how deep you paginate: O(limit)
```

Implementation:

```typescript
app.get('/orders', async (req, res) => {
  const limit = Math.min(Number(req.query.limit) || 20, 100); // cap at 100
  const cursor = req.query.cursor as string | undefined;

  const orders = await prisma.order.findMany({
    take: limit + 1, // fetch one extra to determine if there's a next page
    ...(cursor && {
      cursor: { id: cursor },
      skip: 1, // skip the cursor itself
    }),
    orderBy: { createdAt: 'desc' },
  });

  const hasNextPage = orders.length > limit;
  const results = hasNextPage ? orders.slice(0, limit) : orders;

  res.json({
    data: results,
    pagination: {
      nextCursor: hasNextPage ? results[results.length - 1].id : null,
      hasNextPage,
    },
  });
});
```

**Optimization 2: Field selection (sparse fieldsets)**

Don't return all 30 fields when the client only needs 5:

```typescript
app.get('/orders', async (req, res) => {
  // Client specifies: GET /orders?fields=id,status,total,createdAt
  const allowedFields = ['id', 'status', 'total', 'createdAt', 'customerId'];
  const requestedFields = (req.query.fields as string)?.split(',') || allowedFields;

  const select = Object.fromEntries(
    requestedFields
      .filter((f) => allowedFields.includes(f))
      .map((f) => [f, true]),
  );

  const orders = await prisma.order.findMany({
    take: 20,
    select, // Only fetch requested columns from DB
    orderBy: { createdAt: 'desc' },
  });

  res.json({ data: orders });
});
```

This reduces both database transfer (fewer columns) and JSON serialization cost (fewer fields per object).

**Optimization 3: Response size reduction**

- **Enable gzip/brotli compression** -- typically 70-90% reduction for JSON:
  ```typescript
  import compression from 'compression';
  app.use(compression({ threshold: 1024 })); // compress responses > 1KB
  ```
- **Remove null fields** -- don't send `"middleName": null` for every record
- **Use shorter key names in high-volume APIs** (tradeoff: readability)

**Before/after payload sizes:**

| Scenario | Records | Fields | Raw JSON | Gzipped |
|----------|---------|--------|----------|---------|
| **Before** (no pagination, all fields) | 10,000 | 30 | ~5MB | ~600KB |
| **After** (paginated, selected fields) | 20 | 5 | ~4KB | ~1KB |

**JSON serialization cost explained:**

`JSON.stringify` is synchronous and CPU-bound (as covered in question 13). For large payloads:
- 1MB JSON: ~20-50ms to serialize
- 5MB JSON: ~150-300ms to serialize
- 20MB JSON: ~500ms+ to serialize

During this entire time, the Node.js event loop is blocked. No other requests can be processed. This is why a "fast database query" can still produce a slow endpoint -- the bottleneck isn't the query, it's serializing and transmitting the result.

For cases where you truly need to return large datasets (exports, bulk operations), stream the response instead of buffering:

```typescript
app.get('/orders/export', async (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.write('[');

  let first = true;
  let cursor: string | undefined;

  // Stream in batches — never hold more than 100 records in memory
  while (true) {
    const batch = await prisma.order.findMany({
      take: 100,
      ...(cursor && { cursor: { id: cursor }, skip: 1 }),
      orderBy: { id: 'asc' },
    });

    for (const order of batch) {
      res.write(`${first ? '' : ','}${JSON.stringify(order)}`);
      first = false;
    }

    if (batch.length < 100) break;
    cursor = batch[batch.length - 1].id;
  }

  res.write(']');
  res.end();
});
```

This serializes small chunks (100 records at a time), keeping event loop blocking to ~1-2ms per chunk instead of 300ms for the entire dataset.

</details>

---

## Practical — Profiling & Debugging

<details>
<summary>21. An API endpoint that was fast last month is now consistently slow (p99 went from 200ms to 2s) — walk through the systematic diagnosis process across every layer of the request lifecycle: how you'd isolate whether the problem is in DNS/TLS, the load balancer, application code, database queries, or an external dependency, showing the exact tools and commands at each step</summary>

The goal is to eliminate layers systematically, narrowing from broad to specific. Don't guess -- measure at each layer and let the data tell you where to look next.

**Step 1: Confirm the problem and scope it**

First, check whether the degradation is:
- **Endpoint-specific or service-wide?** If all endpoints are slow, the problem is infrastructure (DB, network, resource exhaustion). If only one endpoint, the problem is in that endpoint's code path.
- **Constant or intermittent?** Constant = likely data growth, missing index, or code change. Intermittent = likely resource contention, GC pauses, or noisy neighbor.
- **Correlated with a deployment?** Check the deploy timeline. If the p99 jump aligns with a release, start there.

```bash
# Check when the degradation started — correlate with deploys
# In Datadog/Grafana: overlay p99 latency graph with deployment markers
```

**Step 2: Eliminate the network layer (DNS, TLS, load balancer)**

```bash
# From a client machine, measure network-layer overhead
curl -w "dns: %{time_namelookup}s\ntcp: %{time_connect}s\ntls: %{time_appconnect}s\nttfb: %{time_starttransfer}s\ntotal: %{time_total}s\n" \
  -o /dev/null -s https://api.example.com/slow-endpoint

# Expected output if network is fine:
# dns: 0.004s | tcp: 0.012s | tls: 0.035s | ttfb: 1.950s | total: 1.960s
# TTFB is high → problem is server-side, not network
```

If DNS or TLS is high (>100ms), check DNS resolver configuration, certificate chain issues, or OCSP stapling. If TTFB minus TLS is small, the problem is before your application.

Check load balancer health: are requests being routed to unhealthy instances? Are any instances showing higher latency than others?

**Step 3: Check application-level metrics**

```bash
# Look at these in your APM dashboard:
# 1. Event loop lag — is the Node.js process CPU-blocked?
# 2. Memory usage — approaching heap limit? Frequent GC pauses?
# 3. Active connection count — pool near capacity?
# 4. Error rate — are retries inflating request count?
```

If event loop lag is high (>50ms), the problem is CPU-bound code blocking the main thread (as covered in question 13). Profile with:

```bash
# Generate a CPU flame graph
node --prof app.js
# After collecting samples:
node --prof-process isolate-*.log > profile.txt
# Or use clinic.js for a visual flame graph:
npx clinic flame -- node app.js
```

**Step 4: Examine distributed traces for the slow endpoint**

Pull traces from Datadog/Jaeger for the specific endpoint. Look at the span breakdown:

```
[GET /slow-endpoint]  ──────────────────────────── 1950ms
  [middleware:auth]     ── 15ms
  [db:findOrders]            ──────────── 1800ms    ← bottleneck
  [serialize:json]                              ── 120ms
```

The trace immediately tells you: 92% of time is in the database query. Now dig into that.

**Step 5: Investigate the database layer**

```sql
-- Check for slow queries on this table
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Run EXPLAIN ANALYZE on the suspected query (as covered in question 16)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE ...;
```

Common findings at this stage:
- **Table grew significantly** -- a query that was fast with 100K rows now scans 5M rows. Missing index or the existing index is no longer selective enough.
- **Lock contention** -- check `pg_stat_activity` for blocked queries:
  ```sql
  SELECT pid, state, wait_event_type, wait_event, query
  FROM pg_stat_activity
  WHERE state != 'idle';
  ```
- **Connection pool exhaustion** -- application waits for a connection, not for the query (covered in detail in question 23).

**Step 6: Check external dependencies**

If the trace shows time spent in external HTTP calls:

```bash
# Check if the external service's latency changed
# Look at outbound HTTP spans in traces
# Check the external service's status page
# Test directly:
curl -w "ttfb: %{time_starttransfer}s\n" -o /dev/null -s https://external-api.com/endpoint
```

**Step 7: Validate with before/after comparison**

Once you identify and fix the root cause (add an index, fix an N+1, add a timeout on an external call), validate using the same measurement approach from question 1: run the same load test or compare production p99 before and after the fix with at least a few hours of traffic.

</details>

<details>
<summary>22. After a deployment, your service starts showing exponentially increasing latency and eventually cascading failures across dependent services — diagnose whether this is a retry storm or thundering herd, show how each pattern manifests differently in metrics and logs, and demonstrate the specific fixes (exponential backoff with jitter, circuit breaking, request coalescing) with code examples</summary>

**Distinguishing the two patterns** (building on the conceptual comparison in question 12):

**Retry storm signature:**
```
Timeline:
t=0:  Deploy lands. New bug causes 10% of requests to fail (500 errors).
t=30s: Clients retry failed requests. Request volume to your service increases 1.3x.
t=60s: Extra load causes more failures (20%). Volume increases 1.6x.
t=90s: 40% failures. Volume 2x. Downstream services start failing too.
t=120s: Full cascade. Everything is timing out.

Metrics:
- Inbound request rate: exponential growth (not from real users — from retries)
- Error rate on downstream: climbs steadily, then spikes
- Logs: same request IDs appearing 3-5 times with increasing timestamps
- Request distribution: skewed — certain endpoints/operations heavily retried
```

**Thundering herd signature:**
```
Timeline:
t=0:  Deploy causes a rolling restart across 20 pods.
t=10s: All 20 pods start simultaneously, each opening 10 DB connections.
t=11s: 200 connections hit PostgreSQL at once (max_connections = 100).
t=12s: Half the pods fail to connect. Health checks fail. Pods restart again.
t=20s: Restart loop — the system never stabilizes.

Metrics:
- Connection count: massive spike at a single point in time
- Error rate: instant spike across ALL instances simultaneously
- Logs: many different request IDs, all with the same timestamp
- Request distribution: uniform — everything fails, not just certain operations
```

**Fix 1: Exponential backoff with jitter (for retry storms)**

```typescript
async function fetchWithRetry(
  url: string,
  options: RequestInit,
  maxRetries = 3,
): Promise<Response> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, {
        ...options,
        signal: AbortSignal.timeout(5000), // always set a timeout
      });

      if (response.ok) return response;

      // Only retry on 5xx (server errors), not 4xx (client errors)
      if (response.status < 500) return response;
    } catch (error) {
      if (attempt === maxRetries) throw error;
    }

    // Exponential backoff: 1s, 2s, 4s + random jitter up to 1s
    // Jitter prevents synchronized retries across clients
    const baseDelay = Math.pow(2, attempt) * 1000;
    const jitter = Math.random() * 1000;
    await new Promise((r) => setTimeout(r, baseDelay + jitter));
  }

  throw new Error(`Failed after ${maxRetries} retries`);
}
```

Without jitter, all clients that failed at t=0 would retry at exactly t=1s, then t=2s -- creating synchronized waves. Jitter spreads retries over time, preventing the amplification loop.

**Fix 2: Circuit breaker (stop calling a failing service)**

```typescript
class CircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private threshold = 5,       // open after 5 failures
    private resetTimeout = 30000, // try again after 30s
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailure > this.resetTimeout) {
        this.state = 'half-open'; // allow one test request
      } else {
        throw new Error('Circuit breaker is open — fast-failing');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failures = 0;
    this.state = 'closed';
  }

  private onFailure() {
    this.failures++;
    this.lastFailure = Date.now();
    if (this.failures >= this.threshold) {
      this.state = 'open'; // stop sending traffic to failing service
    }
  }
}

// Usage
const paymentCircuit = new CircuitBreaker(5, 30000);

async function processPayment(order: Order) {
  return paymentCircuit.execute(() =>
    fetch('http://payment-service/charge', {
      method: 'POST',
      body: JSON.stringify(order),
    }),
  );
}
```

The circuit breaker stops the cascade: instead of every request waiting 5s for a timeout on a dead service, requests fail immediately (fast-fail). This preserves resources and prevents the retry amplification loop.

**Fix 3: Request coalescing (for thundering herd on recovery)**

When the service recovers, prevent all instances from simultaneously hammering a shared resource. The singleflight pattern from question 5 applies here -- at the infrastructure level, implement staggered startup:

```yaml
# Kubernetes: rolling restart with controlled pace
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1        # only 1 new pod at a time
      maxUnavailable: 0  # never remove a pod until the new one is ready
  template:
    spec:
      containers:
        - name: api
          startupProbe:
            httpGet:
              path: /health
            initialDelaySeconds: 5    # wait 5s before first probe
            periodSeconds: 3
            failureThreshold: 10      # give it 30s to warm up
```

Additionally, add connection pool pre-warming with startup delay jitter:

```typescript
// Stagger startup across pods to prevent connection thundering herd
const startupJitter = Math.random() * 5000; // 0-5s random delay
await new Promise((r) => setTimeout(r, startupJitter));
await prisma.$connect(); // now open DB connections
```

</details>

<details>
<summary>23. Your Node.js service is intermittently timing out on database calls even though the database server shows low CPU and memory — walk through diagnosing connection pool exhaustion: what symptoms to look for in application metrics, how to confirm the pool is the bottleneck (vs slow queries or network issues), the exact PgBouncer and ORM pool settings to inspect, and how to fix it without just blindly increasing the pool size</summary>

**The misleading symptom:** Database timeouts usually make you look at the database. But low CPU and memory on the DB server means the database itself is fine -- requests aren't reaching it, or they're queuing before they get there. This is the classic connection pool exhaustion pattern.

**Step 1: Look for pool exhaustion symptoms in application metrics**

| Metric | Healthy | Pool exhaustion |
|--------|---------|----------------|
| DB query latency (p50) | 5ms | 5ms (queries themselves are still fast!) |
| DB query latency (p99) | 20ms | 15,000ms (waiting for a connection) |
| Active pool connections | 3-5 out of 10 | 10 out of 10 (maxed out) |
| Pending pool requests | 0 | 50+ (queued, waiting) |
| Application error logs | None | `Timed out waiting for pool connection` |

The key tell: p50 latency is normal but p99 is catastrophically high. The queries are fast once they run -- the wait is for a connection.

**Step 2: Confirm it's the pool, not slow queries or network**

```sql
-- On PostgreSQL: are queries actually slow?
SELECT pid, state, wait_event_type, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE datname = 'mydb'
ORDER BY duration DESC;

-- If most connections show 'idle' or 'idle in transaction' — the queries finished
-- but connections aren't being released. That's pool mismanagement, not slow queries.
```

```bash
# Network test: is there latency between app and DB?
# From the application container:
psql -h pgbouncer-host -p 6432 -c "SELECT 1" --timing
# If this returns in <5ms, network is fine
```

**Step 3: Inspect PgBouncer state**

```bash
# Connect to PgBouncer admin console
psql -h pgbouncer-host -p 6432 pgbouncer

# Show pool status
SHOW POOLS;
# Key columns:
# cl_active  — client connections actively using a server connection
# cl_waiting — client connections waiting for a server connection (should be 0!)
# sv_active  — server connections executing a query
# sv_idle    — server connections idle in pool
# sv_used    — server connections returned to pool but not yet tested

SHOW STATS;
# avg_wait_time — average time clients wait for a connection (should be <1ms)
```

If `cl_waiting > 0` and `avg_wait_time` is high, PgBouncer is the bottleneck. If PgBouncer shows `sv_idle > 0` (idle server connections available), the bottleneck is between PgBouncer and your ORM -- the ORM pool is full before PgBouncer's pool is.

**Step 4: Inspect ORM pool settings**

```
# Prisma connection string parameters:
DATABASE_URL="postgresql://user:pass@pgbouncer:6432/mydb?connection_limit=10&pool_timeout=10"

# connection_limit: max connections in Prisma's pool (default: num_cpus * 2 + 1)
# pool_timeout: seconds to wait for a connection before throwing (default: 10)
```

Check what the defaults resolve to. On a 4-core machine, Prisma defaults to 9 connections. If you have 2 instances, that's 18 connections to PgBouncer.

**Step 5: Find the root cause (don't just increase the pool)**

Blindly increasing the pool size treats the symptom, not the cause. Common root causes:

**A. Long-running transactions holding connections:**
```sql
-- Find connections stuck in transaction
SELECT pid, state, now() - xact_start AS tx_duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY tx_duration DESC;
```
Fix: Add `idle_in_transaction_session_timeout` in PostgreSQL, or find the application code that starts a transaction but doesn't commit/rollback promptly (often a `prisma.$transaction()` that makes HTTP calls inside it).

**B. Connection leak -- code acquires connections without releasing:**
```typescript
// BAD: Manual connection that might not be released on error
const client = await pool.connect();
const result = await client.query('SELECT ...');
// If this throws, client.release() never runs
await someOtherOperation();
client.release();

// GOOD: Always release in finally
const client = await pool.connect();
try {
  const result = await client.query('SELECT ...');
  await someOtherOperation();
} finally {
  client.release();
}
```

With Prisma, connection leaks are rare since Prisma manages connections internally. But `$transaction` blocks that never resolve will hold connections indefinitely.

**C. Burst traffic exceeding pool capacity:**
If the pool is sized correctly for steady state but too small for traffic spikes, add connection queuing with bounded wait time rather than increasing pool size:

```
# In Prisma: increase pool_timeout to allow brief queuing during spikes
DATABASE_URL="...?connection_limit=10&pool_timeout=15"

# In PgBouncer: add reserve pool for spikes
reserve_pool_size = 5
reserve_pool_timeout = 3
```

**The fix hierarchy:**
1. Find and fix connection leaks / long transactions (root cause)
2. Verify pool size is appropriate for Node.js (5-15 per process, as covered in question 9)
3. Add PgBouncer if not present (multiplexes connections across instances)
4. Add connection queue monitoring and alerting (catch exhaustion before it causes timeouts)
5. Only then consider increasing pool size -- and only if metrics show the pool is genuinely too small for your query concurrency

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you diagnosed and fixed a significant production performance issue — what were the symptoms, how did you identify the root cause (not just the surface-level fix), and how did you validate that your fix actually worked under real traffic?</summary>

**What the interviewer is looking for:**
- Systematic debugging approach, not guessing or trial-and-error
- Ability to distinguish symptoms from root causes
- Use of data and tooling to drive decisions (traces, metrics, profiling)
- Validation discipline -- proving the fix worked, not assuming

**Suggested structure (STAR with technical depth):**

1. **Situation** (2 sentences): What system, what scale, what mattered about it
2. **Symptoms** (specific): What alerts fired, what metrics moved, what users experienced. Use numbers.
3. **Investigation process** (this is the bulk): Walk through your debugging steps in order. What did you check first? What did you eliminate? What tool gave you the breakthrough insight? Show your reasoning at each step.
4. **Root cause** (be precise): Not "the database was slow" but "a missing composite index on (tenant_id, created_at) caused a sequential scan on a table that grew from 500K to 12M rows over 3 months"
5. **Fix and validation**: What you changed, how you verified it worked (before/after metrics under real traffic), and any follow-up actions (monitoring, prevention)

**Example outline to personalize:**

> Our order processing service started showing p99 latency spikes to 8 seconds during peak hours (normally 200ms). PagerDuty fired on our SLA breach alert.
>
> I started by checking if it correlated with a deploy -- it didn't; the degradation was gradual over weeks. Traces showed 85% of time in a single database span. EXPLAIN ANALYZE revealed a Seq Scan on the orders table -- the table had grown 10x in the past quarter and the existing single-column index was no longer selective enough for our multi-filter query pattern.
>
> I added a composite index matching the exact query pattern. p99 dropped from 8s to 150ms within minutes of the migration completing. I set up a recurring query performance review to catch this class of problem before it hits production.

**Key points to hit:**
- Show you used observability tools, not guesswork
- Demonstrate you looked beyond the surface (not just "added an index" but "understood why the existing index stopped being effective")
- Include the validation step -- interviewers specifically listen for whether you confirmed the fix worked

</details>

<details>
<summary>25. Describe a time you designed and implemented a caching strategy for a system — what drove the decision, what layer(s) did you cache at, how did you handle invalidation, and what unexpected problems did caching introduce?</summary>

**What the interviewer is looking for:**
- Thoughtful design process -- why caching was the right solution (not premature optimization)
- Understanding of caching layers and why you chose specific ones
- Realistic handling of invalidation (the hard part of caching)
- Intellectual honesty about problems caching introduced -- this is where senior-level thinking shows

**Suggested structure:**

1. **What drove the decision** (data, not instinct): "Our product detail API had p95 of 400ms. Profiling showed 80% of time was in 3 database queries that returned identical data for 95% of requests. Cache hit rate analysis showed the top 1000 products accounted for 85% of traffic."
2. **Design choices**: Which layer(s) and why. Explain what you considered and rejected. "We used Redis (not CDN) because responses were personalized by user's price group -- only 5 groups, so 5 cache entries per product, not one per user."
3. **Invalidation approach**: TTL? Event-based? Why that choice? "We used event-based invalidation via our existing message bus -- product updates publish an event, the API service subscribes and deletes the cache key. TTL of 5 minutes as a safety net for missed events."
4. **Unexpected problems**: This is the differentiator. Examples:
   - Cache warming after deploy caused a brief stampede
   - Stale data from a race condition between the update event and the cache write
   - Debugging became harder because developers couldn't tell if they were seeing cached or fresh data
   - Memory growth from caching more data than expected
   - The invalidation event bus had its own reliability issues

**Example outline to personalize:**

> Our catalog API served 2000 req/s with p95 of 450ms. Analysis showed the same 500 products accounted for 90% of requests, and the data changed only a few times per day.
>
> I implemented a Redis cache-aside pattern with event-based invalidation. Cache hit rate reached 94%, p95 dropped to 35ms.
>
> The unexpected problem: during a bulk product import (rare but real), 500 invalidation events fired within seconds. Each miss triggered a database query, causing a brief load spike. We added request coalescing to handle this -- only one in-flight recomputation per key.

**Key points to hit:**
- Show the decision was data-driven (cache hit rate analysis, not "caching makes things faster")
- Demonstrate you thought about invalidation upfront, not as an afterthought
- The "unexpected problems" section is what separates a senior answer from a mid-level one -- show you learned something from the implementation

</details>

<details>
<summary>26. Describe a time you had to make a deliberate decision NOT to optimize something — what was the performance problem, why did you decide the optimization wasn't worth it, and how did you communicate that tradeoff to stakeholders?</summary>

**What the interviewer is looking for:**
- Engineering judgment -- knowing when NOT to act is as important as knowing how to act
- Cost-benefit analysis applied to technical work (as discussed in question 12's "premature optimization" section)
- Ability to communicate technical tradeoffs to non-technical stakeholders
- Confidence to push back when optimization pressure comes from the wrong place

**Suggested structure:**

1. **The problem** (specific): "Our reporting endpoint took 3 seconds. A PM flagged it as a performance issue."
2. **Your analysis** (data-driven): "I measured: it was called 12 times per day, by 3 internal users, for monthly reports. No SLA. No customer impact. The 3 seconds came from aggregating 6 months of data -- fundamentally expensive work."
3. **Why you decided against optimizing**: Frame it as resource allocation. "The optimization would require denormalizing the reporting schema and adding a materialized view refresh pipeline -- 2 weeks of work plus ongoing maintenance. For 12 requests per day with no customer impact, the ROI was negative."
4. **What you did instead** (if anything): "I added a loading indicator so the 3 users knew to wait, and documented it as a known limitation. If usage grows to 100+ queries per day, we'd revisit."
5. **How you communicated it**: "I showed the PM the usage numbers and the engineering cost. I framed it as 'two weeks we could spend on X instead' -- making the opportunity cost concrete."

**Example outline to personalize:**

> Our admin dashboard's analytics page loaded in 4 seconds. The product team wanted it under 1 second.
>
> Analysis showed: 8 users, used twice daily, internal-only tool. The slow query aggregated across millions of rows -- optimizing it required either a pre-computed materialized view (complex to maintain, 1 week to build) or denormalization (risk of stale dashboard data).
>
> I proposed keeping the current performance and spending that engineering time on the customer-facing search performance issue (p95 of 800ms, 50K daily users). I presented both options with estimated effort and impact. The PM agreed.
>
> Three months later, dashboard usage hadn't grown. The decision held up.

**Key points to hit:**
- Demonstrate you did the analysis (you measured, you estimated the cost) -- you didn't just say "it's fine"
- Show you offered an alternative use of the engineering time (not just "no")
- Mention follow-up: did you set a threshold for revisiting the decision?
- Avoid framing it as laziness -- frame it as disciplined resource allocation

</details>
