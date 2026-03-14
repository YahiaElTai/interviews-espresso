# Scaling & Reliability

> **26 questions** — 13 theory, 13 practical

- Horizontal vs vertical scaling — tradeoffs, hidden costs of horizontal (state management, coordination, consistency, DB connection exhaustion, cache stampede), when vertical wins
- Stateless service design — shared-nothing architecture, externalizing session state to Redis/database, why statelessness is a prerequisite for horizontal scaling
- Database scaling — read replicas (routing reads vs writes, replication lag handling), partitioning, sharding as last resort
- Multi-region scaling — active-active vs active-passive, latency-based routing, regional failover, data replication challenges across regions
- Capacity planning — traffic estimation, identifying scaling triggers (CPU, queue depth, latency), headroom planning, when to scale proactively vs reactively
- Reliability as a design concern — redundancy (no single points of failure), fault isolation (blast radius containment, bulkhead boundaries), designing for failure from day one vs bolting it on
- Graceful degradation — feature flags, fallback responses, read-only mode, queue shedding, fallback chains, per-dependency timeout and circuit breaker placement
- Resilience patterns in practice — retry amplification across layers; retry budgets; when patterns hurt more than help
- Backpressure vs rate limiting — upstream signal propagation, load shedding during saturation, queue depth as backpressure signal
- Deployment reliability — graceful shutdown (SIGTERM handling, connection draining), readiness signaling, zero-downtime checklist, common rollout failure modes (502 spikes, premature termination)

---

## Foundational

<details>
<summary>1. How do scaling and reliability relate to each other as engineering concerns — why doesn't simply adding more instances automatically make a system more reliable, what new failure modes does horizontal scaling introduce, and how should a senior engineer think about the relationship between capacity, redundancy, and fault tolerance when designing a system?</summary>

Scaling and reliability are related but independent axes. Scaling addresses capacity (can you handle the load), while reliability addresses correctness under failure (does the system keep working when things break). Adding instances increases capacity and provides some redundancy, but it doesn't automatically improve reliability — it can actively hurt it.

**Why more instances does not equal more reliable:**

- More instances means more network hops, more processes that can fail, more coordination needed. The probability of *at least one* instance failing at any given moment increases with count.
- If all instances share a single database, scaling instances just moves the bottleneck — the database becomes the single point of failure under higher load.
- If the instances aren't truly stateless, scaling introduces split-brain scenarios, stale caches, and inconsistent behavior depending on which instance handles a request.

**New failure modes horizontal scaling introduces:**

- **Database connection exhaustion**: 50 instances × 20 connections per pool = 1,000 connections, potentially exceeding the database's max_connections.
- **Cache stampede**: New instances start with cold caches. All of them hit the database simultaneously for the same data.
- **Coordination overhead**: Distributed locks, leader election, and consensus become necessary for work that was trivial in a single process.
- **Consistency challenges**: Data that was ACID-consistent in a single process now requires distributed transaction patterns (sagas, eventual consistency).
- **Deployment complexity**: Rolling updates across 50 instances means two versions of code running simultaneously, requiring backward-compatible changes.

**How to think about the relationship:**

- **Capacity** answers "can we handle the traffic?" — solved by scaling (horizontal or vertical).
- **Redundancy** answers "if one component dies, do we survive?" — solved by eliminating single points of failure (multiple instances, multi-AZ, replicated databases).
- **Fault tolerance** answers "when things break, does the system degrade gracefully?" — solved by circuit breakers, timeouts, fallback responses, and isolation.

Redundancy is a subset of reliability, and scaling can provide redundancy as a side effect, but scaling without fault tolerance just means you have more things that can break in correlated ways. A senior engineer designs for all three independently: scale for load, add redundancy for survival, and build fault tolerance for graceful degradation.

</details>

<details>
<summary>2. What are the real tradeoffs between horizontal and vertical scaling — beyond the textbook answer of "vertical has limits," what are the hidden costs of horizontal scaling (state management, coordination overhead, consistency challenges, database connection exhaustion, cache stampedes), and when does vertical scaling actually win over horizontal in practice?</summary>

**Vertical scaling (scaling up):**

Pros:
- Zero architectural changes — same code, bigger machine
- No distributed system complexity — no coordination, no consistency problems
- Simpler operations — one machine to monitor, debug, and maintain
- Strong consistency is trivial — everything is in-process

Cons:
- Hardware ceiling (you can get very far — 128 cores, 2TB RAM — but there is a limit)
- Single point of failure unless combined with standby replicas
- Cost curve is non-linear — the last 2x of compute costs disproportionately more
- Downtime during upgrade (unless using live migration)

**Horizontal scaling (scaling out):**

Pros:
- Theoretically unlimited capacity
- Natural redundancy (losing one instance doesn't lose the service)
- Cost-efficient at scale (commodity hardware)
- Enables per-component scaling (scale the hot path independently)

Hidden costs:
- **State management**: Any in-memory state (sessions, caches, local files) must be externalized. This adds latency (network hop to Redis/DB) and a new dependency.
- **Coordination overhead**: Distributed locks, leader election, and ordering guarantees that were free in a single process now require explicit infrastructure.
- **Consistency challenges**: Operations that were ACID transactions become distributed — requiring sagas, idempotency keys, and eventual consistency reasoning.
- **Database connection exhaustion**: `instances × pool_size` can easily exceed database limits. You need connection poolers like PgBouncer as a new operational layer.
- **Cache stampede**: During scale-out, new instances have cold caches. They all query the database for the same hot data simultaneously, creating a spike that can overwhelm the DB.
- **Deployment complexity**: Rolling updates mean multiple versions coexist. APIs must be backward-compatible. Database migrations must be non-breaking.
- **Observability complexity**: Logs are spread across instances. Debugging requires correlation IDs and distributed tracing.

**When vertical scaling wins:**

- **Databases**: A bigger database server is almost always preferable to sharding. Sharding introduces application-level routing, cross-shard query complexity, and rebalancing headaches.
- **Low instance count**: If 1-3 large instances handle your load, vertical is simpler and cheaper than 20 small ones with all the distributed overhead.
- **CPU/memory-bound workloads with shared state**: Image processing, data analysis, or in-memory computation where the workload doesn't partition cleanly.
- **Early-stage products**: Traffic is unpredictable. Scaling vertically buys time to understand traffic patterns before committing to horizontal architecture.
- **Latency-sensitive paths**: In-process calls are nanoseconds; network calls are milliseconds. If every microsecond matters, vertical keeps everything in one process.

The pragmatic approach: scale vertically until you can't, design for horizontal from the start (stateless services, externalized state), but don't pay the horizontal tax until you need it.

</details>

<details>
<summary>3. Why should reliability be treated as a design concern rather than an afterthought — how do redundancy, fault isolation, graceful degradation, and timeouts work together as a reliability strategy, and what happens to systems that bolt these on later instead of designing for them from the start?</summary>

Reliability is an architectural property, not a feature you can bolt on. It affects how you structure services, how they communicate, how data flows, and how failures propagate. Retrofitting reliability into a system that wasn't designed for it is like adding earthquake resistance to a finished building — technically possible, far more expensive, and never as good.

**How the reliability tools work together:**

- **Redundancy** eliminates single points of failure. Multiple instances behind a load balancer, multi-AZ databases, replicated caches. This is the foundation — without it, any single failure takes down the system.
- **Fault isolation** (bulkheads) contains the blast radius. Separate connection pools per dependency, dedicated queues per workload type, isolated failure domains. When one thing breaks, it doesn't cascade to everything.
- **Timeouts** prevent indefinite waiting. Every external call gets a deadline. Without them, a slow downstream service consumes your resources indefinitely, eventually exhausting your capacity.
- **Graceful degradation** defines what happens when components fail. Fallback responses, read-only mode, feature shedding. The system stays usable, just with reduced capability.

**How they compose:**

A request comes in → hits a healthy instance (redundancy) → calls a downstream service with a timeout → the downstream is slow, times out → the circuit breaker tracks the failure (fault isolation) → after threshold, the breaker opens → subsequent requests get a fallback response (graceful degradation). The user gets a slightly degraded experience, but the system stays up.

Remove any one layer and the strategy breaks: no timeout means the breaker never trips. No isolation means one failure cascades everywhere. No degradation means a tripped breaker returns a 500 instead of a useful fallback.

**What happens when you bolt reliability on later:**

- **Timeouts can't be added safely**: Code written assuming calls always succeed now needs error handling at every call site. Missed timeout handling causes partial state corruption.
- **Stateful services resist redundancy**: Sessions stored in memory, local file writes, in-process caches with no invalidation — making these services redundant requires rearchitecting the state layer.
- **No fallback paths exist**: The code has one path — the happy path. Adding degradation means designing alternative flows that the original architecture didn't account for.
- **Coupled failure domains**: Services that share a database, a message broker, or a thread pool can't be isolated without breaking apart tightly coupled code.
- **Error handling is shallow**: `catch (e) { log(e); throw e; }` is everywhere. Real resilience requires distinguishing transient from permanent errors, deciding what to retry, and knowing when to fail fast.

The cost of retrofitting is typically 3-5x the cost of building it in. And the result is always patchwork — reliability patterns layered on top of architecture that fights them.

</details>

## Conceptual Depth

<details>
<summary>4. How do read replicas work for database scaling — how do you route reads vs writes correctly, what problems does replication lag introduce (stale reads, read-your-own-write violations), and what strategies exist to handle lag without degrading the user experience?</summary>

**How read replicas work:**

The primary database handles all writes. Changes are replicated (typically asynchronously via WAL streaming in PostgreSQL) to one or more read replicas. Your application routes write queries to the primary and read queries to replicas, distributing the read load.

**Routing reads vs writes:**

The common approaches:
1. **Application-level routing**: The code explicitly chooses which connection to use based on the operation type.
2. **Middleware/proxy routing**: A proxy like PgBouncer or ProxySQL parses queries and routes based on statement type (SELECT → replica, INSERT/UPDATE/DELETE → primary).
3. **Framework support**: ORMs like TypeORM or Prisma support read/write splitting via configuration.

```typescript
// Application-level routing example
class DatabaseRouter {
  constructor(
    private primary: Pool,    // writes
    private replicas: Pool[]  // reads
  ) {}

  getWriteConnection(): Pool {
    return this.primary;
  }

  getReadConnection(): Pool {
    // Round-robin across replicas
    const idx = Math.floor(Math.random() * this.replicas.length);
    return this.replicas[idx];
  }
}
```

**Problems replication lag introduces:**

- **Stale reads**: User updates their profile (write → primary), immediately refreshes (read → replica). The replica hasn't received the change yet. User sees old data.
- **Read-your-own-write violations**: This is the most common complaint. Any workflow where a write is immediately followed by a read of the same data will show stale results if the read hits a replica.
- **Phantom inconsistencies**: A read query joins two tables that have replicated at different speeds, producing results that never existed on the primary.

**Strategies to handle lag:**

1. **Read-from-primary after write**: After a write, route subsequent reads for that user/entity to the primary for a short window (e.g., 5 seconds). Simple and effective for most cases.

2. **Causal consistency tokens**: The primary returns a log sequence number (LSN) after a write. The client sends this LSN with subsequent reads. The replica waits until it has replicated past that LSN before responding.

3. **Sticky sessions**: After a write, pin the user's session to the primary for reads. Simpler but creates uneven load distribution.

4. **Monitoring replication lag**: Track lag metrics and route reads to the primary if lag exceeds a threshold.

```typescript
// Read-from-primary-after-write pattern
class SmartRouter {
  private recentWrites = new Map<string, number>(); // entityId → timestamp
  private readonly STALE_WINDOW_MS = 5000;

  async query(sql: string, entityId: string, isWrite: boolean): Promise<any> {
    if (isWrite) {
      this.recentWrites.set(entityId, Date.now());
      return this.primary.query(sql);
    }

    // If we recently wrote this entity, read from primary
    const lastWrite = this.recentWrites.get(entityId);
    if (lastWrite && Date.now() - lastWrite < this.STALE_WINDOW_MS) {
      return this.primary.query(sql);
    }

    return this.getReadConnection().query(sql);
  }
}
```

**Practical recommendation**: Start with read-from-primary-after-write for critical flows (user profile updates, order placement) and accept eventual consistency for non-critical reads (dashboards, analytics, listing pages). Most applications have a small set of write-then-read flows that need special handling — don't over-engineer the general case.

</details>

<details>
<summary>5. What is the difference between partitioning and sharding for database scaling, why is sharding considered a last resort, what operational complexity does it introduce (cross-shard queries, rebalancing, application-level routing), and what should you exhaust before reaching for sharding?</summary>

**Partitioning vs sharding:**

- **Partitioning** splits a large table into smaller pieces *within the same database instance*. The database engine handles it transparently — queries still go to one server, and the database routes to the correct partition internally. PostgreSQL supports range partitioning (by date), list partitioning (by region), and hash partitioning.
- **Sharding** splits data *across multiple database instances*. Each shard is a separate database, and the application (or a proxy) must know which shard holds which data. This is a distributed systems problem, not just a database optimization.

**Why sharding is a last resort:**

Sharding fundamentally changes how you interact with your data. It introduces problems that don't exist with a single database:

- **Cross-shard queries**: A query that spans multiple shards requires scatter-gather — send to all shards, collect results, merge. This is slow, complex, and can't use database-level joins.
- **Application-level routing**: Your application must know the sharding key and route each query to the correct shard. Every data access path needs awareness of the shard topology.
- **Rebalancing**: When shards become uneven (hot shards), you need to move data between shards with minimal downtime. This is one of the hardest operational challenges in database management.
- **Distributed transactions**: ACID transactions don't span shards. Any operation touching multiple shards requires two-phase commit or saga patterns.
- **Schema migrations**: Every DDL change must be applied to every shard, coordinated and monitored independently.
- **Operational overhead**: N shards means N databases to monitor, back up, tune, and failover.
- **Shard key selection is permanent**: Choosing the wrong shard key means queries that should hit one shard hit all of them. Changing the shard key later requires a full data migration.

**What to exhaust before sharding:**

1. **Vertical scaling**: Bigger database server. Modern databases handle terabytes on a single instance.
2. **Read replicas**: If reads are the bottleneck, replicas are far simpler than sharding.
3. **Connection pooling**: If connection exhaustion is the issue, PgBouncer solves it without architectural changes.
4. **Query optimization**: Proper indexes, query rewrites, and materialized views can improve performance by orders of magnitude.
5. **Table partitioning**: Time-series data partitioned by month, for example, gives partition pruning benefits without distributed system complexity.
6. **Caching**: Redis/Memcached for hot read paths reduces database load significantly.
7. **CQRS**: Separate read and write models so the read side can use a denormalized store optimized for query patterns.
8. **Archiving**: Move old data to cold storage. Many "scaling" problems are really "too much data we never query" problems.

If after exhausting all of these your single database still can't keep up, *then* consider sharding — and shard by a key that minimizes cross-shard queries (typically tenant ID or user ID in multi-tenant systems).

</details>

<details>
<summary>6. What does graceful degradation look like in practice — how do feature flags, fallback responses, read-only mode, and queue shedding each contribute to keeping a system usable under partial failure, and how do you decide which features to shed first when things start breaking?</summary>

Graceful degradation means the system stays usable with reduced capability instead of failing entirely. Each technique handles a different failure scenario.

**Feature flags for degradation:**

Feature flags let you disable non-essential functionality in real-time without deploying code. During an incident, you flip a flag to turn off expensive features.

```typescript
// Feature flag-driven degradation
async function getProductPage(productId: string) {
  const product = await productService.get(productId); // essential — always call

  const recommendations = featureFlags.isEnabled('recommendations')
    ? await recommendationService.get(productId).catch(() => [])
    : []; // skip entirely when flag is off

  const reviews = featureFlags.isEnabled('reviews')
    ? await reviewService.get(productId).catch(() => [])
    : [];

  return { product, recommendations, reviews };
}
```

**Fallback responses:**

When a dependency fails, return a cached or default response instead of an error.

- Recommendation engine down → return popular products or recently viewed items from cache
- Search service down → show category browsing instead
- User preference service down → use default settings

The key: fallback responses must be pre-planned and tested, not improvised during an incident.

**Read-only mode:**

When the write path is compromised (database failover, payment service down), disable writes while keeping reads working. Users can browse, search, and view — they just can't modify or purchase.

This requires designing the system so read and write paths can be independently disabled. If reads and writes are tightly coupled (e.g., a view counter increments on every page load), read-only mode won't work without refactoring.

**Queue shedding:**

When a message queue backs up beyond a threshold, start dropping low-priority messages. An email notification queue backed up to 100K messages? Drop marketing emails, keep transactional ones (order confirmations, password resets).

```typescript
async function processMessage(message: QueueMessage) {
  const queueDepth = await queue.getDepth();

  if (queueDepth > HIGH_WATERMARK) {
    if (message.priority === 'low') {
      await message.ack(); // drop it — shed load
      metrics.increment('messages.shed');
      return;
    }
  }

  await handleMessage(message);
}
```

**How to decide what to shed first:**

Rank features by business criticality and user impact:

| Priority | Keep running | Shed first |
|---|---|---|
| Critical | Checkout, authentication, order status | Recommendations, recently viewed |
| Important | Search, product browsing, cart | Reviews, ratings, personalization |
| Nice-to-have | User preferences, wishlists | Analytics events, marketing popups |

The decision framework:
1. **Revenue-generating flows** stay up longest (checkout > browsing > recommendations)
2. **Data-loss-risk operations** take priority over stateless views (saving a cart > showing recommendations)
3. **Features with cheap fallbacks** shed first (recommendations have a natural fallback — popular items)
4. **Features with no fallback** are either essential (keep them) or disposable (drop them)

Document the degradation tiers *before* incidents happen. During an incident is not the time to debate which features to disable.

</details>

<details>
<summary>7. Why should circuit breakers and timeouts be configured per-dependency rather than globally — how do fallback chains work when a primary dependency fails, what does proper per-dependency isolation look like in a service that calls multiple downstream APIs, and what goes wrong when teams use a single global timeout for all outbound calls?</summary>

**Why per-dependency configuration:**

Different dependencies have wildly different performance characteristics. A global 5-second timeout makes no sense when your cache should respond in 5ms and your payment provider legitimately takes 3 seconds. A global circuit breaker threshold ignores that your analytics service failing is a minor inconvenience while your database failing is critical.

**What goes wrong with global timeouts:**

- **Too generous for fast dependencies**: A 5s timeout on a Redis call means you wait 5 seconds for something that should take 5ms. A dead Redis holds up every request for 5 seconds instead of failing fast.
- **Too aggressive for slow dependencies**: A 500ms timeout on a payment API that legitimately takes 2 seconds causes false failures. The circuit breaker trips on a healthy service.
- **No prioritization**: When resources are constrained, the system can't distinguish between "payment service is slow (critical)" and "analytics service is slow (best-effort)."

**Per-dependency isolation:**

```typescript
interface DependencyConfig {
  timeout: number;
  circuitBreaker: {
    failureThreshold: number;
    resetTimeout: number;
  };
  maxConcurrent: number; // bulkhead
  fallback?: () => Promise<any>;
}

const dependencies: Record<string, DependencyConfig> = {
  paymentService: {
    timeout: 5000,         // payment is slow but critical
    circuitBreaker: { failureThreshold: 3, resetTimeout: 30000 },
    maxConcurrent: 30,     // generous allocation
    // no fallback — payment failures must surface to user
  },
  recommendationEngine: {
    timeout: 500,          // should be fast, fail fast if not
    circuitBreaker: { failureThreshold: 5, resetTimeout: 60000 },
    maxConcurrent: 10,     // limited — non-critical
    fallback: async () => getPopularProducts(), // graceful degradation
  },
  inventoryCache: {
    timeout: 100,          // cache misses should be instant
    circuitBreaker: { failureThreshold: 10, resetTimeout: 10000 },
    maxConcurrent: 50,
    fallback: async () => getFromDatabase(), // fallback to primary store
  },
};
```

**How fallback chains work:**

A fallback chain defines a sequence of alternatives when the primary dependency fails:

```
Primary: recommendation engine (ML model, personalized)
  ↓ fails/times out
Fallback 1: cached recommendations from Redis (recent but not real-time)
  ↓ fails/times out
Fallback 2: popular products from database (generic but always available)
  ↓ fails/times out
Fallback 3: empty array (degrade the UI section entirely)
```

```typescript
async function getRecommendations(userId: string): Promise<Product[]> {
  // Primary
  try {
    return await withTimeout(recommendationEngine.getPersonalized(userId), 500);
  } catch {
    metrics.increment('recommendations.fallback.cache');
  }

  // Fallback 1: cached
  try {
    return await withTimeout(redis.get(`recs:${userId}`), 100);
  } catch {
    metrics.increment('recommendations.fallback.popular');
  }

  // Fallback 2: popular products
  try {
    return await withTimeout(db.query('SELECT * FROM popular_products LIMIT 10'), 200);
  } catch {
    metrics.increment('recommendations.fallback.empty');
  }

  // Fallback 3: nothing
  return [];
}
```

Each level in the chain has its own timeout — you don't want fallbacks to accumulate latency. The total latency budget for the entire chain should be capped (e.g., if primary times out at 500ms, fallbacks should complete within 300ms total so the user doesn't wait 1.5 seconds for recommendations).

**The per-dependency isolation pattern**: Each dependency gets its own timeout, circuit breaker, concurrency limit (bulkhead), and fallback. This means a slow recommendation engine can't exhaust connections needed for payments, and a flaky analytics service can't trip a breaker that affects critical paths.

</details>

<details>
<summary>8. What is retry amplification and why does it cause cascading failures across service layers — how does a retry at one layer multiply into exponential load on downstream services, what are retry budgets, and how do you implement them to cap the total retry volume across an entire call chain?</summary>

**What retry amplification is:**

When services are layered (A → B → C) and each layer has its own retry policy, a single failed request at the bottom multiplies exponentially upward.

Example: A retries 3 times, B retries 3 times, C fails.

- User sends 1 request to A
- A calls B. B calls C. C fails.
- B retries C 3 times (3 calls to C)
- B fails, A retries B 3 times
- Each retry to B triggers 3 retries to C
- Total calls to C: 3 × 3 = **9 calls** from 1 user request

Add another layer (A → B → C → D, each retrying 3 times) and D receives 27 calls. With 5 layers, it's 243 calls from one original request.

**Why this causes cascading failure:**

Service C is already struggling — that's why it's failing. Retry amplification piles 9x the load on it, making recovery impossible. C's increased response time causes B to hold connections longer, exhausting B's connection pool. B starts failing, and A's retries amplify load on B. The failure cascades upward until the entire system is down.

**Retry budgets:**

A retry budget caps the total percentage of requests that can be retries at any given point in the call chain. Instead of each layer independently deciding to retry, there's a system-wide limit.

How it works:
- Each service tracks its ratio of retry requests to total requests
- If retries exceed the budget (e.g., 20% of traffic), additional retries are suppressed
- This prevents the multiplicative explosion

**Implementation:**

```typescript
class RetryBudget {
  private requests = 0;
  private retries = 0;
  private readonly budgetPercent: number;
  private readonly windowMs: number;
  private windowStart: number;

  constructor(budgetPercent = 20, windowMs = 10_000) {
    this.budgetPercent = budgetPercent;
    this.windowMs = windowMs;
    this.windowStart = Date.now();
  }

  private resetIfNeeded() {
    if (Date.now() - this.windowStart > this.windowMs) {
      this.requests = 0;
      this.retries = 0;
      this.windowStart = Date.now();
    }
  }

  recordRequest() {
    this.resetIfNeeded();
    this.requests++;
  }

  canRetry(): boolean {
    this.resetIfNeeded();
    if (this.requests === 0) return true;
    // Allow retries only if they're under budget
    return (this.retries / this.requests) * 100 < this.budgetPercent;
  }

  recordRetry() {
    this.retries++;
  }
}

// Usage in an HTTP client wrapper
const retryBudget = new RetryBudget(20); // 20% budget

async function callWithRetry(url: string, maxRetries = 3): Promise<Response> {
  retryBudget.recordRequest();

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fetch(url, { signal: AbortSignal.timeout(2000) });
    } catch (err) {
      if (attempt === maxRetries) throw err;
      if (!retryBudget.canRetry()) {
        // Budget exhausted — fail fast instead of retrying
        throw new Error('Retry budget exhausted');
      }
      retryBudget.recordRetry();
      await sleep(Math.pow(2, attempt) * 100 + Math.random() * 100); // backoff + jitter
    }
  }
}
```

**Additional defenses against retry amplification:**

1. **Only retry at the edge**: The service closest to the user retries. Intermediate services fail fast and propagate errors upward.
2. **Propagate retry context**: Pass a header (`X-Retry-Count`) so downstream services know this is already a retry and can choose not to retry further.
3. **Combine with circuit breakers**: If the downstream is truly down (breaker open), retries stop entirely.
4. **Distinguish retryable vs non-retryable**: A 400 error won't succeed on retry. Only retry transient failures (as covered in the cloud design patterns section on retry patterns).

</details>

<details>
<summary>9. When do resilience patterns like circuit breakers, retries, and bulkheads hurt more than they help — what are the signs that a team has over-applied these patterns, what failure modes do misconfigured resilience patterns introduce (masking real issues, delaying detection, adding latency), and how do you decide which patterns a given service actually needs?</summary>

**Signs of over-applied resilience patterns:**

- **Every outbound call has a circuit breaker, retry, bulkhead, and fallback** regardless of whether the dependency is critical or best-effort. Configuration files are enormous and nobody understands the interaction between all the settings.
- **Incidents take longer to detect** because fallbacks mask failures. The recommendation engine has been down for hours but nobody noticed because the fallback returns cached results — meanwhile, the cache is serving increasingly stale data.
- **Debugging is harder, not easier**: A request fails and the logs show "circuit breaker open → fallback triggered → retry attempted → bulkhead rejected." Tracing the actual root cause requires understanding the state machine of every pattern in the chain.
- **Latency increases under normal operation**: Retries add latency to transient errors that would have succeeded without them. Bulkhead semaphores add contention. Health checks add overhead. The 99th percentile is worse than without the patterns.
- **False positives**: Circuit breakers trip on latency spikes that are normal for the dependency. The service degrades unnecessarily.

**Failure modes of misconfigured patterns:**

- **Retries masking bugs**: A service returns 500 because of a code bug. Retries "fix" it by hitting a different instance that doesn't have the bug (or the timing is different). The bug goes undiagnosed because the retry succeeds often enough.
- **Circuit breakers preventing recovery**: The breaker trips and the half-open probe sends a single request to a service that's recovering but slow. The probe times out, the breaker stays open. The downstream is healthy but the breaker won't let traffic through.
- **Bulkheads causing self-DoS**: Bulkhead limits set too low reject legitimate traffic during normal spikes. The service throttles itself unnecessarily.
- **Fallbacks creating silent data corruption**: The fallback returns cached data that's no longer valid, and downstream consumers treat it as fresh. Stale prices, stale inventory counts, stale permissions.
- **Retry + circuit breaker interaction**: Retries keep the failure count high, causing the breaker to trip faster than it should. Or the breaker opens before retries have a chance to succeed on transient errors.

**How to decide which patterns a service actually needs:**

Ask these questions for each dependency:

1. **What happens if this dependency is completely unavailable?**
   - Service can't function at all → timeout + circuit breaker (fail fast, don't waste resources)
   - Service can function with degraded experience → timeout + fallback
   - Nobody notices → maybe just a timeout is enough

2. **Is the failure mode transient or sustained?**
   - Transient (network blips) → retry with backoff
   - Sustained (service down for minutes) → circuit breaker to stop trying

3. **What's the blast radius?**
   - One slow dependency can starve others → bulkhead
   - Dependencies are isolated naturally (separate connection pools already) → bulkhead is redundant

4. **Is the dependency in the critical path?**
   - Yes (payment, auth) → full pattern suite justified
   - No (analytics, telemetry) → fire-and-forget with a timeout, nothing more

**The pragmatic starting point:**

- **Every external call**: timeout (non-negotiable)
- **Critical dependencies**: timeout + retry (2-3 attempts, backoff + jitter) + circuit breaker
- **Non-critical dependencies**: timeout + fallback (no retry — if it fails, use the fallback)
- **Bulkhead**: only when you've observed or can reason about one dependency starving others

Add patterns in response to observed failure modes, not preemptively. As covered in the cloud design patterns topic: patterns are medicine, not vitamins.

</details>

<details>
<summary>10. How does backpressure differ from rate limiting as a strategy for handling overload — what does upstream signal propagation mean in practice, when should a system shed load vs propagate backpressure, how does queue depth serve as a backpressure signal, and what happens when a system has rate limiting but no backpressure mechanism?</summary>

**The core difference:**

- **Rate limiting** is a hard cap on incoming requests, typically applied at the edge. It's a unilateral decision: "I will only accept N requests per second, regardless of why." It protects the server but doesn't inform the client about system capacity.
- **Backpressure** is a feedback signal from consumer to producer: "Slow down, I can't keep up." It's collaborative — the producer adjusts its rate based on the consumer's capacity. The signal flows upstream through the system.

**Upstream signal propagation in practice:**

In a pipeline A → B → C, if C is slow:
1. B's outbound queue to C fills up (queue depth increases)
2. B detects the growing queue and slows its own processing rate
3. B's inbound queue from A fills up
4. A detects B's queue is full and either slows its own rate or rejects new work
5. The signal propagates all the way back to the original producer

This is how TCP flow control works (receiver window), how reactive streams work (Subscriber.request(n)), and how well-designed message systems work (consumer pauses, producer detects and throttles).

**Queue depth as a backpressure signal:**

```
Queue depth low (< 100)  → process normally, full speed
Queue depth medium (100-500) → slow down, process at reduced rate
Queue depth high (500-1000) → pause new consumption, let queue drain
Queue depth critical (> 1000) → shed load, drop low-priority messages
```

The queue depth is a natural indicator of the gap between production rate and consumption rate. Monitoring it provides early warning before the system tips into failure.

**When to shed load vs propagate backpressure:**

| Scenario | Strategy |
|---|---|
| Temporary spike, consumers will catch up | Backpressure — slow producers down, queue absorbs the burst |
| Sustained overload, consumers can never catch up | Load shedding — drop low-priority work, protect high-priority |
| External clients you don't control | Rate limiting at the edge (you can't propagate backpressure to the internet) |
| Internal pipeline with cooperative services | Backpressure — signal upstream to slow down |
| Queue approaching memory limits | Load shedding — drop messages before the broker crashes |

**What happens with rate limiting but no backpressure:**

The system accepts work at the rate limit and queues it internally. If the consumer falls behind:

1. Internal queues grow unbounded (no signal tells anyone to slow down)
2. Memory usage climbs
3. Processing latency increases (messages sit in queue for minutes/hours)
4. If the queue is in-memory, the process eventually OOMs
5. If the queue is persistent, disk fills up and the broker stops accepting messages — but by this point, the system has been producing stale results for a long time

Rate limiting is like a bouncer at the door — it controls who gets in. Backpressure is like the kitchen telling the waiter to stop taking orders — it propagates capacity information through the system. You need both: rate limiting at the edges to protect against external abuse, and backpressure internally to coordinate capacity across services.

</details>

<details>
<summary>11. Why is stateless service design a prerequisite for horizontal scaling -- what does shared-nothing architecture mean in practice, where should session state be externalized (Redis, database, signed tokens), what are the tradeoffs between these options, and what subtle statefulness (in-memory caches, local file writes, sticky sessions) do teams miss when they think their services are stateless?</summary>

**Why statelessness enables horizontal scaling:**

If a service stores state in memory (sessions, caches, counters), any specific request must go to the specific instance that holds that state. This means:
- You can't freely route requests to any instance (load balancing is constrained)
- Adding instances doesn't help unless you can shard the state (complex)
- Losing an instance loses the state it held (reliability problem)
- Scaling down is dangerous (you might evict an instance with active sessions)

**Shared-nothing architecture** means each instance is identical and interchangeable. Any instance can handle any request. No instance depends on data held by another instance. The only shared resource is the external data store (database, Redis).

**Where to externalize session state:**

| Approach | Pros | Cons | When to use |
|---|---|---|---|
| **Redis** | Fast (sub-ms reads), built-in TTL, supports complex data structures | Additional infrastructure, network hop per request, Redis itself is a SPOF unless clustered | Best for frequently read/written session data: shopping carts, wizard progress, temporary preferences |
| **Database** | Already exists, durable, no new infrastructure | Slower than Redis, adds load to the database, needs cleanup of expired sessions | Best for durable state that must survive restarts: user profiles, settings |
| **Signed tokens (JWT)** | No server-side storage at all, no network hop, scales infinitely | Can't revoke individual tokens (without a blacklist, which is server-side state), token size grows with claims, sensitive data must not be in the payload | Best for authentication: stateless auth tokens with user ID, roles, expiry. Use short expiry + refresh tokens |

**Subtle statefulness teams miss:**

1. **In-memory caches without coordination**: An instance caches product prices. Another instance updates the price. The first instance serves stale prices until its cache TTL expires. This isn't stateless — it's stateful with uncoordinated state.

2. **Local file writes**: Uploading files to the local filesystem. The next request goes to a different instance and can't find the file. Always use object storage (S3) or a shared filesystem.

3. **Sticky sessions**: The load balancer routes the same user to the same instance. This makes the service *effectively* stateful — if that instance dies, the session is lost. It also creates uneven load distribution.

4. **In-process scheduled jobs**: A cron job or `setInterval` running inside the service. Scale to 5 instances and the job runs 5 times. You need external job scheduling (cron jobs in K8s, distributed locks).

5. **Connection-specific state**: WebSocket connections hold state by nature. Scaling WebSocket services requires a pub/sub broker (Redis pub/sub) to broadcast messages across instances.

6. **Counters and rate limiters in memory**: Per-user rate limiting using an in-memory counter. Each instance counts independently, so the actual rate limit is `limit × instance_count`. Use Redis for shared counters.

The test: "If I kill any instance right now, does anything break besides the in-flight requests on that instance?" If the answer is yes, you have hidden statefulness.

</details>

<details>
<summary>12. How do you approach capacity planning for a growing service -- how do you estimate future traffic, what metrics should trigger scaling decisions (CPU, queue depth, latency percentiles), how much headroom should you maintain, and when should you scale proactively vs reactively?</summary>

**Estimating future traffic:**

1. **Historical growth rate**: Look at the last 6-12 months of traffic data. If traffic grew 10% month-over-month, extrapolate forward with a margin.
2. **Business-driven projections**: Sales team is planning a Black Friday campaign? Marketing launching in a new region? These create step-function increases, not gradual growth.
3. **Seasonal patterns**: E-commerce peaks during holidays, tax software peaks in April. Historical patterns predict future spikes.
4. **Load testing**: Run synthetic load at 2x and 5x current traffic to find where things break. This tells you your ceiling, not just your current state.

**Metrics that should trigger scaling:**

| Metric | Why it matters | Scaling trigger |
|---|---|---|
| **CPU utilization** | Sustained high CPU means compute-bound saturation | Scale when p50 > 60-70% sustained |
| **Request latency (p99)** | Latency degradation is the first sign users notice | Scale when p99 exceeds SLO threshold |
| **Queue depth** | Growing queue means consumers can't keep up | Scale when depth grows consistently over 5-10 minutes |
| **Error rate** | Increasing 5xx errors indicate resource exhaustion | Scale when error rate exceeds baseline by 2-3x |
| **Memory utilization** | Memory pressure causes GC pauses, OOMs | Scale when sustained > 80% |
| **Connection pool utilization** | Near-exhausted pools cause queueing and timeouts | Scale (or add pooling) when pool usage > 80% |
| **Event loop lag (Node.js)** | High event loop delay means the process is saturated | Scale when lag exceeds 100ms consistently |

**Key insight**: Use *leading* indicators (latency, queue depth, event loop lag) rather than *lagging* indicators (error rate, OOM kills). By the time errors spike, users are already impacted.

**How much headroom:**

- **Day-to-day**: Maintain 30-40% headroom above normal peak traffic. This absorbs organic spikes without scaling events.
- **Known events**: Provision 2-3x expected peak for planned events (product launches, sales). Better to over-provision and scale down than scramble during the event.
- **Autoscaling buffer**: Autoscaling has latency (30-90 seconds for a new pod to be ready). Your current capacity must handle the load during that ramp-up period.

**Proactive vs reactive scaling:**

| Proactive (scale ahead) | Reactive (scale on demand) |
|---|---|
| Planned events (Black Friday, product launch) | Organic traffic growth |
| Scheduled traffic patterns (morning ramp-up) | Unexpected viral traffic |
| New region launches with known user count | Cost optimization — scale down during off-hours |
| Compliance requirements (must handle X RPS) | Development/staging environments |

**Practical approach:**

- Set up autoscaling based on CPU and custom metrics (queue depth, latency)
- Use HPA (Horizontal Pod Autoscaler) in Kubernetes with sensible min/max bounds
- Pre-scale proactively before known events
- Run quarterly capacity reviews comparing actual growth against projections
- Alert on headroom shrinking below threshold, not just on saturation

```yaml
# Kubernetes HPA example
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65  # scale before saturation
    - type: Pods
      pods:
        metric:
          name: http_request_latency_p99
        target:
          type: AverageValue
          averageValue: "500m"  # 500ms p99 threshold
```

</details>

<details>
<summary>13. What are the tradeoffs between active-active and active-passive multi-region architectures — how does latency-based routing work, what makes regional failover harder than it sounds, and what data replication challenges (conflict resolution, eventual consistency, write routing) make active-active significantly more complex than active-passive?</summary>

**Active-passive:**

One region (primary) handles all traffic. The other region (standby) is a warm copy — data is replicated there, infrastructure is provisioned, but it doesn't serve production traffic until failover.

Pros:
- Simple data model — one source of truth, no conflicts
- Writes always go to one region — no conflict resolution needed
- Lower cost — standby region runs minimal compute (just enough to receive replicas)
- Simpler testing — you're testing one production path

Cons:
- **Wasted capacity**: The standby region sits idle until failover
- **Failover is not instant**: DNS propagation (minutes), database promotion (seconds to minutes), cache warming (minutes). Total failover time: 5-30 minutes
- **Failover confidence**: If you never fail over, you don't know if it works. Regular failover drills are essential but operationally expensive
- **Latency**: Users far from the primary region experience higher latency with no alternative

**Active-active:**

Both regions serve production traffic simultaneously. Writes happen in both regions. Data is replicated bidirectionally.

Pros:
- Lower latency for all users (route to nearest region)
- No failover needed — if one region dies, the other already handles traffic
- Better resource utilization — both regions work, not one idle
- Scales capacity by adding regions

Cons:
- **Conflict resolution is hard**: User updates their profile in region A and region B simultaneously. Which write wins? You need conflict resolution strategies (last-writer-wins, vector clocks, CRDTs, application-level merge logic).
- **Eventual consistency is unavoidable**: A write in region A isn't immediately visible in region B. Cross-region replication adds 50-200ms latency. Some reads will be stale.
- **Write routing complexity**: Some data can be written in any region (user preferences). Some data must be written in a specific region (financial transactions requiring strong consistency). Routing rules become complex.
- **Significantly more testing**: Every feature must be tested for concurrent writes, conflict resolution, and stale reads.

**Latency-based routing:**

DNS services (Route 53, Cloudflare) measure latency from the user to each region and return the IP of the closest region. The user's request goes to the lowest-latency region automatically.

```
User in Frankfurt → Route 53 → eu-west-1 (30ms)
User in Tokyo     → Route 53 → ap-northeast-1 (20ms)
User in New York  → Route 53 → us-east-1 (15ms)
```

Complications:
- DNS caching means routing changes aren't instant (TTL-dependent)
- "Closest" by latency doesn't always mean "best" — a nearby region under heavy load might be slower than a farther one
- Health checks must be integrated — don't route to a region that's degraded

**Why regional failover is harder than it sounds:**

1. **DNS propagation**: Changing DNS records takes minutes due to caching. Some clients cache aggressively and ignore TTL.
2. **Database promotion**: In active-passive, the standby database must be promoted to primary. This involves WAL replay, connection repointing, and potential data loss of un-replicated writes (RPO > 0).
3. **State migration**: In-flight sessions, cached data, queued messages — none of this migrates automatically.
4. **Dependency fan-out**: Your service fails over, but what about your dependencies? If your payment provider is region-specific, failing over your service doesn't help.
5. **Cache cold start**: The standby region has cold caches. Failover traffic hits the database directly, potentially overwhelming it (same cache stampede problem as horizontal scaling).
6. **Runbook complexity**: Failover involves multiple coordinated steps across multiple systems. Under incident pressure, mistakes happen.

**Data replication challenges in active-active:**

| Challenge | Description | Common solution |
|---|---|---|
| Conflict resolution | Concurrent writes to the same record in different regions | Last-writer-wins (LWW) by timestamp, CRDTs for mergeable data, application-level merge |
| Replication lag | Cross-region replication takes 50-200ms | Accept eventual consistency for most reads, route user to their "home" region for write-heavy flows |
| Write amplification | Every write is replicated to every region | Accept the cost, or partition writes by region (user belongs to one region) |
| Schema migrations | Must be backward-compatible since both regions run during migration | Online schema changes, expand-contract pattern |

**Practical recommendation**: Start with active-passive. Move to active-active only when latency requirements demand it (global user base, < 100ms response time SLA) or when your RPO/RTO requirements are stricter than what active-passive failover can deliver. Most systems never need active-active.

</details>

## Practical — Implementation & Configuration

<details>
<summary>14. Show how you'd implement graceful degradation in a Node.js service that depends on a recommendation engine and a payment service — demonstrate fallback responses when the recommendation engine is down (cached/default recommendations), feature flag-driven degradation to read-only mode, and explain how you'd prioritize which capabilities to shed under increasing failure</summary>

```typescript
// Feature flag service (backed by LaunchDarkly, Unleash, or a simple config)
interface FeatureFlags {
  isEnabled(flag: string): boolean;
}

// Circuit breaker wrapper (simplified — use opossum or similar in production)
interface CircuitBreaker {
  fire<T>(fn: () => Promise<T>): Promise<T>;
  isOpen(): boolean;
}

// Degradation tiers — documented and agreed on before incidents
enum DegradationLevel {
  NORMAL = 'normal',
  PARTIAL = 'partial',     // non-critical features disabled
  READ_ONLY = 'read_only', // writes disabled
  MINIMAL = 'minimal',     // only health checks and static content
}

class ProductPageService {
  constructor(
    private recommendationEngine: CircuitBreaker,
    private paymentService: CircuitBreaker,
    private redis: Redis,
    private db: Pool,
    private featureFlags: FeatureFlags,
    private metrics: MetricsClient,
  ) {}

  async getProductPage(productId: string, userId: string) {
    // Core product data — always required, no fallback
    const product = await this.db.query(
      'SELECT * FROM products WHERE id = $1',
      [productId],
    );

    if (!product) throw new NotFoundError('Product not found');

    // Recommendations — non-critical, multiple fallback levels
    const recommendations = await this.getRecommendations(productId, userId);

    // Purchase availability — depends on degradation level
    const canPurchase = this.featureFlags.isEnabled('purchases_enabled')
      && !this.paymentService.isOpen();

    return {
      product,
      recommendations,
      canPurchase,
      degraded: !canPurchase || recommendations.source !== 'live',
    };
  }

  private async getRecommendations(
    productId: string,
    userId: string,
  ): Promise<{ items: Product[]; source: string }> {
    // If feature is disabled entirely, skip everything
    if (!this.featureFlags.isEnabled('recommendations')) {
      return { items: [], source: 'disabled' };
    }

    // Primary: live recommendation engine
    try {
      const items = await this.recommendationEngine.fire(() =>
        this.fetchRecommendations(productId, userId),
      );
      // Cache successful response for fallback use
      await this.redis
        .setex(`recs:${productId}`, 3600, JSON.stringify(items))
        .catch(() => {}); // don't fail if cache write fails
      return { items, source: 'live' };
    } catch {
      this.metrics.increment('recommendations.fallback');
    }

    // Fallback 1: cached recommendations
    try {
      const cached = await this.redis.get(`recs:${productId}`);
      if (cached) {
        return { items: JSON.parse(cached), source: 'cache' };
      }
    } catch {
      this.metrics.increment('recommendations.cache_miss');
    }

    // Fallback 2: popular products (always available in DB)
    try {
      const popular = await this.db.query(
        'SELECT * FROM products ORDER BY sales_count DESC LIMIT 8',
      );
      return { items: popular.rows, source: 'popular' };
    } catch {
      // Even the DB fallback failed
      return { items: [], source: 'empty' };
    }
  }

  // Read-only mode for checkout
  async checkout(cartId: string, userId: string) {
    if (!this.featureFlags.isEnabled('purchases_enabled')) {
      throw new ServiceUnavailableError(
        'Purchases are temporarily disabled. Your cart is saved.',
      );
    }

    if (this.paymentService.isOpen()) {
      throw new ServiceUnavailableError(
        'Payment processing is temporarily unavailable. Please try again shortly.',
      );
    }

    // Proceed with checkout...
    return this.processCheckout(cartId, userId);
  }
}
```

**Degradation priority (using the shedding priorities from Q6):**

| Priority | Shed when... | Feature | Fallback |
|---|---|---|---|
| 1 (first to go) | Any non-critical failure | Personalized recommendations | Popular products |
| 2 | Recommendation engine + cache down | Product reviews, ratings | Hide section |
| 3 | Increasing error rates | User activity tracking, analytics | Drop silently |
| 4 | Payment service unstable | New purchases | Read-only mode (browsing + cart, no checkout) |
| 5 (last to go) | Severe degradation | Search | Category browsing fallback |
| Never shed | — | Authentication, product catalog, cart persistence | — |

**Key design principles:**

- **Fallback responses should be pre-computed and cached proactively**, not computed during the incident. Popular products should already be in a fast-access table or cache.
- **Feature flags should be checkable without network calls** during incidents. If your feature flag service is also down, flags should default to a safe state (typically: non-critical features off, critical features on).
- **The `source` field in the response** lets the frontend know the data is degraded and display appropriate UI (e.g., "Recommended for you" becomes "Popular products" with no misleading personalization claim).
- **Metrics on every fallback path** so you know in real-time which degradation paths are active and for how long.

</details>

<details>
<summary>15. Implement a backpressure mechanism in a Node.js service that consumes from a message queue — show how you'd use queue depth as a signal to slow down consumption, how to propagate backpressure upstream (e.g., returning HTTP 429 or pausing consumers), and what load shedding looks like when the queue hits a critical depth threshold</summary>

Implementing the backpressure concepts from Q10 — using queue depth thresholds to signal overload and propagate pressure upstream:

```typescript
import { EventEmitter } from 'events';

interface QueueMessage {
  id: string;
  body: unknown;
  priority: 'high' | 'normal' | 'low';
  timestamp: number;
  ack(): Promise<void>;
  nack(): Promise<void>;
}

interface QueueConsumer {
  pause(): void;
  resume(): void;
  on(event: 'message', handler: (msg: QueueMessage) => void): void;
}

// Backpressure thresholds
const THRESHOLDS = {
  LOW_WATERMARK: 100,      // normal operation
  MEDIUM_WATERMARK: 500,   // slow down consumption
  HIGH_WATERMARK: 2000,    // pause consumption, start shedding low-priority
  CRITICAL: 5000,          // shed everything except high-priority
} as const;

enum PressureLevel {
  NORMAL = 'normal',
  ELEVATED = 'elevated',
  HIGH = 'high',
  CRITICAL = 'critical',
}

class BackpressureConsumer extends EventEmitter {
  private pressureLevel = PressureLevel.NORMAL;
  private processingCount = 0;
  private readonly maxConcurrent: number;
  private monitorInterval: NodeJS.Timeout | null = null;

  constructor(
    private consumer: QueueConsumer,
    private queue: { getDepth(): Promise<number> },
    private metrics: MetricsClient,
    maxConcurrent = 10,
  ) {
    super();
    this.maxConcurrent = maxConcurrent;
  }

  start() {
    // Monitor queue depth periodically
    this.monitorInterval = setInterval(() => this.checkPressure(), 5_000);

    this.consumer.on('message', (msg) => this.handleMessage(msg));
    this.consumer.resume();
  }

  stop() {
    if (this.monitorInterval) clearInterval(this.monitorInterval);
    this.consumer.pause();
  }

  private async checkPressure() {
    const depth = await this.queue.getDepth();
    const previousLevel = this.pressureLevel;

    this.metrics.gauge('queue.depth', depth);

    if (depth >= THRESHOLDS.CRITICAL) {
      this.pressureLevel = PressureLevel.CRITICAL;
      this.consumer.pause(); // stop consuming entirely
    } else if (depth >= THRESHOLDS.HIGH_WATERMARK) {
      this.pressureLevel = PressureLevel.HIGH;
      this.consumer.pause(); // pause until depth drops
    } else if (depth >= THRESHOLDS.MEDIUM_WATERMARK) {
      this.pressureLevel = PressureLevel.ELEVATED;
      this.consumer.resume(); // consume, but at reduced concurrency
    } else {
      this.pressureLevel = PressureLevel.NORMAL;
      this.consumer.resume();
    }

    if (previousLevel !== this.pressureLevel) {
      this.metrics.increment('backpressure.level_change', {
        from: previousLevel,
        to: this.pressureLevel,
      });
      this.emit('pressure_change', this.pressureLevel);
    }
  }

  private async handleMessage(msg: QueueMessage) {
    // Load shedding based on pressure level
    if (this.shouldShed(msg)) {
      this.metrics.increment('messages.shed', { priority: msg.priority });
      await msg.ack(); // acknowledge to remove from queue
      return;
    }

    // Concurrency limit — enforce backpressure locally
    const concurrencyLimit = this.getEffectiveConcurrency();
    if (this.processingCount >= concurrencyLimit) {
      await msg.nack(); // return to queue, will be redelivered
      return;
    }

    this.processingCount++;
    try {
      await this.processMessage(msg);
      await msg.ack();
    } catch (err) {
      await msg.nack();
      this.metrics.increment('messages.processing_error');
    } finally {
      this.processingCount--;
    }
  }

  private shouldShed(msg: QueueMessage): boolean {
    switch (this.pressureLevel) {
      case PressureLevel.CRITICAL:
        // Only process high-priority messages
        return msg.priority !== 'high';
      case PressureLevel.HIGH:
        // Drop low-priority messages
        return msg.priority === 'low';
      case PressureLevel.ELEVATED:
        // Drop old low-priority messages (stale > 5 minutes)
        return (
          msg.priority === 'low' &&
          Date.now() - msg.timestamp > 5 * 60 * 1000
        );
      default:
        return false;
    }
  }

  private getEffectiveConcurrency(): number {
    switch (this.pressureLevel) {
      case PressureLevel.CRITICAL:
        return Math.ceil(this.maxConcurrent * 0.25); // 25% capacity
      case PressureLevel.HIGH:
        return Math.ceil(this.maxConcurrent * 0.5);  // 50% capacity
      case PressureLevel.ELEVATED:
        return Math.ceil(this.maxConcurrent * 0.75); // 75% capacity
      default:
        return this.maxConcurrent;
    }
  }

  // Expose pressure level for HTTP API layer
  getPressureLevel(): PressureLevel {
    return this.pressureLevel;
  }

  private async processMessage(msg: QueueMessage): Promise<void> {
    // Actual message processing logic
  }
}
```

**Propagating backpressure to HTTP producers:**

```typescript
// HTTP endpoint where producers submit work to the queue
class IngestController {
  constructor(
    private backpressureConsumer: BackpressureConsumer,
    private queue: MessageQueue,
  ) {}

  async handleIngest(req: Request, res: Response) {
    const pressure = this.backpressureConsumer.getPressureLevel();

    // Propagate backpressure via HTTP status codes
    if (pressure === PressureLevel.CRITICAL) {
      return res.status(503).json({
        error: 'Service overloaded',
        retryAfter: 60,
      });
    }

    if (pressure === PressureLevel.HIGH) {
      // Accept high-priority, reject normal/low
      if (req.body.priority !== 'high') {
        return res.status(429).json({
          error: 'System under load — only high-priority requests accepted',
          retryAfter: 30,
        });
      }
    }

    if (pressure === PressureLevel.ELEVATED) {
      // Accept but signal the producer to slow down
      res.setHeader('Retry-After', '5');
      res.setHeader('X-Backpressure', 'elevated');
    }

    await this.queue.publish(req.body);
    return res.status(202).json({ accepted: true });
  }
}
```

**Key points:**

- **Queue depth is the primary signal**, checked on an interval rather than per-message (checking per-message adds latency and load on the queue).
- **Load shedding is priority-based**: Low-priority messages are shed first, high-priority messages are shed only at critical levels.
- **Backpressure propagation uses HTTP semantics**: 429 (Too Many Requests) with `Retry-After` header tells producers to back off. 503 means stop sending entirely.
- **Concurrency reduction** gradually throttles processing — don't go from full speed to zero. Smooth transitions prevent oscillation.
- **Shedding means ack-ing and discarding**, not nack-ing (which would redeliver and grow the queue further). Dead-letter the shed messages if you need auditability.

</details>

## Practical — Database Scaling

<details>
<summary>16. Set up read replica routing in a Node.js/TypeScript application — show how you'd configure the database client to send writes to the primary and reads to replicas, handle the replication lag problem for read-after-write scenarios (e.g., user creates a record then immediately views it), and explain the tradeoffs between routing strategies (sticky connections, primary fallback, causal consistency tokens)</summary>

Building on the routing concept from Q4, here's a production-ready implementation that adds explicit read options, service-layer write tracking, and auto-detection of query type:

```typescript
import { Pool, PoolConfig } from 'pg';

interface ReadWriteRouter {
  query(sql: string, params?: unknown[]): Promise<any>;
  writeQuery(sql: string, params?: unknown[]): Promise<any>;
  readQuery(sql: string, params?: unknown[], opts?: ReadOptions): Promise<any>;
}

interface ReadOptions {
  // Force read from primary (for read-after-write consistency)
  fromPrimary?: boolean;
  // Entity that was recently written — check if we should use primary
  entityId?: string;
}

class DatabaseRouter implements ReadWriteRouter {
  private primary: Pool;
  private replicas: Pool[];
  private replicaIndex = 0;

  // Track recent writes: entityId → timestamp
  private recentWrites = new Map<string, number>();
  private readonly STALE_WINDOW_MS = 5_000; // 5 seconds

  constructor(config: {
    primary: PoolConfig;
    replicas: PoolConfig[];
  }) {
    this.primary = new Pool({
      ...config.primary,
      max: 20, // sized for writes + read-after-write overflow
    });

    this.replicas = config.replicas.map(
      (replicaConfig) =>
        new Pool({
          ...replicaConfig,
          max: 30, // sized for read-heavy traffic
        }),
    );

    // Clean up stale entries every 30 seconds
    setInterval(() => this.cleanupRecentWrites(), 30_000);
  }

  // Auto-detect: SELECT → replica, everything else → primary
  async query(sql: string, params?: unknown[]): Promise<any> {
    const isRead = sql.trimStart().toUpperCase().startsWith('SELECT');
    return isRead
      ? this.readQuery(sql, params)
      : this.writeQuery(sql, params);
  }

  async writeQuery(sql: string, params?: unknown[]): Promise<any> {
    return this.primary.query(sql, params);
  }

  // Track a write for read-after-write consistency
  trackWrite(entityId: string) {
    this.recentWrites.set(entityId, Date.now());
  }

  async readQuery(
    sql: string,
    params?: unknown[],
    opts: ReadOptions = {},
  ): Promise<any> {
    // Route to primary if explicitly requested or if recent write detected
    if (opts.fromPrimary) {
      return this.primary.query(sql, params);
    }

    if (opts.entityId) {
      const lastWrite = this.recentWrites.get(opts.entityId);
      if (lastWrite && Date.now() - lastWrite < this.STALE_WINDOW_MS) {
        return this.primary.query(sql, params);
      }
    }

    // Round-robin across replicas
    const replica = this.replicas[this.replicaIndex % this.replicas.length];
    this.replicaIndex++;
    return replica.query(sql, params);
  }

  private cleanupRecentWrites() {
    const cutoff = Date.now() - this.STALE_WINDOW_MS;
    for (const [key, ts] of this.recentWrites) {
      if (ts < cutoff) this.recentWrites.delete(key);
    }
  }

  async shutdown() {
    await Promise.all([
      this.primary.end(),
      ...this.replicas.map((r) => r.end()),
    ]);
  }
}
```

**Usage in a service layer:**

```typescript
class UserService {
  constructor(private db: DatabaseRouter) {}

  async updateProfile(userId: string, data: ProfileUpdate) {
    await this.db.writeQuery(
      'UPDATE users SET name = $1, email = $2 WHERE id = $3',
      [data.name, data.email, userId],
    );
    // Track this write so subsequent reads go to primary
    this.db.trackWrite(userId);
  }

  async getProfile(userId: string) {
    // Pass entityId so router checks for recent writes
    return this.db.readQuery(
      'SELECT * FROM users WHERE id = $1',
      [userId],
      { entityId: userId },
    );
  }

  async listUsers(page: number) {
    // No entityId — always safe to read from replica
    return this.db.readQuery(
      'SELECT * FROM users ORDER BY created_at DESC LIMIT 20 OFFSET $1',
      [(page - 1) * 20],
    );
  }
}
```

**Tradeoffs between routing strategies:**

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| **Primary fallback (shown above)** | After a write, route reads for that entity to primary for N seconds | Simple, no infrastructure changes, per-entity granularity | Adds load to primary during write-heavy periods, 5s window is a guess |
| **Sticky sessions** | Route all of a user's reads to primary after any write | Simpler logic — no per-entity tracking | Over-routes to primary (user writes once, ALL reads go to primary), creates uneven load |
| **Causal consistency tokens** | Primary returns LSN after write; replica waits until it has replicated past that LSN | Precise — no unnecessary primary reads, no guessing on time window | Requires database support (PostgreSQL has `pg_last_wal_replay_lsn()`), adds latency if replica is behind |

**Causal consistency token approach (PostgreSQL-specific):**

```typescript
// After write, get the current WAL position
const result = await this.primary.query('SELECT pg_current_wal_lsn()');
const lsn = result.rows[0].pg_current_wal_lsn;
// Store in session/cookie/header for subsequent reads

// Before read on replica, ensure it has caught up
await replica.query(
  `SELECT pg_last_wal_replay_lsn() >= $1::pg_lsn AS caught_up`,
  [lsn],
);
// If not caught up, fall back to primary
```

**Practical recommendation**: Start with the primary-fallback approach (time-window-based). It covers 95% of cases with minimal complexity. Move to causal consistency tokens only if you need precise consistency guarantees and your replication lag is unpredictable.

</details>

<details>
<summary>17. A service scaled to 50 instances is exhausting database connections — walk through how you'd diagnose this, show how to configure connection pooling (PgBouncer or application-level pooling), calculate the right pool size given instance count and database connection limits, and explain the tradeoffs between transaction-level and session-level pooling modes</summary>

**Diagnosing the problem:**

1. **Check PostgreSQL connection count:**
   ```sql
   SELECT count(*) FROM pg_stat_activity;
   SELECT max_connections FROM pg_settings WHERE name = 'max_connections';
   -- Typical default: 100. With 50 instances × 20 pool size = 1,000 needed.
   ```

2. **Check for connection errors in application logs:**
   - `FATAL: too many connections for role` — direct evidence
   - `connection timeout` or `ECONNREFUSED` — connection pool exhausted
   - Increasing response times — connections queued waiting for availability

3. **Check per-instance pool metrics:**
   ```typescript
   // If using node-postgres Pool, check these:
   const poolStats = {
     total: pool.totalCount,      // connections created
     idle: pool.idleCount,        // connections sitting unused
     waiting: pool.waitingCount,  // queries waiting for a connection
   };
   // If waitingCount > 0 consistently, pool is undersized or connections are leaking
   ```

4. **Check for connection leaks:** Queries that open a connection but don't release it (missing `client.release()` in node-postgres, or uncaught errors in transaction blocks).

**Calculating the right pool size:**

```
Database max_connections = 200 (after reserving ~10 for superuser/monitoring)
Available connections = 190
Instance count = 50

Per-instance pool size = floor(190 / 50) = 3 connections per instance
```

3 connections per instance is very small. This is the fundamental problem — at 50 instances with direct connections, the math doesn't work unless you increase `max_connections` (which has memory costs — each PostgreSQL connection uses ~5-10MB) or add a connection pooler.

**Solution: PgBouncer as a connection pooler:**

PgBouncer sits between your application and PostgreSQL. It multiplexes many application connections over a small number of database connections.

```ini
; pgbouncer.ini
[databases]
mydb = host=postgres-primary port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0

; Pool sizing
default_pool_size = 20        ; actual connections to PostgreSQL per database/user
max_client_conn = 2000        ; total client connections PgBouncer accepts
reserve_pool_size = 5         ; extra connections for spikes
reserve_pool_timeout = 3      ; seconds before using reserve pool

; Pooling mode
pool_mode = transaction       ; connections returned after each transaction

; Timeouts
server_idle_timeout = 300
client_idle_timeout = 60
query_timeout = 30
```

**With PgBouncer:**
- 50 instances × 20 connections each = 1,000 connections to PgBouncer (fine — PgBouncer handles thousands of lightweight connections)
- PgBouncer maintains only 20-25 actual connections to PostgreSQL
- The multiplexing ratio is 1000:25 = 40:1

**Application-level pooling (without PgBouncer):**

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: 'mydb',
  max: 4,                    // small pool — 50 instances × 4 = 200
  idleTimeoutMillis: 30_000, // release idle connections
  connectionTimeoutMillis: 5_000, // fail fast if no connection available
  statement_timeout: 10_000, // PostgreSQL session parameter, passed through by pg
});

// Always use pool.query() or properly release clients
// BAD: const client = await pool.connect(); // must remember client.release()
// GOOD: await pool.query('SELECT ...'); // auto-released
```

**Transaction-level vs session-level pooling:**

| Mode | How it works | Pros | Cons |
|---|---|---|---|
| **Transaction** | Connection returned to pool after each transaction (or auto-committed statement) | Maximum multiplexing — 40:1+ ratios possible, handles many clients with few DB connections | Can't use session-level features: `SET`, prepared statements, `LISTEN/NOTIFY`, advisory locks, temp tables |
| **Session** | Connection held for the entire client session | Full PostgreSQL feature support, behaves like a direct connection | Minimal multiplexing — basically 1:1 mapping, defeats the purpose at high instance counts |
| **Statement** | Connection returned after each statement | Even higher multiplexing | Can't use multi-statement transactions, very limited usefulness |

**Practical recommendation:**

- Use **transaction mode** (the default and most common). Most applications don't use session-level features.
- If you need prepared statements, use PgBouncer 1.21+ which supports them in transaction mode, or use `pg` driver's `{ prepared: false }` option.
- If you must use `LISTEN/NOTIFY` or advisory locks, create a separate PgBouncer pool in session mode for just those connections, while keeping the main pool in transaction mode.

**The complete fix for the 50-instance scenario:**
1. Deploy PgBouncer (often as a sidecar in Kubernetes, one per pod, or as a shared service)
2. Point application connection strings to PgBouncer
3. Set PgBouncer `default_pool_size` = 20-30 (actual DB connections)
4. Set application pool `max` = 10-20 per instance (connections to PgBouncer, cheap)
5. Set `pool_mode = transaction`
6. Monitor `pg_stat_activity` to verify actual connection count stays within limits

</details>

## Practical — Deployment Reliability

<details>
<summary>18. Configure a zero-downtime deployment for a Node.js service running in Kubernetes — show the Deployment spec with rolling update strategy, the application-level SIGTERM handler that stops accepting new requests and drains in-flight connections, the preStop hook, and proper terminationGracePeriodSeconds. Walk through the full shutdown sequence and explain what happens to requests in transit at each step</summary>

**Kubernetes Deployment spec:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # never reduce below desired count
      maxSurge: 1              # add 1 new pod before removing old
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      terminationGracePeriodSeconds: 60  # total time for graceful shutdown
      containers:
        - name: order-service
          image: order-service:1.2.3
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]  # critical — explained below
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

**Why the `preStop: sleep 5`:**

When Kubernetes decides to terminate a pod, two things happen *concurrently*:
1. The preStop hook begins executing
2. The pod is removed from the Service endpoints (load balancer stops routing to it)

The preStop hook must complete before SIGTERM is sent to the container. The endpoint removal is asynchronous — it takes a few seconds for kube-proxy/iptables/envoy to update. During that window, the load balancer may still send new requests to the terminating pod. The `preStop: sleep 5` delays SIGTERM delivery, giving the endpoint removal time to propagate. Without it, the pod stops accepting requests before the load balancer stops sending them, causing 502s.

**Application-level graceful shutdown:**

```typescript
import { createServer, Server, IncomingMessage, ServerResponse } from 'http';

const server = createServer(app);
let isShuttingDown = false;
let activeConnections = new Set<any>();

// Track active connections
server.on('connection', (socket) => {
  activeConnections.add(socket);
  socket.on('close', () => activeConnections.delete(socket));
});

function gracefulShutdown(signal: string) {
  console.log(`Received ${signal}. Starting graceful shutdown...`);
  isShuttingDown = true;

  // 1. Stop accepting new connections
  server.close((err) => {
    if (err) {
      console.error('Error closing server:', err);
      process.exit(1);
    }
    console.log('All connections drained. Exiting.');
    process.exit(0);
  });

  // 2. Set a hard deadline — if connections don't drain in time, force exit
  const FORCE_SHUTDOWN_MS = 50_000; // less than terminationGracePeriodSeconds
  setTimeout(() => {
    console.error('Forced shutdown — connections did not drain in time');
    process.exit(1);
  }, FORCE_SHUTDOWN_MS);

  // 3. Close idle keep-alive connections immediately
  for (const socket of activeConnections) {
    // Idle sockets (no in-flight request) can be destroyed immediately
    if (!(socket as any)._httpMessage) {
      socket.destroy();
    }
  }
}

// Health endpoint that reflects shutdown state
app.get('/health/ready', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting_down' });
  }
  res.status(200).json({ status: 'ready' });
});

app.get('/health/live', (_req, res) => {
  res.status(200).json({ status: 'alive' });
});

// Register shutdown handlers
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

server.listen(3000);
```

**The full shutdown sequence, step by step:**

```
Time 0s:  Kubernetes begins executing the preStop hook.
          Concurrently, Kubernetes starts removing the pod from Service endpoints.

Time 0-5s: preStop sleep runs. The application hasn't received SIGTERM yet.
           Load balancer endpoint removal propagates across the cluster.
           Existing requests continue being processed normally.
           New requests may still arrive (endpoint removal in progress).

Time 5s:  preStop sleep finishes. SIGTERM is delivered to the application.
           Application sets isShuttingDown = true.
           server.close() stops accepting NEW connections.
           Readiness probe returns 503 (belt-and-suspenders with endpoint removal).
           In-flight requests continue processing to completion.
           Idle keep-alive connections are closed immediately.

Time 5-55s: In-flight requests drain. Each request completes and its
            connection closes. server.close() callback fires when the
            last connection closes.

Time 55s: If connections haven't drained, force exit (FORCE_SHUTDOWN_MS).

Time 60s: Kubernetes sends SIGKILL if process is still running
          (terminationGracePeriodSeconds exceeded).
```

**What happens to requests in transit:**

| Request state at shutdown | What happens |
|---|---|
| Being processed by old pod | Completes normally — server.close() waits for in-flight requests |
| In load balancer queue, routed to old pod | preStop sleep gives time for endpoint removal. If it arrives before removal, it's processed. If after server.close(), gets connection refused → LB routes to another pod |
| New request from client | Routed to a new pod by the load balancer (old pod removed from endpoints) |
| Keep-alive connection, idle | Destroyed immediately during shutdown |
| Keep-alive connection, active request | Request completes, then connection closes |

**The critical numbers:**
- `preStop sleep` (5s) < `FORCE_SHUTDOWN_MS` (50s) < `terminationGracePeriodSeconds` (60s)
- This chain ensures: endpoint removal completes → app drains connections → forced exit if needed → Kubernetes kills if all else fails

</details>

<details>
<summary>19. A rolling deployment is causing intermittent 502 errors — walk through the diagnosis process: why do 502s happen during rollouts (premature pod termination before connections drain, readiness probe misconfiguration, load balancer not updated), how you'd identify which step in the rollout sequence is causing the issue, and what the specific fixes are for each root cause</summary>

**Why 502s happen during rollouts — the three common root causes:**

A 502 means the load balancer (or ingress controller) sent a request to a backend that couldn't handle it — the connection was refused, reset, or the backend sent an invalid response.

**Root cause 1: No preStop hook (most common)**

The pod receives SIGTERM and immediately stops accepting connections. But the endpoint removal from kube-proxy/iptables is asynchronous — it takes 1-5 seconds to propagate. During that window, the load balancer still routes to the terminating pod, which is already closed. Result: connection refused → 502.

*Diagnosis:*
- 502s correlate exactly with pod terminations (check `kubectl get events`)
- The errors are brief (1-5 seconds) and stop once the endpoint is removed
- Application logs show SIGTERM received immediately, no delay

*Fix:*
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

**Root cause 2: Readiness probe misconfiguration**

The new pod passes the readiness probe before it's actually ready to serve traffic (e.g., the HTTP server is listening but the database connection pool isn't initialized, or the application cache isn't warmed). Traffic arrives, the pod can't handle it, and returns errors.

*Diagnosis:*
- 502s/500s happen at the *start* of the rollout (new pods), not at the end (old pods terminating)
- New pod logs show errors like "database connection not established" or "cache not initialized"
- The probe is too simple (just returns 200 without checking dependencies)

*Fix:*
```typescript
app.get('/health/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');        // verify DB connection
    await redis.ping();                // verify Redis connection
    if (!cacheWarmed) {                // verify initialization complete
      return res.status(503).json({ status: 'warming' });
    }
    res.status(200).json({ status: 'ready' });
  } catch {
    res.status(503).json({ status: 'not_ready' });
  }
});
```

Also adjust probe timing:
```yaml
readinessProbe:
  initialDelaySeconds: 10    # give the app time to initialize
  periodSeconds: 5
  failureThreshold: 3        # 3 failures before removing from endpoints
  successThreshold: 1
```

**Root cause 3: Insufficient terminationGracePeriodSeconds**

Long-running requests (file uploads, report generation, WebSocket connections) take longer to drain than the grace period allows. Kubernetes sends SIGKILL, the process dies mid-request, and the client gets a connection reset → 502.

*Diagnosis:*
- 502s happen exactly at the `terminationGracePeriodSeconds` mark (default: 30s)
- Application logs show "forced shutdown" or the process simply disappears
- The affected requests are consistently long-running operations

*Fix:*
```yaml
terminationGracePeriodSeconds: 120  # increase for long-running requests
```
And ensure the application's force-shutdown timeout is less than this value.

**Diagnosis process (step by step):**

1. **Correlate timing**: Do 502s happen when old pods terminate (preStop/drain issue) or when new pods start (readiness issue)?
   ```bash
   # Check pod lifecycle events during rollout
   kubectl get events --sort-by='.lastTimestamp' | grep -E '(Killing|Unhealthy|Started)'
   ```

2. **Check for preStop hook**: `kubectl get deployment order-service -o yaml | grep -A5 preStop`. If missing, that's likely the cause.

3. **Check readiness probe**: `kubectl describe pod <new-pod>` — look for `Readiness probe failed` events. If the probe never fails but the pod isn't ready, the probe is too permissive.

4. **Check application logs during shutdown**:
   ```bash
   kubectl logs <terminating-pod> --previous  # logs from the killed container
   ```
   Look for: Did the app receive SIGTERM? Did it start draining? Did it get SIGKILL'd?

5. **Check grace period vs drain time**:
   ```bash
   kubectl get pod <pod> -o yaml | grep terminationGracePeriodSeconds
   ```
   Compare with the actual time your longest requests take.

6. **Check rollout strategy**:
   ```yaml
   # If maxUnavailable > 0, you're removing pods before new ones are ready
   maxUnavailable: 0  # safe default
   maxSurge: 1        # add new before removing old
   ```

**The complete checklist for 502-free rollouts:**
- `maxUnavailable: 0` in the rolling update strategy
- `preStop: sleep 5` on all containers
- Readiness probe that checks actual dependency health
- `terminationGracePeriodSeconds` > max expected request duration + preStop delay
- Application-level SIGTERM handler that drains connections
- `server.close()` called on SIGTERM (stops new connections, finishes in-flight)

</details>

<details>
<summary>20. Configure readiness probes for a Node.js service that depends on a database and a Redis cache — show how the readiness check verifies downstream dependencies are reachable, explain why readiness probes should check dependency health vs just returning 200, what happens when the probe fails (traffic removed but pod stays running), and how misconfigured readiness probes can cause cascading failures when a shared dependency goes down</summary>

**Readiness probe implementation:**

```typescript
interface HealthStatus {
  status: 'ready' | 'not_ready';
  checks: Record<string, { status: 'up' | 'down'; latencyMs: number }>;
}

class HealthController {
  constructor(
    private db: Pool,
    private redis: Redis,
  ) {}

  // Readiness: "Can this pod serve traffic?"
  async readiness(req: Request, res: Response) {
    const checks: HealthStatus['checks'] = {};
    let allHealthy = true;

    // Check database
    const dbStart = Date.now();
    try {
      await Promise.race([
        this.db.query('SELECT 1'),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('timeout')), 2000),
        ),
      ]);
      checks.database = { status: 'up', latencyMs: Date.now() - dbStart };
    } catch {
      checks.database = { status: 'down', latencyMs: Date.now() - dbStart };
      allHealthy = false;
    }

    // Check Redis
    const redisStart = Date.now();
    try {
      await Promise.race([
        this.redis.ping(),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('timeout')), 1000),
        ),
      ]);
      checks.redis = { status: 'up', latencyMs: Date.now() - redisStart };
    } catch {
      checks.redis = { status: 'down', latencyMs: Date.now() - redisStart };
      allHealthy = false;
    }

    const status: HealthStatus = {
      status: allHealthy ? 'ready' : 'not_ready',
      checks,
    };

    res.status(allHealthy ? 200 : 503).json(status);
  }

  // Liveness: "Is this pod stuck/crashed?" — deliberately simple
  liveness(req: Request, res: Response) {
    res.status(200).json({ status: 'alive' });
  }
}
```

**Kubernetes configuration:**

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3      # 3 consecutive failures → remove from endpoints
  successThreshold: 1      # 1 success → add back to endpoints
  timeoutSeconds: 3        # probe timeout (must be > health check timeout)
livenessProbe:
  httpGet:
    path: /health/live
    port: 3000
  initialDelaySeconds: 15
  periodSeconds: 15
  failureThreshold: 3      # 3 failures → restart the pod
  timeoutSeconds: 2
```

**Why readiness probes should check dependency health (not just return 200):**

A probe that always returns 200 tells Kubernetes "send me traffic" even when the pod can't actually serve requests. If the database is down, the pod accepts requests and immediately returns 500 errors. The user gets errors, the load balancer thinks the pod is healthy, and no traffic reroutes to pods that might be in a better state (e.g., pods in a different AZ where the database is reachable).

A dependency-aware readiness probe removes the pod from the load balancer when it can't fulfill requests, so traffic flows only to pods that can actually serve it.

**What happens when the probe fails:**

1. Pod fails readiness probe 3 times consecutively (30 seconds with above config)
2. Kubernetes removes the pod from the Service's Endpoints
3. The load balancer stops routing new traffic to this pod
4. **The pod keeps running** — it's not restarted (that's what liveness probes are for)
5. The pod continues to execute readiness probes
6. When the dependency recovers and the probe passes, the pod is added back to Endpoints
7. Traffic resumes to this pod

This is the critical difference: readiness removes traffic, liveness restarts the pod. A readiness failure is "I can't help right now," not "I'm broken."

**The cascading failure trap with shared dependencies:**

The dangerous scenario: 10 pods all depend on the same Redis instance. Redis has a brief network blip (2 seconds). All 10 pods fail their readiness probes. Kubernetes removes all 10 from the Service. Now the service has **zero** ready pods. All incoming traffic gets 503'd even though the application code is perfectly healthy and Redis will be back in seconds.

When Redis recovers, all 10 pods pass their next readiness check and come back online simultaneously — potentially causing a thundering herd on the database as cached data is refetched.

**How to prevent cascading readiness failures:**

1. **Don't check non-critical dependencies in readiness:**

```typescript
async readiness(req: Request, res: Response) {
  // Only check critical dependencies that make the service unable to
  // serve ANY useful response
  const dbHealthy = await this.checkDatabase();

  // Redis is nice-to-have — the service can fall back to DB for cache misses
  // Don't include it in readiness

  res.status(dbHealthy ? 200 : 503).json({ database: dbHealthy });
}
```

2. **Use degraded-but-ready semantics:**

```typescript
async readiness(req: Request, res: Response) {
  const dbHealthy = await this.checkDatabase();
  const redisHealthy = await this.checkRedis();

  // Only fail readiness for dependencies without fallbacks
  const canServeTraffic = dbHealthy; // Redis has a fallback, DB doesn't

  res.status(canServeTraffic ? 200 : 503).json({
    database: dbHealthy ? 'up' : 'down',
    redis: redisHealthy ? 'up' : 'down',
    degraded: !redisHealthy, // signal that we're running in degraded mode
  });
}
```

3. **Tune failure thresholds conservatively:**

```yaml
readinessProbe:
  failureThreshold: 5    # not 1 — tolerate brief blips
  periodSeconds: 10      # 5 × 10 = 50 seconds before removal
```

**The decision framework for what to check in readiness:**

| Dependency | Has fallback? | Include in readiness? |
|---|---|---|
| Primary database | No — can't serve any requests without it | Yes |
| Redis cache | Yes — can fall back to database | No |
| External API (recommendations) | Yes — can return defaults | No |
| Message broker (for writes) | Partial — can queue locally | Maybe (depends on criticality) |
| Auth service | No — can't authenticate requests | Yes |

Rule: Only fail readiness for dependencies where failure means the pod **cannot serve any useful response at all**. If the pod can serve degraded responses, keep it ready and let the application handle the degradation.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>21. After scaling a service from 3 to 20 instances, you observe a sudden spike of identical requests hitting the database — diagnose this as a cache stampede, explain why it happens specifically during horizontal scale-out (cold caches on new instances, synchronized TTL expiry), and show the concrete fixes: request coalescing, staggered TTLs, cache warming on startup, and probabilistic early recomputation</summary>

**Diagnosis: Cache stampede**

You see a spike of identical `SELECT` queries for the same data (product catalog, configuration, popular items) hitting the database simultaneously. The spike correlates exactly with the scale-out event. This is a cache stampede — multiple instances all miss the cache at the same time and all hit the database for the same data.

**Why horizontal scale-out specifically triggers this:**

1. **Cold caches on new instances**: If the service uses any in-memory caching (even as a first-level cache before Redis), 17 new instances start with empty caches. The first request to each instance for any cached data becomes a cache miss → database hit. With 17 instances and 100 popular keys, that's potentially 1,700 simultaneous queries for data that 3 instances were serving from cache.

2. **Synchronized TTL expiry**: Even with a shared cache (Redis), if all cache entries were populated at the same time (e.g., after a deploy or cache flush), they all expire at the same time. With 20 instances, 20 concurrent requests for the same expired key each independently query the database — a classic thundering herd.

3. **Autoscaling timing**: Autoscaling events tend to add multiple instances at once, not one at a time. The burst of cold instances creates a correlated cache miss storm.

**Fix 1: Request coalescing (singleflight pattern)**

Multiple concurrent requests for the same cache key share a single database query instead of each issuing its own.

```typescript
class CoalescingCache {
  private inFlight = new Map<string, Promise<any>>();

  constructor(private redis: Redis, private db: Pool) {}

  async get<T>(key: string, fetchFn: () => Promise<T>, ttlSeconds: number): Promise<T> {
    // Check cache first
    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached);

    // If someone is already fetching this key, wait for their result
    const existing = this.inFlight.get(key);
    if (existing) return existing;

    // This instance is the first to miss — fetch and share the result
    const promise = fetchFn().then(async (result) => {
      await this.redis.setex(key, ttlSeconds, JSON.stringify(result));
      this.inFlight.delete(key);
      return result;
    }).catch((err) => {
      this.inFlight.delete(key);
      throw err;
    });

    this.inFlight.set(key, promise);
    return promise;
  }
}

// Usage
const cache = new CoalescingCache(redis, db);
const product = await cache.get(
  `product:${id}`,
  () => db.query('SELECT * FROM products WHERE id = $1', [id]),
  3600,
);
```

This coalesces within a single instance. For cross-instance coalescing, use a distributed lock (Redis `SET NX` with short TTL) so only one instance across the entire fleet fetches from the database.

**Fix 2: Staggered TTLs**

Prevent synchronized expiry by adding random jitter to TTL values.

```typescript
function ttlWithJitter(baseTtlSeconds: number, jitterPercent = 0.1): number {
  const jitter = baseTtlSeconds * jitterPercent;
  return Math.floor(baseTtlSeconds + Math.random() * jitter * 2 - jitter);
}

// Base TTL: 3600s. Actual TTL: 3240s - 3960s (±10%)
await redis.setex(key, ttlWithJitter(3600), value);
```

This spreads expirations across a time window instead of all hitting at once.

**Fix 3: Cache warming on startup**

Pre-populate the cache before the instance starts accepting traffic. Combine with readiness probes so traffic doesn't arrive until the cache is warm.

```typescript
async function warmCache(): Promise<void> {
  // Fetch the top N most-accessed keys
  const hotKeys = await db.query(
    'SELECT id, data FROM products WHERE popular = true LIMIT 500',
  );

  await Promise.all(
    hotKeys.rows.map((row) =>
      redis.setex(`product:${row.id}`, ttlWithJitter(3600), JSON.stringify(row)),
    ),
  );

  console.log(`Cache warmed with ${hotKeys.rows.length} entries`);
}

// In startup sequence
await warmCache();
cacheWarmed = true; // readiness probe now returns 200
server.listen(3000);
```

**Fix 4: Probabilistic early recomputation (XFetch)**

Before the cache entry expires, proactively recompute it with increasing probability as the expiry approaches. This avoids the cliff edge of expiration.

```typescript
async function getWithEarlyRecompute<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttlSeconds: number,
): Promise<T> {
  const raw = await redis.get(key);
  if (raw) {
    const { value, expiresAt, delta } = JSON.parse(raw);
    const remaining = expiresAt - Date.now();

    // Probabilistic early recompute: as expiry approaches, probability increases
    // delta = time the last fetch took (in ms)
    const shouldRecompute =
      remaining > 0 &&
      remaining < delta * 10 * Math.log(Math.random()) * -1;

    if (!shouldRecompute) return value;
    // Fall through to recompute
  }

  const start = Date.now();
  const value = await fetchFn();
  const delta = Date.now() - start;

  await redis.setex(
    key,
    ttlSeconds,
    JSON.stringify({
      value,
      expiresAt: Date.now() + ttlSeconds * 1000,
      delta,
    }),
  );

  return value;
}
```

**Which fixes to apply:**

| Fix | Effort | Impact | When to use |
|---|---|---|---|
| Staggered TTLs | Low | Medium | Always — no reason not to |
| Request coalescing | Medium | High | When you have hot keys accessed by many concurrent requests |
| Cache warming | Medium | High | When you know your hot key set and can precompute it |
| Probabilistic early recompute | High | High | High-traffic systems where even brief stampedes cause problems |

Start with staggered TTLs + request coalescing. Add cache warming if you can identify your hot key set. Probabilistic early recompute is for high-scale systems where the other fixes aren't sufficient.

</details>

<details>
<summary>22. Service A calls Service B, which calls Service C. Service C starts responding slowly (2s instead of 200ms). Within minutes, all three services are failing — walk through exactly how retry amplification caused this cascade: how retries at each layer multiplied the load, why timeouts were configured incorrectly, and show the specific changes to retry budgets, per-layer timeouts, and circuit breaker thresholds that would have prevented this</summary>

**The cascade, step by step:**

**Initial state (before the incident):**
- C's normal response time: 200ms
- All three services have independent 5s timeouts and 3 retries
- No circuit breakers configured
- No retry budgets

**T+0: C starts responding slowly (2s instead of 200ms)**

Something happened to C — maybe a slow query, GC pause, or resource contention. Each request now takes 2s.

**T+1 min: B's connection pool starts saturating**

B is waiting 2s per call to C instead of 200ms. B's connection pool (sized for 200ms calls) now holds connections 10x longer. Pool fills up. New requests to B start queueing, waiting for a connection.

B retries failed/timed-out calls to C 3 times. Each retry waits up to 5s. So a single request from A → B can generate up to 4 calls to C (1 original + 3 retries), each taking 2-5s. That's up to 20 seconds of connection hold time per request.

**T+2 min: A starts timing out on calls to B**

B is now slow because its connections to C are saturated. A's calls to B start timing out. A retries 3 times. Each retry to B generates up to 4 calls to C.

**Load multiplication:** With 3 retries at each layer, this produces the 4×4 = 16x amplification covered in Q8. At 100 RPS to A, C receives up to 1,600 RPS. C was already struggling at normal load; 16x load guarantees it collapses entirely.

**T+3 min: Full cascade**

- C is completely overwhelmed, returning 503s or timing out
- B's entire connection pool is exhausted waiting for C, so B can't serve any requests (including ones that don't need C)
- A's connection pool to B is exhausted, so A can't serve any requests
- All three services are returning 5xx errors

**Why the timeouts were wrong:**

The 5s timeout at every layer is independent — each service waits the full 5 seconds before giving up. With retries:
- A waits up to 5s × 4 attempts = 20s before giving up on B
- B waits up to 5s × 4 attempts = 20s before giving up on C
- A user request can theoretically take 20s × 4 = 80 seconds of system time

Meanwhile, the user gave up after 3 seconds and closed the tab. All that work is wasted.

**The fixes:**

**Fix 1: Timeout budget propagation (as covered in question 8)**

```
A receives request with 3s user-facing deadline
A reserves 500ms for its own work → passes 2.5s to B
B reserves 300ms for its own work → passes 2.2s to C
C must respond in 2.2s or fail
```

If C takes 2s, there's just enough time. If C takes 2.5s, B fails fast and doesn't waste A's remaining budget.

```typescript
// A's call to B
const deadline = Date.now() + 3000;
const response = await fetch('http://service-b/endpoint', {
  headers: { 'X-Request-Deadline': String(deadline) },
  signal: AbortSignal.timeout(2500), // 3s - 500ms overhead
});
```

**Fix 2: Per-layer timeouts that decrease downstream**

```
A → B timeout: 3s (user-facing, generous)
B → C timeout: 1s (internal, tight — C should be fast)
```

If C is slow, B fails fast in 1s instead of holding connections for 5s. This limits the blast radius.

**Fix 3: Retry budgets (as covered in question 8)**

```typescript
// Each service limits retries to 20% of total traffic
// If C is failing 50% of requests, B stops retrying after budget is exhausted
const retryBudget = new RetryBudget(20); // 20% budget

// Only retry at the EDGE (Service A), not at intermediate layers
// B should NOT retry calls to C — let A decide whether to retry the whole chain
```

**Rule: Only retry at the outermost layer.** Intermediate services should fail fast and propagate errors upward. This eliminates the multiplicative effect entirely.

**Fix 4: Circuit breakers on B → C**

```typescript
// B's circuit breaker for calls to C
const circuitBreaker = new CircuitBreaker({
  failureThreshold: 5,      // 5 failures in the window
  failureWindow: 10_000,    // 10 second window
  resetTimeout: 15_000,     // try again after 15s
  halfOpenRequests: 2,      // 2 probe requests in half-open
});
```

After 5 failures, B stops calling C entirely and returns a failure (or fallback) immediately. This:
- Stops the load amplification on C, giving it room to recover
- Frees B's connection pool so B can serve requests that don't need C
- Propagates the failure cleanly to A, which can decide what to do

**Fix 5: Bulkhead on B**

B should isolate its connection pool for C from other dependencies (as covered in question 7). If C's pool is saturated, B's calls to other healthy services (D, E) continue unaffected.

**The corrected architecture:**

```
A (edge):
  - User-facing timeout: 3s
  - Retry: 2 attempts with backoff + jitter
  - Retry budget: 20%

B (intermediate):
  - Timeout to C: 1s
  - NO retries to C (fail fast, let A retry the whole call)
  - Circuit breaker: 5 failures in 10s → open for 15s
  - Bulkhead: separate connection pool for C (max 20 concurrent)

C:
  - Internal timeout budget: honor X-Request-Deadline header
  - Shed load when overloaded (return 503 early rather than processing slowly)
```

With these fixes, the same scenario plays out differently: C slows down → B's circuit breaker trips after 5 failures → B immediately fails subsequent calls to C → A sees failures and retries (hitting a different B instance or getting the same fast failure) → user gets an error in 3 seconds instead of waiting 80 seconds → C is under normal load (no amplification) and recovers in seconds.

</details>

<details>
<summary>23. A message queue's depth is growing rapidly and consumer lag is increasing — walk through how you'd diagnose whether this is a backpressure problem (consumers can't keep up) vs a traffic spike (producers sending more than normal), what immediate load shedding steps you'd take to prevent the queue from overwhelming downstream services, and how you'd set up monitoring and alerts to catch this earlier next time</summary>

**Step 1: Determine if it's a producer spike or consumer slowdown**

Check two metrics side by side:

- **Producer publish rate** (messages/second into the queue)
- **Consumer processing rate** (messages/second consumed)

```
Scenario A — Traffic spike:
  Producer rate: 5,000 msg/s (normally 1,000)
  Consumer rate: 1,000 msg/s (normal)
  → Producers are flooding the queue. Consumers are fine.

Scenario B — Consumer slowdown:
  Producer rate: 1,000 msg/s (normal)
  Consumer rate: 200 msg/s (normally 1,000)
  → Consumers are degraded. Something downstream is slow.

Scenario C — Both:
  Producer rate: 3,000 msg/s (elevated)
  Consumer rate: 500 msg/s (degraded)
  → Both sides are contributing.
```

**How to check:**

1. **Queue broker metrics**: RabbitMQ Management UI, AWS SQS CloudWatch, or Kafka consumer group lag. These show publish rate, consume rate, and depth over time.
2. **Consumer processing time**: Check the consumer's per-message processing duration. If it jumped from 50ms to 2s, the consumer (or its downstream dependencies) is the bottleneck.
3. **Consumer error rate**: If consumers are failing and nacking messages, the same messages are redelivered repeatedly, inflating apparent queue depth.
4. **Consumer count**: Did a consumer pod crash or get OOM-killed? `kubectl get pods` — if consumer replicas dropped, that explains reduced throughput.

**Step 2: Immediate load shedding**

The priority is preventing the growing queue from overwhelming downstream services when consumers catch up (a burst of processing after the backlog clears can be as damaging as the original problem).

**If it's a producer spike:**

```typescript
// At the producer's HTTP ingestion endpoint — reject low-priority work
app.post('/events', async (req, res) => {
  const queueDepth = await queue.getDepth();

  if (queueDepth > CRITICAL_THRESHOLD) {
    if (req.body.priority !== 'high') {
      return res.status(429).json({
        error: 'System under load',
        retryAfter: 60,
      });
    }
  }

  await queue.publish(req.body);
  res.status(202).json({ accepted: true });
});
```

**If it's a consumer slowdown:**

1. **Scale consumers** if the bottleneck is consumer count (not a downstream dependency):
   ```bash
   kubectl scale deployment order-consumer --replicas=10
   ```

2. **If a downstream dependency is slow**, scaling consumers just adds more load on the already-struggling dependency. Instead:
   - Reduce consumer concurrency to limit downstream load
   - Enable circuit breaker on the consumer's downstream call
   - Temporarily pause consumption and fix the downstream issue first

3. **Shed aged messages**: Messages older than a threshold are likely stale and processing them wastes resources.
   ```typescript
   async function processMessage(msg: QueueMessage) {
     const ageMs = Date.now() - msg.timestamp;
     if (ageMs > MAX_MESSAGE_AGE_MS) {
       metrics.increment('messages.expired');
       await msg.ack(); // discard stale message
       // Optionally move to dead-letter queue for later analysis
       return;
     }
     await handleMessage(msg);
   }
   ```

4. **Priority-based shedding** (as implemented in question 15): Drop low-priority messages, process high-priority only.

**Step 3: Set up monitoring and alerts for next time**

| Metric | Alert condition | Why |
|---|---|---|
| Queue depth | > 2x normal peak, sustained 5 min | Early warning before it becomes critical |
| Consumer lag (Kafka) | Lag increasing for 10 min | Consumers falling behind |
| Consumer processing time p99 | > 3x baseline | Downstream dependency slowing down |
| Consumer error rate | > 5% of messages | Nack/retry loops inflating queue |
| Producer publish rate | > 3x normal | Unexpected traffic spike |
| Queue depth rate of change | Growing > 100 msg/s for 5 min | Queue is filling faster than draining |

**Key alerts to configure:**

```yaml
# Prometheus alerting rules
groups:
  - name: queue-health
    rules:
      - alert: QueueDepthHigh
        expr: rabbitmq_queue_messages > 5000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Queue depth is {{ $value }} (threshold: 5000)"

      - alert: QueueDepthCritical
        expr: rabbitmq_queue_messages > 20000
        for: 2m
        labels:
          severity: critical

      - alert: ConsumerLagIncreasing
        expr: rate(rabbitmq_queue_messages[5m]) > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Queue depth has been growing for 10 minutes"

      - alert: ConsumerProcessingSlowdown
        expr: histogram_quantile(0.99, rate(message_processing_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
```

**Dashboard essentials:**

- **Side-by-side graph**: producer rate vs consumer rate. When they diverge, depth grows.
- **Queue depth with trend line**: Not just current depth but the rate of change. A queue at 1,000 but growing at 500/s is more urgent than a queue at 5,000 but stable.
- **Consumer processing time histogram**: Shows if the distribution is shifting (all consumers slow) or bimodal (some consumers hitting a bad downstream instance).
- **Dead-letter queue depth**: Messages that repeatedly fail end up here. A growing DLQ means a persistent processing bug, not a capacity issue.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you had to scale a system to handle significantly more load than it was designed for — what was the trigger, what scaling approach did you choose (horizontal, vertical, architectural changes), what unexpected problems did you hit during the scaling process, and what would you do differently?</summary>

**What the interviewer is looking for:**

- Systematic approach to scaling, not just "we added more servers"
- Ability to identify the actual bottleneck before throwing resources at it
- Awareness of cascading effects of scaling decisions (DB connections, cache behavior, state management)
- Learning from unexpected problems — honest reflection, not a polished success story

**Suggested structure (STAR with depth):**

1. **Situation + trigger**: What was the system, what load was it designed for, and what changed? (New customer onboarding, viral traffic spike, seasonal peak, business growth)
2. **Analysis before action**: How did you identify the bottleneck? What metrics told you where to scale? (Don't skip this — it shows engineering judgment)
3. **Approach chosen and why**: Did you scale horizontally, vertically, or make architectural changes? What alternatives did you consider and reject?
4. **Unexpected problems**: This is the most important part. What broke that you didn't anticipate? (Database connection exhaustion, cache stampede, hot partitions, state management issues, deployment pipeline bottlenecks)
5. **Resolution and outcome**: How did you fix the unexpected problems? What was the final architecture?
6. **What you'd do differently**: Concrete improvements — not vague "better planning" but specific technical or process changes

**Example outline to personalize:**

- **Trigger**: A major customer onboarded, bringing 5x the event volume to our audit log system
- **Analysis**: Monitored CPU (fine), DB connections (approaching limit), consumer lag (growing). Bottleneck was database write throughput, not compute.
- **Approach**: Scaled consumers horizontally to process events faster. Also vertically scaled the database instance.
- **Unexpected problem**: Horizontal scaling of consumers caused DB connection exhaustion (20 consumers x 10 connections = 200, exceeding the database's 100 connection limit). Also hit a cache stampede on configuration data as new consumer instances started with cold caches.
- **Fix**: Added PgBouncer for connection pooling, implemented cache warming on consumer startup, and added connection pool monitoring alerts.
- **What I'd do differently**: Would have load-tested at the target scale first instead of scaling in production. Would have added connection pool metrics and alerts before scaling, not after the incident.

**Key points to hit:**

- Show you measured before scaling (not guessing)
- Mention the specific bottleneck (CPU, DB, network, memory) — vague answers signal lack of experience
- The unexpected problem should be technically specific (not "things broke")
- The retrospective should show growth: "I now always check X before scaling"

</details>

<details>
<summary>25. Describe a time you debugged a cascading failure in a distributed system — what were the symptoms, how did you trace the root cause across services, what was the fix, and what resilience patterns did you put in place afterward to prevent recurrence?</summary>

**What the interviewer is looking for:**

- Methodical debugging under pressure (not random guessing)
- Ability to trace causality across service boundaries using observability tools
- Understanding of *why* cascading failures happen (not just what happened)
- Proactive hardening after the incident — resilience patterns chosen for specific reasons

**Suggested structure:**

1. **Symptoms**: What alerted you? What did the dashboards/alerts show? Be specific — "all services returning 5xx" is vague; "Service A latency p99 spiked from 200ms to 30s, followed by B's error rate jumping to 60% two minutes later" shows real debugging.
2. **Initial hypothesis and investigation**: What did you check first? How did you narrow down the root cause? (Distributed tracing, log correlation, dependency graphs, metrics timelines)
3. **Root cause discovery**: What was actually failing, and why did it cascade? (Slow dependency without timeout, retry amplification, shared resource exhaustion, missing circuit breaker)
4. **Immediate fix**: What did you do to stop the bleeding? (Kill/restart, feature flag off, circuit breaker manually tripped, scale down a component)
5. **Resilience patterns added**: What you put in place afterward — tied to the specific failure mode, not a generic list of patterns.
6. **Process improvements**: Runbooks, alerts, load testing, chaos engineering — what changed beyond code?

**Example outline to personalize:**

- **Symptoms**: Alert fired on elevated 5xx rates for the order service. Customer support reported checkout failures. Dashboard showed order service latency climbing from 300ms to 15s over 5 minutes.
- **Investigation**: Traced a slow request — distributed tracing showed 90% of the time was spent waiting on the inventory service. Inventory service logs showed it was waiting on a third-party supplier API that had gone unresponsive. No timeout was configured on that call.
- **Root cause**: Supplier API hung indefinitely. Inventory service held connections open waiting for responses. Connection pool exhausted. Order service retried calls to inventory 3 times (retry amplification). Order service's own connection pool filled up waiting for inventory. Entire checkout flow blocked.
- **Immediate fix**: Restarted inventory service to clear stuck connections. Temporarily bypassed real-time inventory check (used cached stock levels).
- **Patterns added**: (1) 2-second timeout on all outbound calls in inventory service. (2) Circuit breaker on the supplier API call — trips after 5 failures, falls back to cached inventory. (3) Bulkhead isolating supplier API connections from other inventory service calls. (4) Removed retries from order → inventory (fail fast, let the user retry).
- **Process changes**: Added runbook for "inventory service degraded" with clear steps. Added alert on supplier API latency p99. Added synthetic monitoring that calls the supplier API every 30 seconds.

**Key points to hit:**

- Show the timeline and how you correlated events across services
- Name the specific tools you used (Grafana, Jaeger/Zipkin, Kibana, `kubectl logs`)
- Explain the causal chain — not just "A failed then B failed" but *why* A's failure caused B's failure (connection exhaustion, retry amplification, shared resource)
- Each resilience pattern added should map to a specific failure mode you observed — don't list patterns you added "just in case"

</details>

<details>
<summary>26. Tell me about a time you implemented graceful degradation for a production service — what dependencies were involved, how did you decide what to degrade and in what order, and how did the degradation strategy perform when it was actually needed?</summary>

**What the interviewer is looking for:**

- Intentional design thinking — degradation was planned, not improvised during an incident
- Business-aware prioritization — understanding which features matter most to users and revenue
- Technical implementation details — not just "we added feature flags" but how the system actually behaves under partial failure
- Real-world validation — did the strategy actually work when needed?

**Suggested structure:**

1. **Context**: What service, what dependencies, why was graceful degradation needed? (Previous incident, reliability requirements, SLA commitments)
2. **Dependency mapping**: What did the service depend on? Which dependencies were critical vs nice-to-have?
3. **Degradation tiers**: How did you prioritize what to shed? Who was involved in that decision (engineering, product, business)?
4. **Implementation**: Feature flags, circuit breakers, fallback responses, read-only mode — concrete technical approach
5. **Testing the degradation paths**: How did you verify the fallbacks worked? (Chaos testing, dependency kill switches, regular drills)
6. **Real-world performance**: When degradation was actually triggered, what happened? Did it work as expected? What surprised you?

**Example outline to personalize:**

- **Context**: Our product page service called 5 downstream services. A previous incident where the recommendation engine went down caused the entire product page to 500, even though recommendations are a small sidebar feature. We decided to implement proper graceful degradation.
- **Dependency mapping**: (1) Product catalog — critical, no fallback possible. (2) Pricing service — critical for purchase. (3) Inventory — important but can show "check availability." (4) Recommendations — nice-to-have, has natural fallback (popular products). (5) Reviews — nice-to-have, can hide section.
- **Degradation tiers**: Worked with product manager to define three tiers. Tier 1 (shed first): recommendations, reviews. Tier 2: personalized pricing (fall back to list price). Tier 3 (protect): catalog, basic pricing, authentication. Documented in a runbook.
- **Implementation**: Used circuit breakers with fallback functions for each non-critical dependency. Feature flags for manual override during extended outages. Each dependency had its own timeout and connection pool (bulkhead pattern). Response included a `degraded: true` field so the frontend could adjust UI.
- **Testing**: Monthly "dependency kill" tests where we blocked traffic to one dependency in staging and verified the degradation path. Also ran in production with a specific test product that had all dependencies exercised.
- **Real-world result**: Three months later, the review service had a database migration go wrong and was down for 45 minutes. The circuit breaker tripped in 15 seconds, the review section disappeared from product pages, and no customers noticed. Previously this would have been a P1 incident.

**Key points to hit:**

- Show collaboration with product/business on prioritization (not a purely engineering decision)
- Mention testing the degradation paths — untested fallbacks are worse than no fallbacks (they give false confidence)
- Be honest about what didn't work perfectly — maybe a fallback returned stale data longer than expected, or the feature flag system itself had a brief outage
- Emphasize the outcome: reduced blast radius, faster recovery, or avoided a customer-facing incident

</details>
