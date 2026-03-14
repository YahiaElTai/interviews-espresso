# System Design Fundamentals

> **27 questions**

- Interview framework: requirement gathering (functional/non-functional), estimation, API definition, high-level design, deep dive — pacing and interviewer communication
- Back-of-the-envelope estimation: powers of two, QPS, storage, bandwidth, latency numbers
- Load balancers: L4 vs L7, algorithms (round-robin, least connections, consistent hashing, weighted), reverse proxy overlap
- CDNs: push vs pull models, cache invalidation (TTL, purge, versioned URLs)
- Caching layers and strategies: cache-aside, write-through, write-behind, refresh-ahead
- Message queues and task queues: decoupling, load leveling, retry, ordering, backpressure
- Idempotency: idempotency keys, at-least-once vs exactly-once delivery, safe retries in distributed systems
- Consistency models: strong, eventual, causal, read-your-writes, CAP theorem and PACELC extension
- Database replication: single-leader, multi-leader, leaderless
- SQL vs NoSQL selection criteria
- Database scaling: horizontal vs vertical scaling, read replicas, sharding, federation, denormalization
- Sharding strategies: hash-based, range-based, directory-based, consistent hashing
- Communication protocols: HTTP/REST, gRPC, WebSockets, SSE, UDP — choosing the right protocol for a given scenario
- Rate limiting: token bucket, sliding window, fixed window — client-side vs server-side, distributed rate limiting
- Distributed coordination: leader election, consensus (Raft/Paxos at a high level), distributed locks, ZooKeeper/etcd use cases
- Failure modes: network partitions, cascading failures, single points of failure
- Designing for the common case: read-heavy vs write-heavy identification, hot spots, avoiding premature optimization, latency vs throughput tradeoffs

---

## Interview Framework & Estimation

<details>
<summary>1. How should you structure a system design interview from start to finish — what are the phases (requirement gathering, estimation, API definition, high-level design, deep dive), why does this structure exist, how do you decide how long to spend on each phase, and what signals tell you you're going off track?</summary>

**The five phases** (for a typical 45-minute interview):

**1. Requirement Gathering (5-7 min)** — Clarify functional requirements (what the system does) and non-functional requirements (scale, latency, availability, consistency). Ask about users, traffic, data size, geographic distribution. The goal is to narrow scope — you cannot design "all of Twitter" in 45 minutes, so agree with the interviewer on which parts matter.

**2. Back-of-the-Envelope Estimation (3-5 min)** — Convert requirements into numbers: QPS, storage, bandwidth. This grounds the design in reality and tells you early whether you need sharding, caching, or CDNs. Skip this if the interviewer signals they want to move on.

**3. API Definition (3-5 min)** — Define the core API endpoints or interfaces. This forces you to think about the system from the client's perspective and locks down the contract before you design the internals. It also surfaces read/write ratios and access patterns.

**4. High-Level Design (10-15 min)** — Draw the major components: clients, load balancers, services, databases, caches, queues. Show data flow for the core use cases. This is the skeleton — don't go deep on any one component yet.

**5. Deep Dive (15-20 min)** — Pick 2-3 areas that are most interesting or challenging and go deep. The interviewer often guides this. This is where you show depth: sharding strategy, consistency model, failure handling, specific algorithm choices.

**Why this structure exists:** It mirrors how senior engineers actually think about systems — start broad, constrain the problem, then go deep where it matters. It also prevents the common failure mode of jumping straight into implementation details without understanding the problem.

**How to allocate time:** The deep dive gets the most time because it's where you demonstrate senior-level thinking. If you're spending more than 7 minutes on requirements, you're probably not being decisive enough. If you haven't started the high-level design by minute 15, you're behind.

**Signals you're off track:**
- The interviewer is redirecting you or asking "what about X?" — they're steering you toward something you missed
- You've been talking about one component for 10 minutes without the interviewer engaging — move on
- You haven't drawn anything by minute 15 — start sketching
- You're designing for edge cases before the happy path works — step back

**Key principle:** Communicate constantly. State your assumptions out loud. Say "I'm going to focus on X because..." before diving deep. This lets the interviewer course-correct early instead of watching you go down the wrong path for 10 minutes.

</details>

<details>
<summary>2. Why is back-of-the-envelope estimation important in a design interview, and how do you calculate QPS, storage, and bandwidth from product requirements -- walk through a concrete example showing how you go from "10 million daily active users" to specific numbers, why should you memorize powers of two (2^10 through 2^40) as estimation anchors, and what mistakes do candidates commonly make in estimation?</summary>

**Why estimation matters:** It transforms vague requirements into concrete constraints that drive design decisions. "We need to handle a lot of traffic" is useless. "We need 12K QPS with 500ms p99 latency" tells you whether a single database can handle it, whether you need caching, and what kind of sharding strategy to use.

**Concrete example — "10 million DAU" social media feed:**

**QPS calculation:**
- 10M DAU, assume each user makes ~20 requests/day (feed loads, likes, posts)
- Total requests/day = 10M * 20 = 200M
- QPS = 200M / 86,400 seconds ~= 2,300 QPS (average)
- Peak QPS = 2-5x average ~= 5,000-12,000 QPS

**Storage calculation:**
- Assume 500K new posts/day, average post size = 1 KB (text + metadata)
- Daily storage = 500K * 1 KB = 500 MB/day
- Monthly = ~15 GB, yearly = ~180 GB (text only — manageable on a single node)
- Images/media change this drastically: 500K posts * 20% with images * 2 MB avg = 200 GB/day

**Bandwidth:**
- Read bandwidth = 2,300 QPS * 10 KB avg response = 23 MB/s = ~184 Mbps (average)
- Write bandwidth = much smaller (posts are less frequent than reads)

**Powers of two you should know:**

| Power | Value | Useful for |
|-------|-------|------------|
| 2^10 | ~1 thousand (1 KB) | Small text records |
| 2^20 | ~1 million (1 MB) | Images, small files |
| 2^30 | ~1 billion (1 GB) | Database tables in memory |
| 2^40 | ~1 trillion (1 TB) | Large database on disk |

These are mental anchors for quick math. Knowing that 1 million seconds is ~11.5 days helps you convert between time units quickly.

**Other useful anchors:**
- 86,400 seconds/day (~10^5 for quick math)
- 2.5 million seconds/month (~2.5 * 10^6)
- A single machine can handle ~10K-50K simple requests/sec (depending on workload)
- A single PostgreSQL instance can handle ~10K-50K QPS for simple reads

**Common mistakes:**
- **False precision:** Saying "11,574 QPS" when the assumptions are rough. Round aggressively — the point is order-of-magnitude reasoning.
- **Forgetting peak vs average:** Systems must handle peak load, not average. Peak is typically 2-5x average, sometimes 10x for spiky workloads.
- **Ignoring read vs write ratio:** Most systems are heavily read-biased (10:1 to 100:1). This changes everything about your caching and replication strategy.
- **Spending too long:** Estimation should take 3-5 minutes. Get ballpark numbers and move on. You can refine later.
- **Not connecting numbers to decisions:** The estimation should conclude with "...and this tells us we need X" (e.g., "12K QPS means a single database won't work, we need read replicas or caching").

</details>

<details>
<summary>3. What are the key latency numbers every engineer should know (memory read, SSD seek, network round-trip, disk sequential read) and how do these numbers influence design decisions -- when do they tell you "this must be cached," "this can hit the database on every request," or "this needs to be colocated"?</summary>

**Key latency numbers (approximate orders of magnitude):**

| Operation | Latency | Order |
|-----------|---------|-------|
| L1 cache reference | 0.5 ns | |
| L2 cache reference | 7 ns | |
| Main memory reference | 100 ns | |
| SSD random read | 150 μs | 1000x memory |
| HDD random read | 10 ms | 100x SSD |
| SSD sequential read (1 MB) | 1 ms | |
| HDD sequential read (1 MB) | 20 ms | |
| Network round-trip (same datacenter) | 0.5 ms | |
| Network round-trip (cross-region, US) | 40-80 ms | |
| Network round-trip (cross-continent) | 100-200 ms | |
| Mutex lock/unlock | 25 ns | |
| Send 1 KB over 1 Gbps network | 10 μs | |

**How these numbers drive design decisions:**

**"This must be cached":** If your database query takes 5-20 ms and you need sub-millisecond response times, you need an in-memory cache (Redis/Memcached — memory reference is ~100ns, network hop to Redis is ~0.5ms). If you're making the same query thousands of times per second, the math is obvious: caching eliminates most of those 5-20ms hits.

**"This can hit the database on every request":** If your latency budget is 200ms and the database query is 5ms, you have plenty of headroom. For internal tools, admin panels, or low-QPS endpoints, going directly to the database is simpler and fine. Don't cache prematurely.

**"This needs to be colocated":** Cross-region round-trips of 80-200ms mean that if a single request requires 3-4 sequential calls to a remote service, you're already at 300-800ms just from network latency. This tells you either: colocate the services in the same region, batch the calls, or use a CDN to bring data closer to users.

**Practical implications for system design interviews:**
- If users are global and you need < 100ms responses, you need CDN and/or multi-region deployment — physics makes cross-continent round-trips 100ms+ unavoidable
- Sequential disk reads are ~20x faster than random reads — this is why append-only logs (Kafka, LSM trees) are fast for writes
- Memory is ~1000x faster than SSD — this is why Redis handles 100K+ ops/sec while a database handles 10K-50K
- Each network hop adds 0.5ms within a datacenter — a request that chains through 10 microservices adds at least 5ms of pure network latency

</details>

<details>
<summary>4. How do you define APIs during a system design interview — why should you lock down the API contract early, how do you decide between REST endpoints vs RPC-style vs event-driven interfaces, and how does the API definition expose assumptions about read/write ratios and access patterns that shape the rest of the design?</summary>

**Why lock down the API early:** The API is the contract between the client and your system. Defining it early forces you to think about what the system actually does from the user's perspective before getting lost in infrastructure details. It also creates a shared reference point with the interviewer — they can see your assumptions about the product behavior and correct you before you build the wrong thing.

**How to define APIs in an interview:**

Keep it concise. List 3-5 core endpoints for the main use cases:

```
POST   /posts              — Create a post (write path)
GET    /feed?userId=X      — Get user's feed (read path, most traffic)
POST   /posts/:id/like     — Like a post (high-frequency write)
GET    /posts/:id          — Get single post (read path)
```

For each, note the key parameters, response shape, and any important behavior (pagination, sorting). Don't write full OpenAPI specs — that's wasted time.

**Choosing the interface style:**

**REST** — Default choice for client-facing APIs. Well-understood, cacheable (GET responses can be cached by CDN and browser), works with any client. Use when: the system is request-response, resources map naturally to URLs, cacheability matters.

**RPC (gRPC)** — Choose for internal service-to-service communication where low latency and strong typing matter. Protocol Buffers give you smaller payloads and code generation. Use when: you control both client and server, you need streaming, or you have polyglot services that benefit from shared schema.

**Event-driven** — Choose when the action doesn't need a synchronous response. A user uploads a photo — the upload API returns immediately, and image processing happens asynchronously via a message queue. Use when: work can be deferred, you need decoupling, or the consumer processes at a different rate than the producer.

**How API definition reveals access patterns:**

The API you define implicitly encodes your assumptions, and a good interviewer will probe them:

- If your feed endpoint is `GET /feed?userId=X`, you're assuming reads are user-scoped and frequent — this suggests caching per user and read replicas
- If you have a `POST /like` endpoint separate from `POST /posts`, you've identified that likes are a high-frequency, small write — maybe stored differently than posts
- If your API returns paginated results, you've assumed the full dataset is too large to return at once — this shapes your database query strategy
- If you don't have a real-time endpoint (WebSocket/SSE), you've assumed polling or eventual consistency is acceptable for feed updates

**The API also exposes read/write ratio:** Count the endpoints — if 4 out of 5 are GET requests, the system is read-heavy. This immediately tells you: invest in caching, consider read replicas, maybe use a CDN for static content.

</details>

<details>
<summary>5. What does "designing for the common case" mean in system design — how do you identify the dominant access patterns (read-heavy vs write-heavy, latency-sensitive vs throughput-sensitive), why is it dangerous to optimize for edge cases too early, and how do latency vs throughput tradeoffs influence which building blocks you choose?</summary>

**What it means:** Optimize your architecture for the workload that represents 90%+ of traffic, not the 1% edge case. A social media feed is read 100x more than it's written to — design the read path to be fast, even if it makes writes slightly more complex (e.g., fan-out on write, denormalized data).

**How to identify dominant access patterns:**

Ask these questions during requirement gathering:
- **Read vs write ratio:** "How often do users read vs create content?" Most systems are 10:1 to 1000:1 read-heavy. Chat systems and logging are closer to write-heavy.
- **Latency vs throughput:** "Does the user wait for a response, or is this background processing?" User-facing requests need low latency. Batch analytics needs high throughput.
- **Access skew:** "Is access uniform or does a small set of items get most of the traffic?" Hot items (trending posts, popular products) need caching; uniform access might not.

**Common patterns and their implications:**

| Pattern | Optimize for | Building blocks |
|---------|-------------|----------------|
| Read-heavy, latency-sensitive | Fast reads | Cache layers, read replicas, CDN, denormalization |
| Write-heavy, throughput-sensitive | Write throughput | Append-only logs, write-behind cache, sharding, async processing |
| Read-heavy, uniform access | Horizontal read scaling | Read replicas, partitioning |
| Read-heavy, skewed access | Hot spot mitigation | Caching with TTL, CDN for popular content |

**Why premature edge-case optimization is dangerous:**

- **Complexity cost:** Designing for edge cases (e.g., a viral post with 10M likes when normal posts get 100) adds complexity to every code path, not just the edge case. You end up with a system that's complex everywhere but optimal nowhere.
- **Wrong assumptions:** Until you have real traffic data, you're guessing which edges matter. Teams routinely over-engineer for scenarios that never happen.
- **Time cost in interviews:** If you spend 10 minutes designing a celebrity-tweet fan-out optimization before you have a working basic design, you've lost the interview.

**The right approach:** Design for the common case first. Then identify the specific edge cases that would break it, and add targeted solutions (e.g., "for accounts with > 1M followers, we switch from fan-out-on-write to fan-out-on-read").

**Latency vs throughput tradeoff in building block selection:**

- **Caching** increases throughput AND reduces latency — it's almost always a win for read-heavy systems, but adds consistency complexity
- **Message queues** trade latency for throughput — the user doesn't get an immediate result, but the system can process work at its own pace and handle bursts
- **Denormalization** trades write complexity for read latency — pre-computing data means reads are fast but writes must update multiple places
- **Replication** trades consistency for both latency and availability — reading from a nearby replica is fast but might be slightly stale

</details>

## Building Blocks — Traffic & Networking

<details>
<summary>6. Why do systems need load balancers, what is the difference between L4 (transport) and L7 (application) load balancing, how do you decide which layer to operate at, and how does a load balancer differ from a reverse proxy — when do the two roles overlap and when are they distinct?</summary>

**Why load balancers exist:** A single server has finite capacity. Load balancers distribute incoming traffic across multiple backend servers, enabling horizontal scaling, eliminating single points of failure, and allowing rolling deployments (drain one server, update it, add it back).

**L4 vs L7 load balancing:**

**L4 (transport layer)** operates on TCP/UDP. It sees IP addresses, ports, and connection metadata — but not the HTTP request content. It forwards the entire TCP connection to a backend server based on simple rules.

- Fast and efficient — no need to parse HTTP headers or payloads
- Lower resource overhead
- Cannot make routing decisions based on URL path, headers, or cookies
- Example: AWS Network Load Balancer (NLB)

**L7 (application layer)** operates on HTTP. It terminates the client connection, inspects the request (URL, headers, cookies, body), and makes intelligent routing decisions.

- Can route `/api/*` to API servers and `/static/*` to a CDN origin
- Can do sticky sessions based on cookies
- Can modify requests/responses (add headers, rewrite URLs)
- Can terminate TLS
- Higher resource overhead because it parses every request
- Example: AWS Application Load Balancer (ALB), NGINX, HAProxy

**When to choose which:**
- **L4** when you need raw throughput and don't need content-based routing — e.g., TCP load balancing for database connections, game servers, or gRPC streams
- **L7** for most web applications — you almost always want URL-based routing, TLS termination, and the ability to inspect headers

**Load balancer vs reverse proxy:**

They overlap significantly. A **reverse proxy** sits in front of backend servers, accepts client requests, and forwards them to the appropriate backend. It can also cache responses, compress content, terminate TLS, and provide a single entry point.

A **load balancer** specifically distributes traffic across multiple instances of the same service.

**When they overlap:** An NGINX instance acting as both reverse proxy (TLS termination, caching) and load balancer (round-robin across 5 app servers) — this is the common case. Most tools (NGINX, HAProxy, ALB) do both.

**When they're distinct:** A reverse proxy in front of a single backend server (no load balancing needed, just TLS termination and caching). Or a pure L4 load balancer that doesn't inspect or modify traffic (just distributes connections).

In system design interviews, you can usually treat them as the same box in your diagram and say "load balancer / reverse proxy" — then specify the routing logic if it matters.

</details>

<details>
<summary>7. How do load balancing algorithms work and when would you choose each one — compare round-robin, weighted round-robin, least connections, and consistent hashing, explain the tradeoffs of each (simplicity vs fairness vs stickiness), and why consistent hashing is critical for caching layers and sharded datastores?</summary>

**Round-robin:** Requests go to servers in sequential order: A, B, C, A, B, C... Simple, no state needed, works well when all servers have similar capacity and requests have similar cost.

- **Pro:** Dead simple, no overhead
- **Con:** Doesn't account for server capacity differences or request complexity. If one server is slower, it accumulates a backlog while still receiving the same share of requests.

**Weighted round-robin:** Same as round-robin but servers get different weights. A server with weight 3 gets 3x the traffic of a server with weight 1. Useful when servers have different hardware specs.

- **Pro:** Accounts for capacity differences
- **Con:** Weights are static — doesn't adapt to runtime conditions like one server being overloaded due to a noisy neighbor

**Least connections:** Routes each new request to the server currently handling the fewest active connections. Adapts dynamically to real load.

- **Pro:** Naturally handles heterogeneous request costs — slow requests keep connections open longer, so the server gets fewer new requests
- **Con:** Requires tracking connection counts across all servers, slightly more overhead. Doesn't account for CPU/memory — a server with few connections might still be CPU-bound.

**Consistent hashing:** Hashes the request (by key — e.g., user ID or cache key) and maps it to a server on a hash ring. The same key always goes to the same server.

- **Pro:** Stickiness without server-side session state. When a server is added or removed, only ~1/N of keys need to remap (minimal disruption).
- **Con:** Can create hot spots if the hash function or key distribution is skewed. More complex to implement correctly.

**When to choose each:**

| Algorithm | Best for |
|-----------|----------|
| Round-robin | Stateless services with uniform requests (most web API servers) |
| Weighted round-robin | Mixed hardware fleet, canary deployments (new version gets low weight) |
| Least connections | Requests with variable processing time (file uploads, video encoding) |
| Consistent hashing | Caching layers, sharded data stores, anything where the same key must hit the same server |

**Why consistent hashing is critical for caching and sharding:**

For a cache to be effective, the same key must consistently route to the same cache server. If you use round-robin, a request for user 123's profile might hit cache server A (miss), then B (miss), then C (miss) — the data is never cached effectively.

With consistent hashing, user 123 always maps to server B. Cache hits are maximized. When server B goes down, only B's keys redistribute — servers A and C keep their existing cached data. Without consistent hashing, adding or removing a server reshuffles almost all keys, causing a cache stampede.

The same logic applies to sharded databases — you need a deterministic mapping from key to shard, and you need that mapping to be stable when shards are added or removed. Consistent hashing with virtual nodes (covered in question 23) solves both problems.

</details>

<details>
<summary>8. Why are CDNs a fundamental building block in system design — how do push vs pull CDN models work, when would you choose each, and how do you handle cache invalidation at the CDN layer (TTL expiry, explicit purge, versioned URLs)? What goes wrong when CDN caching strategy is poorly designed?</summary>

**Why CDNs are fundamental:** They solve the physics problem — light in fiber takes ~100-200ms to cross continents. A CDN puts content on edge servers geographically close to users, reducing latency from 200ms to 10-30ms. For any system serving a global user base, a CDN is not optional for static assets and often worthwhile for cacheable API responses.

CDNs also absorb traffic. If 10M users request the same homepage image, the origin server serves it once to the CDN, and the CDN serves it 10M times. This reduces origin load by orders of magnitude.

**Push vs pull CDN:**

**Pull (origin-pull):** The CDN fetches content from your origin server on the first request (cache miss), then caches it at the edge. Subsequent requests are served from the edge until the TTL expires.

- Content is only cached when requested — no wasted storage for unpopular content
- First request for each piece of content hits the origin (cold start penalty)
- Simple to set up — just point the CDN at your origin
- Best for: most web applications, content with unpredictable popularity

**Push:** You explicitly upload content to the CDN. The CDN doesn't know about your origin — you manage what's on the edge.

- Full control over what's cached and when
- No cold start — content is on the edge before the first request
- More operational overhead — you must manage uploads, updates, and deletions
- Best for: large files you know will be popular (video streaming, software downloads), or when origin servers can't handle even occasional cache misses

**Cache invalidation strategies:**

**TTL expiry:** Set a time-to-live on each cached item. After TTL expires, the next request triggers a fresh fetch from the origin. Simple but coarse — you're choosing between stale data (long TTL) and more origin hits (short TTL).

**Explicit purge:** Send an API call to the CDN to invalidate specific URLs or patterns. Use when content changes are infrequent but must be reflected immediately (e.g., a news article correction). Most CDNs support purge APIs, but purge propagation takes seconds to minutes across all edge locations.

**Versioned URLs:** Append a version or hash to the URL: `/style.v3.css` or `/bundle.a1b2c3.js`. Set a very long TTL (1 year). When content changes, the URL changes, so the browser/CDN automatically fetches the new version. Old versions remain cached (harmless). This is the gold standard for static assets — it's effectively "immutable caching."

**What goes wrong with poor CDN strategy:**

- **Caching user-specific data:** If you accidentally cache API responses that contain user-specific data at the CDN, one user sees another user's data. This is a serious security bug. Always set `Cache-Control: private` or `no-store` for authenticated endpoints.
- **Short TTLs on everything:** Defeats the purpose of the CDN — most requests still hit the origin.
- **No cache-busting for deploys:** Deploying new JavaScript without changing the URL means users get stale code cached by the CDN, leading to bugs and broken UIs.
- **Not varying on the right headers:** If your API returns different content based on `Accept-Language` but the CDN doesn't vary on that header, users get the wrong language.

</details>

<details>
<summary>9. How do rate limiting algorithms (token bucket, sliding window, fixed window) work, what are the tradeoffs of each -- when would you apply rate limiting at the client vs the server, and what additional challenges does distributed rate limiting introduce when you have multiple API server instances?</summary>

**Token bucket:** A bucket holds up to N tokens. Each request consumes one token. Tokens are added at a fixed rate (e.g., 10/second). If the bucket is empty, the request is rejected (or queued). The bucket can accumulate tokens up to its maximum, allowing short bursts.

- **Pro:** Allows bursts up to bucket capacity — smooth for real users who send requests in clusters
- **Pro:** Simple to implement, memory-efficient (just two numbers: current tokens and last refill time)
- **Con:** Burst size is a tuning knob — too large allows abuse spikes, too small feels restrictive

**Fixed window:** Divide time into fixed windows (e.g., 1-minute intervals). Count requests in the current window. If count exceeds the limit, reject. Reset at the window boundary.

- **Pro:** Very simple — just a counter and a timestamp
- **Con:** Burst at window boundary — if the limit is 100/min, a user can send 100 requests at 0:59 and 100 more at 1:00, effectively getting 200 in 2 seconds

**Sliding window log:** Track the timestamp of every request. For each new request, count how many timestamps fall within the last N seconds. More accurate than fixed window.

- **Pro:** No boundary burst problem
- **Con:** Memory-heavy — stores every request timestamp

**Sliding window counter:** A hybrid — weight the current window count and previous window count based on how far into the current window you are. E.g., 70% through the current window: effective count = (current count) + (previous count * 0.3).

- **Pro:** Smooths the boundary burst problem with minimal memory (two counters per window)
- **Con:** Approximation, not exact — but good enough for nearly all use cases

**Client vs server rate limiting:**

**Server-side** is the enforcement point — clients can always be bypassed or faked, so server-side rate limiting is non-negotiable for protection.

**Client-side** rate limiting is a courtesy mechanism. It prevents well-behaved clients from hitting the server's limits, provides better UX (show "slow down" immediately instead of getting 429s), and reduces unnecessary network calls. Common in SDKs and API clients with built-in backoff.

**Distributed rate limiting challenges:**

With multiple API server instances behind a load balancer, each server only sees a fraction of a user's requests. If the limit is 100/min and you have 5 servers, each server sees ~20 requests — none individually exceeds 100, so the limit is never enforced.

**Solutions:**

1. **Centralized counter (Redis):** All servers increment a shared counter in Redis for each rate limit key (e.g., `rate:user:123`). Redis's `INCR` + `EXPIRE` or `EVALSHA` (Lua script for atomic check-and-increment) makes this fast and atomic. This is the most common approach.

2. **Sticky sessions:** Route the same user to the same server (consistent hashing on user ID). Each server enforces locally. Simpler but less flexible — if that server goes down, limits reset.

3. **Approximate local limiting:** Each server enforces limits locally with the total limit divided by server count (100/5 = 20 per server). Simple but inaccurate — uneven load distribution means some users get more or fewer requests than intended.

The Redis approach is the industry standard for distributed rate limiting. The extra ~0.5ms network hop to Redis per request is negligible compared to the actual request processing time.

</details>

<details>
<summary>10. How do you choose the right communication protocol for a given system design scenario — compare HTTP/REST, gRPC, WebSockets, SSE, and UDP in terms of latency, connection overhead, bidirectionality, and browser support, and explain the specific scenarios where each one is the right choice and where it would be a poor fit?</summary>

| Protocol | Latency | Connection overhead | Bidirectional | Browser support | Payload |
|----------|---------|-------------------|---------------|-----------------|---------|
| HTTP/REST | Medium (new conn per request, or keep-alive) | Low per request (reuses connections with HTTP/2) | No (request-response) | Excellent | Text (JSON), flexible |
| gRPC | Low (HTTP/2 multiplexing, binary) | Low (persistent connection) | Yes (bidirectional streaming) | Limited (needs grpc-web proxy) | Binary (Protocol Buffers) |
| WebSockets | Very low (persistent connection) | High initial handshake, then low | Yes (full duplex) | Excellent | Any (text or binary) |
| SSE | Low (persistent connection) | Medium (one HTTP connection held open) | No (server-to-client only) | Good (native EventSource API) | Text |
| UDP | Lowest (no connection setup) | None | Yes (but manual) | No (except WebRTC) | Any |

**When to choose each:**

**HTTP/REST** — The default for client-facing APIs and most service-to-service communication. Choose when: request-response semantics fit, you want cacheability (CDN, browser cache), you need broad client compatibility, and the interaction is "client asks, server answers."

- Poor fit: real-time bidirectional communication, high-frequency polling scenarios, internal services where payload size matters at scale

**gRPC** — Choose for internal service-to-service communication, especially in microservice architectures. Strong typing from Protocol Buffers, efficient binary serialization, built-in streaming. Code generation means your client/server interfaces stay in sync.

- Poor fit: browser-facing APIs (requires a proxy), simple CRUD APIs where REST is sufficient, teams without the infrastructure to manage proto files

**WebSockets** — Choose when you need real-time bidirectional communication: chat applications, collaborative editing, live gaming, real-time dashboards where both client and server push data.

- Poor fit: simple request-response APIs (overkill), scenarios where server-only push suffices (use SSE instead — simpler), stateless architectures (WebSockets are inherently stateful and harder to load balance)

**SSE (Server-Sent Events)** — Choose when the server needs to push updates to the client but the client doesn't need to send data back over the same channel: live feeds, notification streams, real-time score updates. Simpler than WebSockets — it's just an HTTP response that never ends.

- Poor fit: bidirectional communication, binary data, environments where long-lived HTTP connections are problematic (some proxies/load balancers)

**UDP** — Choose when latency matters more than reliability: real-time video/audio (WebRTC), online gaming (position updates), DNS lookups. No connection handshake, no ordering guarantees, no retransmission — you accept packet loss in exchange for speed.

- Poor fit: anything where data loss is unacceptable (financial transactions, API calls), browser applications (no direct UDP access)

**Decision framework for interviews:** Start with REST. Switch to gRPC if the question involves internal microservice communication with performance requirements. Add WebSockets or SSE only if the requirements explicitly call for real-time updates. Mention UDP only for media streaming or gaming scenarios.

</details>

## Building Blocks — Data & Storage

<details>
<summary>11. What are the caching strategies (cache-aside, write-through, write-behind, refresh-ahead), how does each one work, what consistency tradeoffs does each introduce, and how do you decide which strategy fits a given access pattern — why is cache-aside the most common default, and when would write-through or write-behind be a better choice?</summary>

**Cache-aside (lazy loading):** The application checks the cache first. On a miss, it reads from the database, writes the result to the cache, and returns it. On writes, the application writes to the database and invalidates (or updates) the cache entry.

- **Consistency:** Data can be stale if another process updates the database without invalidating the cache. Stale window = TTL duration.
- **Failure mode:** Cache goes down — application still works, just slower (falls through to database).
- **Why it's the default:** Simple, the application controls everything, and the cache is just an optimization layer. You can add it incrementally to specific queries without changing your write path.

**Write-through:** Every write goes to the cache AND the database synchronously. The cache is always up-to-date for data that has been written.

- **Consistency:** Strong — the cache is never stale for written data (but data loaded from DB before the write-through policy started may still be stale).
- **Tradeoff:** Write latency increases because every write hits two stores. Data that's written but never read wastes cache space.
- **Best for:** Systems where you read data immediately after writing it (e.g., user profile updates followed by profile page load).

**Write-behind (write-back):** Writes go to the cache immediately, and the cache asynchronously flushes to the database later (batched or after a delay).

- **Consistency:** Weak — the database is behind the cache. If the cache crashes before flushing, data is lost.
- **Tradeoff:** Extremely fast writes (just a memory write), but you risk data loss and increased complexity.
- **Best for:** Write-heavy workloads where temporary data loss is acceptable (e.g., analytics counters, gaming leaderboards, view counts).

**Refresh-ahead:** The cache proactively refreshes entries before they expire, based on predicted access patterns. If an entry is accessed frequently and is approaching its TTL, the cache fetches a fresh copy in the background.

- **Consistency:** Reduces stale window compared to cache-aside (entries are refreshed before they expire).
- **Tradeoff:** Wastes resources if predictions are wrong (refreshing data nobody requests). Complex to implement correctly.
- **Best for:** Data that's read very frequently with predictable access patterns (e.g., a product catalog, configuration data).

**Decision guide:**

| Scenario | Strategy |
|----------|----------|
| General-purpose caching, first time adding cache | Cache-aside |
| Read-after-write consistency matters | Write-through |
| Very high write throughput, some data loss acceptable | Write-behind |
| Hot data with predictable access, low-latency reads critical | Refresh-ahead |

In practice, most systems use cache-aside as the starting point. Write-through is added for specific paths where read-after-write consistency matters. Write-behind is rare outside of specialized high-write systems.

</details>

<details>
<summary>12. Where can caching layers exist in a system (client, CDN, application, database) and why does the position of the cache matter — how do you reason about which layer to cache at, what happens when you have multiple caching layers that interact, and what are the common pitfalls (stale data, cache stampede, cold start)?</summary>

**Caching layers from client to database:**

**1. Client/browser cache:** `Cache-Control` headers tell the browser to store responses locally. Fastest possible cache — no network call at all. Use for static assets (JS, CSS, images) and immutable API responses. Zero cost to your infrastructure.

**2. CDN edge cache:** Caches content at geographically distributed edge servers. Reduces latency for global users and absorbs traffic from your origin. Use for static assets, public API responses, and any cacheable content. (Covered in question 8.)

**3. Application-level cache (Redis/Memcached):** Sits between your application servers and the database. Stores computed results, database query results, or serialized objects. This is where most system design caching decisions happen. Use for hot data that's expensive to compute or frequently read.

**4. Database query cache / buffer pool:** The database itself caches frequently accessed data pages in memory (PostgreSQL shared buffers, MySQL buffer pool). This is automatic but limited — it helps with repeated queries on the same data but doesn't reduce connection overhead or query parsing cost.

**Why position matters:** Each layer closer to the client is faster but harder to invalidate. Client cache is instant but you can't force a user's browser to clear it. CDN cache is fast but purge propagation takes time. Application cache (Redis) is the sweet spot for most dynamic data — you control it fully and it's still sub-millisecond.

**Multi-layer cache interaction:**

A typical request might check: browser cache -> CDN -> Redis -> database. Each layer reduces the load on the layer behind it. But multiple layers create consistency challenges:

- If you update the database and invalidate Redis but not the CDN, users see stale CDN data
- If you invalidate the CDN but not browser caches, users see stale data until their browser cache expires
- Debugging "why is the user seeing old data" becomes harder — you must check each layer

**Rule of thumb:** Use versioned URLs for static assets (infinite client/CDN TTL, change URL on update). Use short TTLs + invalidation for dynamic data in Redis. Don't cache user-specific data at the CDN layer.

**Common pitfalls:**

**Stale data:** The fundamental caching problem. Mitigate with: short TTLs for volatile data, explicit invalidation on writes, versioned URLs for static assets. Accept that some staleness is the price of caching — the question is whether the staleness window is acceptable for your use case.

**Cache stampede (thundering herd):** A popular cache entry expires, and hundreds of concurrent requests all miss the cache simultaneously, flooding the database. Solutions:
- **Locking:** Only one request fetches from DB; others wait for the cache to be populated
- **Stale-while-revalidate:** Serve the stale value while one request refreshes it in the background
- **Jittered TTLs:** Add randomness to TTLs so entries don't expire at the same time

**Cold start:** After a deployment, cache restart, or new region spin-up, the cache is empty and all requests hit the database. Solutions:
- **Cache warming:** Pre-populate the cache with known hot data before routing traffic
- **Gradual traffic shift:** Slowly ramp traffic to the new instance while the cache fills
- **Capacity planning:** Ensure your database can handle the cold-start load temporarily

</details>

<details>
<summary>13. Why are message queues a fundamental building block in system design — explain how they enable decoupling, load leveling, and retry, how ordering guarantees and backpressure work, how task queues differ from message queues (dedicated workers processing jobs vs general pub/sub), and when introducing a queue adds unnecessary complexity vs when it's essential for the architecture?</summary>

**Why message queues are fundamental:**

They solve three problems that appear in almost every system at scale:

**1. Decoupling:** The producer doesn't need to know who consumes the message or whether the consumer is even running. An order service publishes "order created" and doesn't care whether the notification service, analytics service, and inventory service each pick it up. This means services can be deployed, scaled, and failed independently.

**2. Load leveling:** If your API receives 10,000 requests/sec in a spike but your processing backend can only handle 2,000/sec, a queue absorbs the burst. The backend processes at its own pace. Without a queue, the spike overwhelms the backend and requests fail.

**3. Retry and durability:** If a consumer fails to process a message, the queue holds it for retry. Messages aren't lost when a server crashes. This is essential for operations that must eventually succeed (payment processing, email delivery).

**Ordering guarantees:**

- **No ordering (most queues by default):** Messages may be delivered out of order. Fine for independent tasks.
- **Partition-level ordering (Kafka):** Messages within the same partition are ordered. You send all messages for user 123 to the same partition, so their events are processed in sequence. But different users' messages are processed in parallel.
- **Strict global ordering:** All messages across all consumers are in order. Very expensive — essentially serializes processing to a single consumer. Rarely needed.

**Backpressure:** When consumers can't keep up, the queue grows. Backpressure mechanisms prevent unbounded growth:
- **Bounded queue:** Rejects or blocks new messages when full — the producer slows down
- **Consumer scaling:** Auto-scale consumers based on queue depth
- **Dead letter queue (DLQ):** Messages that fail after N retries go to a DLQ for manual inspection instead of blocking the queue

**Message queues vs task queues:**

**Message queue (pub/sub):** General-purpose message passing. Multiple consumers can subscribe to the same topic. Examples: Kafka, RabbitMQ, SNS/SQS. Used for event streaming, inter-service communication.

**Task queue:** A specialized pattern where jobs are placed in a queue and dedicated workers pick them up one at a time. Each job is processed by exactly one worker. Examples: Bull (Redis-backed, Node.js), Celery (Python), SQS with a single consumer group. Used for background jobs: image processing, report generation, email sending.

The distinction is about consumption semantics: pub/sub = one message, many consumers. Task queue = one job, one worker.

**When a queue is essential vs unnecessary complexity:**

**Essential:**
- Async processing that the user shouldn't wait for (image thumbnailing, email sending)
- Systems where producer and consumer operate at different rates
- Cross-service communication where coupling is unacceptable
- Operations that need reliable retry (payment processing)

**Unnecessary complexity:**
- Simple request-response where the client needs the result immediately and latency matters
- Two services with matching throughput and tight coupling that's actually desirable (e.g., an API gateway calling an auth service — you need the response before proceeding)
- Low-traffic systems where a synchronous HTTP call is simpler and fast enough
- When the team doesn't have the operational maturity to monitor queue depth, DLQs, and consumer lag

</details>

<details>
<summary>14. Why is idempotency a critical design requirement in distributed systems -- explain what idempotency keys are, how they enable safe retries, the difference between at-least-once and exactly-once delivery semantics, and how you would design an idempotent API endpoint that handles duplicate requests gracefully?</summary>

**Why idempotency is critical:** In distributed systems, retries are inevitable. Networks drop packets, load balancers time out, clients retry on error. If the server received and processed the first request but the response was lost, the client will retry — and without idempotency, the operation happens twice. Charging a customer $100 twice because of a network glitch is unacceptable.

An operation is **idempotent** if executing it multiple times produces the same result as executing it once. `GET /user/123` is naturally idempotent. `POST /payments` is not — unless you design it to be.

**Idempotency keys:** A unique identifier (typically a UUID) that the client generates and sends with the request. The server uses this key to detect duplicates.

```typescript
// Client sends:
// POST /payments
// Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
// Body: { amount: 100, currency: "USD" }

// Server implementation:
async function processPayment(req: Request) {
  const idempotencyKey = req.headers['idempotency-key'];

  // Check if we've already processed this request
  const existing = await db.query(
    'SELECT response FROM idempotency_keys WHERE key = $1',
    [idempotencyKey]
  );

  if (existing) {
    return existing.response; // Return the stored response
  }

  // Atomically claim the idempotency key (prevents race between duplicate requests)
  const claimed = await db.query(
    `INSERT INTO idempotency_keys (key, status, created_at)
     VALUES ($1, 'processing', NOW())
     ON CONFLICT (key) DO NOTHING
     RETURNING key`,
    [idempotencyKey]
  );

  if (!claimed) {
    // Another request claimed it first — wait briefly and return its result
    return retryUntilComplete(idempotencyKey);
  }

  // Process the payment (only one request reaches here per key)
  const result = await chargeCustomer(req.body);

  // Store the response so retries can return the same result
  await db.query(
    `UPDATE idempotency_keys SET response = $2, status = 'complete' WHERE key = $1`,
    [idempotencyKey, JSON.stringify(result)]
  );

  return result;
}
```

The idempotency key record should have a TTL (e.g., 24-48 hours) — you don't need to store it forever.

**At-least-once vs exactly-once delivery:**

**At-least-once:** The system guarantees every message is delivered, but some may be delivered more than once. The consumer must be idempotent to handle duplicates. This is what most message queues provide (SQS, RabbitMQ, Kafka consumer groups).

**Exactly-once:** Each message is processed exactly one time. True exactly-once is extremely difficult in distributed systems because it requires coordination between the message broker and the processing system. Kafka achieves it within its ecosystem using transactions (producer writes to Kafka and commits the consumer offset atomically), but across system boundaries (e.g., Kafka to database), you need idempotent consumers to achieve effective exactly-once.

**In practice:** Design for at-least-once delivery + idempotent consumers. This gives you "effectively exactly-once" processing without the complexity of distributed transactions.

**Designing an idempotent endpoint — key considerations:**

1. **The idempotency key must be client-generated** — if the server generates it, the client doesn't have it for retries
2. **The check and the processing must be atomic** — use a database transaction or atomic upsert to prevent a race where two identical requests both pass the check
3. **Store the response, not just a flag** — the client needs the same response on retry, not just "already processed"
4. **Scope keys appropriately** — usually per-user or per-API-key, not globally
5. **Handle in-flight duplicates** — if request A is still processing when duplicate request B arrives, B should wait or return 409, not start a second processing

</details>

## Consistency, Replication & Data Models

<details>
<summary>15. What are the consistency models you need to reason about in system design (strong, eventual, causal, read-your-writes) — explain what guarantees each model provides, why stronger consistency costs more in latency and availability, and how do you choose the right consistency model for different parts of the same system?</summary>

**Strong consistency:** Every read returns the most recently written value. After a write completes, all subsequent reads — from any node — see that write. This is what a single-machine database gives you.

- **Cost:** Every write must be acknowledged by a majority of nodes before returning. Reads must go to the leader or verify freshness. This adds latency (wait for replication) and reduces availability (if the leader is down, writes are blocked).
- **Use when:** Financial transactions, inventory counts (selling more than you have), anything where stale data causes real-world harm.

**Eventual consistency:** After a write, replicas will eventually converge to the same value, but there's no guarantee when. Reads may return stale data.

- **Cost:** Almost zero — writes are fast (no coordination), reads can go to any replica. But clients may see inconsistent views.
- **Use when:** Social media likes/counts (showing 998 likes instead of 1000 is fine), DNS propagation, analytics data.

**Causal consistency:** If operation A causally precedes operation B (B depends on A's result), then everyone sees A before B. Concurrent operations (no causal relationship) can be seen in any order.

- **Cost:** Between strong and eventual. Requires tracking causal dependencies (vector clocks or similar), but doesn't require global ordering.
- **Use when:** Comment threads (reply must appear after the parent), collaborative editing.

**Read-your-writes (session consistency):** A user always sees their own writes. Other users may see stale data, but the writing user gets immediate consistency for their own actions.

- **Cost:** Low — just route the user's reads to the same replica that handled their write (or to the leader) for a short window after writing.
- **Use when:** User profile updates ("I changed my name but the page still shows the old one" is a terrible UX), form submissions.

**Why stronger = more expensive:** Strong consistency requires coordination — nodes must communicate and agree before responding. This communication takes time (latency) and requires nodes to be reachable (availability). The CAP theorem formalizes this: during a network partition, you must choose between consistency and availability.

**Choosing per data path within one system:**

A well-designed system uses different consistency models for different data:

| Data | Model | Why |
|------|-------|-----|
| Account balance | Strong | Overdrafts are real-world harm |
| User profile (for the user) | Read-your-writes | User expects to see their changes |
| User profile (for others) | Eventual | A few seconds of staleness is fine |
| Like/view counts | Eventual | Approximate counts are acceptable |
| Inventory for purchase | Strong | Overselling is costly |
| Product catalog browsing | Eventual | Stale product descriptions are harmless |
| Chat message ordering | Causal | Messages must appear in conversational order |

This per-path thinking is a strong signal of senior design ability in interviews. Don't apply one model to the entire system.

</details>

<details>
<summary>16. What does the CAP theorem actually say and why is it frequently misunderstood — explain the real constraint (you must choose between consistency and availability during a network partition), how PACELC extends CAP to cover the non-partition case (latency vs consistency tradeoff), and how these theorems guide real design decisions rather than being academic trivia?</summary>

**What CAP actually says:** In a distributed system experiencing a network partition (P), you must choose between Consistency (C) and Availability (A). You cannot have both simultaneously during a partition.

- **Consistency (C):** Every read returns the most recent write or an error.
- **Availability (A):** Every request receives a non-error response (but the data may be stale).
- **Partition tolerance (P):** The system continues operating despite network failures between nodes.

**The common misunderstanding:** People say "pick two out of three" — as if you could choose CA, CP, or AP. But partitions are not optional. Networks fail. You don't "choose" partition tolerance — you have to tolerate partitions because they happen. The real choice is: when a partition occurs, do you sacrifice consistency (return stale data but stay available) or availability (reject requests until the partition heals)?

**During normal operation, you get both C and A.** The tradeoff only materializes during a partition.

**CP systems (choose consistency):** During a partition, refuse to serve requests on the side that can't reach the leader. Example: a single-leader PostgreSQL with synchronous replication — if the replica can't reach the leader, it stops serving reads rather than return potentially stale data.

**AP systems (choose availability):** During a partition, serve requests from whatever node is reachable, accepting that responses might be stale. Example: Cassandra, DynamoDB — every node can accept reads and writes, even during a partition. Conflicts are resolved later.

**PACELC extension:** CAP only covers the partition case. PACELC says: during a **P**artition, choose **A** or **C**. **E**lse (no partition), choose **L**atency or **C**onsistency.

This is the more useful framing because most of the time there's no partition, and you're still making tradeoffs:
- **PA/EL (e.g., Cassandra):** During partition, choose availability. Without partition, choose low latency (reads from any replica, may be stale).
- **PC/EC (e.g., single-leader PostgreSQL with sync replication):** During partition, choose consistency. Without partition, still choose consistency (reads go to leader, higher latency).
- **PA/EC (e.g., MongoDB with majority reads):** During partition, choose availability. Without partition, choose consistency (read from primary or majority).

**How this guides real design decisions:**

1. **Database selection:** If your system absolutely needs strong consistency (financial ledger), choose a CP database or a single-leader setup. If you need global availability and can tolerate staleness (social media feed), choose an AP system.

2. **Multi-region architecture:** Going multi-region forces the CAP choice. Synchronous cross-region replication (CP) adds 100ms+ latency per write. Async replication (AP) is fast but risks data loss and stale reads.

3. **Per-feature decisions:** Your payment service might be CP while your feed service is AP. This is the practical application — different parts of your system make different tradeoffs based on the cost of inconsistency vs unavailability.

</details>

<details>
<summary>17. How do the three database replication models (single-leader, multi-leader, leaderless) work and what are the tradeoffs of each in terms of consistency, write availability, and latency -- when would you choose each one in a system design, and what failure modes are unique to each model?</summary>

**Single-leader (primary-secondary):** All writes go to one leader node. The leader replicates changes to followers (replicas). Reads can go to the leader (strong consistency) or followers (eventual consistency, lower latency).

- **Consistency:** Strong if reading from leader. Eventual if reading from followers (replication lag).
- **Write availability:** Single point of failure — if the leader goes down, writes are blocked until failover completes (seconds to minutes). Automated failover (promote a follower) helps but risks split-brain if the old leader comes back.
- **Write latency:** Low for single-region (write to one node). High for multi-region if leader is in a distant region.
- **When to choose:** Most applications. This is the default — PostgreSQL, MySQL, MongoDB (default config). It's simple, well-understood, and sufficient for the vast majority of systems.
- **Unique failure mode:** Failover complications — the new leader may be behind the old leader, causing data loss. Split-brain if two nodes both believe they're the leader.

**Multi-leader:** Multiple nodes accept writes. Each leader replicates to the others. Used for multi-region deployments where you want local write latency.

- **Consistency:** Eventual. Write conflicts are possible — two leaders accept conflicting writes for the same record. Conflict resolution is required (last-write-wins, application-level merge, CRDTs).
- **Write availability:** High — each region has a local leader. A regional failure doesn't block writes in other regions.
- **Write latency:** Low — writes go to the local leader.
- **When to choose:** Multi-region deployments where write latency matters (collaborative editing, globally distributed applications). Rare — the conflict resolution complexity is significant.
- **Unique failure mode:** Write conflicts. Two users edit the same document in different regions simultaneously. How do you merge? Last-write-wins loses data. Custom merge logic is application-specific and error-prone.

**Leaderless:** Any node can accept reads and writes. A write is sent to multiple nodes simultaneously, and a read queries multiple nodes. Uses quorum — a write succeeds if W nodes acknowledge, a read succeeds if R nodes respond. As long as W + R > N (total nodes), you're guaranteed to read at least one node with the latest write.

- **Consistency:** Tunable via quorum parameters. W + R > N gives strong read consistency. Lower quorum gives eventual consistency with higher availability.
- **Write availability:** Very high — no single leader to fail. As long as W nodes are reachable, writes succeed.
- **Write latency:** Moderate — must wait for W acknowledgments.
- **When to choose:** Systems needing very high write availability and tolerance to node failures. Cassandra and DynamoDB use this model. Good for time-series data, IoT ingestion, and workloads where availability matters more than consistency.
- **Unique failure modes:**
  - **Read repair / anti-entropy:** Stale nodes must eventually get updated. If anti-entropy processes fall behind, reads with low R values may return stale data.
  - **Sloppy quorum:** During a partition, writes may go to "wrong" nodes (hinted handoff). The data eventually migrates back, but there's a window of inconsistency.

**Summary table:**

| | Single-leader | Multi-leader | Leaderless |
|--|---|---|---|
| Write destination | One node | Multiple leaders | Any node |
| Conflict handling | None (one writer) | Required (merge/LWW) | Required (quorum/vector clocks) |
| Write availability | Low (leader failure blocks writes) | High | Very high |
| Consistency | Strong (from leader) | Eventual | Tunable |
| Complexity | Low | High | Medium |
| Typical use | Most applications | Multi-region writes | High-availability, write-heavy |

</details>

<details>
<summary>18. How do you decide between SQL and NoSQL in a system design — what are the real selection criteria (data model, query patterns, consistency requirements, scale characteristics, team expertise) beyond the surface-level "SQL for relational, NoSQL for everything else," and what are common mistakes teams make when choosing, and what are the consequences?</summary>

**The real selection criteria:**

**1. Data model and relationships:**
- **SQL** excels when data has clear relationships and you need JOINs: users have orders, orders have items, items belong to categories. The relational model enforces structure and integrity (foreign keys, constraints).
- **NoSQL** excels when data is self-contained and hierarchical: a product catalog entry with embedded reviews, a user session object, event logs. If you rarely JOIN across entities, forcing them into a relational model adds complexity for no benefit.

**2. Query patterns:**
- **SQL** shines for ad-hoc queries. You don't know all your access patterns upfront? SQL lets you query any combination of columns with WHERE, GROUP BY, JOIN. Analytics, reporting, and exploratory queries are much easier.
- **NoSQL** (key-value, document) is optimized for known access patterns. DynamoDB requires you to define your access patterns upfront and design your table/index structure around them. Unknown future queries may be impossible or expensive.

**3. Consistency requirements:**
- **SQL databases** (PostgreSQL, MySQL) provide ACID transactions by default. Multi-row, multi-table transactions are straightforward.
- **NoSQL databases** vary widely. DynamoDB gives you single-item transactions easily but multi-item transactions are limited. Cassandra has no multi-row transactions. MongoDB added multi-document transactions but they're expensive.
- If your domain requires "either both of these things happen or neither does" across multiple records, SQL is the safer bet.

**4. Scale characteristics:**
- **SQL** scales vertically well and horizontally for reads (read replicas). Write scaling requires sharding, which is complex and loses some relational benefits (cross-shard JOINs).
- **NoSQL** (Cassandra, DynamoDB) was built for horizontal write scaling. If you need to write millions of records/sec across many nodes, this is the native model.
- But most systems don't need that scale. A single PostgreSQL instance handles a lot more than people think.

**5. Team expertise:**
- This is underrated. A team proficient in PostgreSQL will ship faster and more reliably with PostgreSQL than with a "better-fit" technology they don't know. Operational maturity matters — running Cassandra well requires specific expertise.

**Common mistakes:**

**Choosing NoSQL because "it scales better":** Most systems never hit the scale where SQL can't handle it. You lose ACID, JOINs, and ad-hoc queries for a scaling benefit you never use. PostgreSQL can handle millions of rows with proper indexing and read replicas.

**Choosing SQL when the data model is document-shaped:** Forcing deeply nested, variable-structure data into normalized tables creates dozens of JOINs and complex migrations. A document database stores the natural shape of the data.

**Choosing based on hype:** Picking MongoDB because it's popular, then needing multi-document transactions and complex aggregation pipelines that would be trivial SQL queries.

**Not considering operational cost:** Running a Cassandra cluster requires monitoring compaction, tuning consistency levels, managing tombstones, and understanding gossip protocol failures. That's a different skill set than running PostgreSQL with well-known backup/restore patterns.

**The pragmatic default:** Start with PostgreSQL unless you have a specific, measurable reason not to. It handles JSON documents (JSONB), full-text search, time-series data (with TimescaleDB), and horizontal reads. Switch to NoSQL when you actually hit its limits — not when you imagine you might.

</details>

<details>
<summary>19. How do distributed coordination primitives (leader election, distributed locks, consensus) work at a high level — why are protocols like Raft and Paxos necessary, what problems do tools like ZooKeeper and etcd solve, and when would you reach for a distributed lock vs an alternative approach in a system design?</summary>

**Why coordination primitives exist:** In a distributed system, multiple nodes need to agree on things: who is the leader, whether a particular resource is locked, what the current configuration is. Without coordination, you get split-brain (two leaders), double-spending (two nodes process the same payment), or inconsistent state.

**Consensus protocols (Raft, Paxos):**

The fundamental problem: how do N nodes agree on a single value when some nodes may crash, messages may be lost, and there's no shared memory?

**Paxos** was the first proven solution (Lamport, 1989). It works through a propose-accept-learn protocol where a majority of nodes must agree. Correct but notoriously difficult to understand and implement.

**Raft** (2013) solves the same problem with a more understandable design. It decomposes consensus into leader election, log replication, and safety:
1. Nodes elect a leader through voting (a node needs a majority of votes)
2. The leader accepts all writes and replicates them to followers
3. A write is committed when a majority of nodes have it
4. If the leader fails, followers detect the timeout and elect a new leader

Both require a majority (quorum) of nodes to be available. A 5-node cluster tolerates 2 failures. A 3-node cluster tolerates 1.

**What ZooKeeper and etcd solve:**

You don't implement Raft yourself. ZooKeeper and etcd are coordination services that implement consensus internally and expose higher-level primitives:

- **Leader election:** Services register as candidates. One wins. Others watch and take over if the leader dies.
- **Distributed locks:** Acquire a lock on a path/key. Only one holder at a time. Automatic release if the holder crashes (via session/lease expiry).
- **Configuration management:** Store and watch config values. All nodes get notified when config changes.
- **Service discovery:** Services register themselves. Clients query for available instances.

**etcd** is used by Kubernetes for all cluster state. **ZooKeeper** is used by Kafka (older versions), Hadoop, and HBase.

**When to use distributed locks vs alternatives:**

**Reach for a distributed lock when:**
- Exactly-one execution is required (only one node should process a specific job at a time)
- Access to an external resource must be serialized (e.g., writing to a shared file)

**Consider alternatives first:**
- **Idempotency:** If the operation is idempotent, let multiple nodes process it — duplicates are harmless. Cheaper and more available than locking.
- **Partitioning:** Assign each record to a specific node (via consistent hashing). No lock needed — the assignment determines who processes it.
- **Optimistic concurrency:** Use version numbers / ETags. Read the version, make changes, write with "only if version hasn't changed." No lock, no coordination — just retry on conflict.
- **Queue with single consumer:** A message queue with a single consumer per partition achieves exclusive processing without distributed locks.

**Distributed locks are dangerous because:**
- The lock holder can crash while holding the lock — you need TTL-based automatic expiry
- TTL too short: the lock expires while the holder is still working (GC pause, slow network), and another node acquires it — now two nodes hold the "lock"
- This is the fencing token pattern: each lock acquisition gets a monotonically increasing token. The resource rejects operations with old tokens.
- Locks reduce throughput (serialization) and availability (lock service is a dependency)

Use them as a last resort, not a first tool.

</details>

## Scaling & Partitioning

<details>
<summary>20. How do you decide between vertical and horizontal scaling in a system design -- what are the limits of each approach, why is vertical scaling often the right first step, and what signals tell you it is time to move to horizontal scaling?</summary>

**Vertical scaling (scale up):** Add more CPU, RAM, or disk to a single machine. A single powerful server handles all traffic.

**Limits:**
- Hardware ceiling — there's a maximum machine size available from cloud providers (e.g., AWS `u-24tb1.metal` with 24 TB RAM exists, but it's expensive and you'll hit a ceiling eventually)
- Single point of failure — one machine means one failure domain
- Cost curve is non-linear — doubling specs often more than doubles the cost
- No geographic distribution — one machine is in one region

**Horizontal scaling (scale out):** Add more machines and distribute the workload across them.

**Limits:**
- Application complexity — your code must handle distributed state, network partitions, and coordination
- Data partitioning challenges — sharding a database is fundamentally harder than just making the database bigger
- Not all workloads parallelize — a single query that needs to scan all data can't benefit from more machines unless you partition the data
- Operational overhead — more machines means more to monitor, deploy, and manage

**Why vertical scaling is often the right first step:**

1. **Simplicity:** No distributed systems problems. No sharding, no replication lag, no consensus protocols. A single PostgreSQL instance on a large machine is dramatically simpler than a sharded cluster.

2. **Cost-effective at moderate scale:** A modern single machine (64 cores, 256 GB RAM, NVMe SSDs) can handle a surprising amount of traffic — often thousands of QPS for a well-indexed database. Many startups run for years on a single database instance.

3. **Faster iteration:** When your architecture is simple, you ship features faster. Premature horizontal scaling adds weeks of engineering effort to every feature.

4. **Moore's law still helps:** Hardware keeps getting faster. The largest cloud instances today are bigger than what was available 5 years ago.

**Signals it's time to move to horizontal scaling:**

- **CPU/memory consistently at 70-80%** on the largest instance available, and you've already optimized queries and caching
- **Single point of failure is unacceptable** — you need high availability, which requires at least 2 machines regardless of load
- **Geographic latency** — users in Asia are getting 200ms responses from your US server, and you need multi-region
- **Write throughput exceeds single-node capacity** — read replicas help with reads, but if writes saturate the single leader, you need sharding
- **You've done the simple things first** — indexing, query optimization, caching, connection pooling. If you haven't optimized your existing setup, don't scale horizontally to hide the inefficiency

**In system design interviews:** Start by estimating whether a single machine can handle the load. If yes, say so and keep the design simple. Interviewers want to see you can scale, but they also want to see you don't over-engineer.

</details>

<details>
<summary>21. What are the database-specific scaling strategies (read replicas, federation, denormalization) -- explain when each approach is appropriate, why read replicas only help read-heavy workloads, and how federation and denormalization trade normalization for scale?</summary>

**Read replicas:** Create copies of the primary database that serve read traffic. Writes still go to the primary, which replicates changes to replicas asynchronously.

- **When appropriate:** Read-heavy workloads (10:1 read/write or higher). Most web applications — users browse products, read feeds, view profiles far more than they create content.
- **Why they only help reads:** Every replica receives the full write stream from the primary. Adding replicas doesn't reduce write load on the primary — it actually increases it slightly (each replica is another replication target). If your bottleneck is writes, replicas don't help.
- **Tradeoff:** Replication lag means replicas may serve stale data (eventual consistency). For most reads this is fine, but for read-after-write consistency you need to route to the primary (as discussed in question 15).
- **Practical limit:** 5-10 replicas is typical. Beyond that, the primary's replication overhead becomes significant.

**Federation (functional partitioning):** Split the database by function — users in one database, orders in another, products in a third. Each database handles a subset of the domain.

- **When appropriate:** When different domains have very different access patterns or scale requirements. The users table might be read-heavy while the orders table is write-heavy — separate databases let you optimize each independently.
- **What it trades:** You lose cross-domain JOINs. Querying "all orders by user X with product details" now requires your application to query multiple databases and join in code. Foreign key constraints across databases are impossible.
- **Tradeoff:** Simpler than sharding (each database is still a single node), but only works when the domain naturally splits into independent pieces. If most queries need data from multiple domains, federation creates more problems than it solves.

**Denormalization:** Duplicate data to avoid expensive JOINs at read time. Store the user's name directly on the order record instead of joining to the users table every read.

- **When appropriate:** When read performance matters more than storage efficiency and write simplicity. A product listing page that would require 5 JOINs across normalized tables can be a single read from a denormalized document.
- **What it trades:** Write complexity increases — when a user changes their name, you must update it everywhere it's duplicated. Data inconsistency is possible if an update misses a copy. Storage increases (duplicated data).
- **Tradeoff:** Read queries become simpler and faster, but the write path becomes harder to maintain. This is essentially the same tradeoff as a materialized view.

**How these strategies compose:** They're not mutually exclusive. A common progression:

1. Start with a single database (vertical scaling — question 20)
2. Add read replicas when reads become the bottleneck
3. Add caching (Redis) for hot data
4. Denormalize specific high-traffic queries
5. Federate if distinct domains can be cleanly separated
6. Shard when a single domain's write volume exceeds one node's capacity

</details>

<details>
<summary>22. How do sharding strategies (hash-based, range-based, directory-based) work, what are the tradeoffs of each — when does hash-based sharding cause hot spots, when does range-based sharding suffer from uneven distribution, and why is directory-based sharding more flexible but introduces a single point of failure?</summary>

Sharding splits a single dataset across multiple database instances (shards), each holding a portion of the data. The shard key determines which shard a record belongs to.

**Hash-based sharding:** Apply a hash function to the shard key (e.g., `hash(userId) % numShards`) to determine the shard. Records are distributed pseudo-randomly across shards.

- **Pro:** Even distribution — a good hash function spreads data uniformly regardless of key patterns
- **Pro:** Simple to implement — deterministic mapping from key to shard
- **Con:** Range queries are impossible — "all users with IDs 1000-2000" must query every shard because the hash scatters them
- **Con:** Adding/removing shards requires rehashing most keys (~N/(N+1) records move when going from N to N+1 shards). Consistent hashing (question 23) solves this.
- **When it causes hot spots:** When many records share the same shard key value. If you shard by `userId` and one user generates 50% of all writes (a celebrity on a social platform), that user's shard becomes a hot spot. The hash distributes users evenly, but can't distribute one user's traffic across shards.

**Range-based sharding:** Assign contiguous ranges of the shard key to each shard. E.g., shard 1 holds users A-F, shard 2 holds G-L, etc. Or by date: shard 1 holds January data, shard 2 holds February.

- **Pro:** Range queries are efficient — "all orders from March" hits exactly one shard
- **Pro:** Adding shards is straightforward — split an existing range in two
- **Con:** Uneven distribution when the key distribution is skewed. If you shard alphabetically and 40% of usernames start with S-Z, that shard is overloaded while A-F is underutilized.
- **Con:** Time-based ranges create hot spots — the "current month" shard handles all writes while historical shards are idle.
- **When to use:** Data with natural ordering that supports range queries (time-series data, geographic data), and when the key distribution is relatively uniform.

**Directory-based sharding:** A lookup table (directory) maps each key (or key range) to a shard. Every query first consults the directory to find the right shard.

- **Pro:** Maximum flexibility — you can rebalance by updating the directory without moving all data at once. You can handle hot spots by splitting a single key's data across shards.
- **Pro:** Supports arbitrary mapping logic — not limited to hash or range functions
- **Con:** The directory is a single point of failure and a performance bottleneck. Every read and write starts with a directory lookup.
- **Con:** The directory itself must be highly available and low-latency — typically backed by a replicated store like etcd or ZooKeeper.
- **When to use:** Systems where rebalancing flexibility is worth the operational complexity (large-scale multi-tenant platforms where tenant sizes vary dramatically).

**Summary:**

| Strategy | Distribution | Range queries | Resharding cost | Complexity |
|----------|-------------|---------------|-----------------|------------|
| Hash-based | Even | Impossible | High (without consistent hashing) | Low |
| Range-based | Depends on key distribution | Efficient | Medium (split ranges) | Medium |
| Directory-based | Controllable | Depends on mapping | Low (update directory) | High |

**In interviews:** Hash-based is the default for most scenarios. Mention range-based when the interviewer asks about time-series or range queries. Mention directory-based when discussing rebalancing flexibility or multi-tenant systems.

</details>

<details>
<summary>23. How does consistent hashing work and why is it critical for distributed systems — explain the problem it solves (minimizing data movement when nodes are added or removed), how virtual nodes improve load distribution, and where consistent hashing appears in practice (load balancers, distributed caches, partitioned databases)?</summary>

**The problem:** With naive hash-based sharding (`hash(key) % N`), adding or removing a node changes N, which remaps almost every key. If you go from 4 to 5 nodes, ~80% of keys move to a different node. For a cache cluster, this means 80% cache miss rate immediately — a thundering herd hitting your database.

**How consistent hashing works:**

1. Imagine a hash ring from 0 to 2^32 (or any large range). Both nodes and keys are hashed onto this ring.
2. To find which node owns a key, walk clockwise from the key's position on the ring until you hit a node. That node owns the key.
3. When a node is added, it takes responsibility for the keys between itself and the previous node on the ring. Only those keys move — all other key-to-node mappings stay the same.
4. When a node is removed, its keys shift to the next node clockwise. Again, only that node's keys are affected.

**Result:** Adding or removing a node moves only ~1/N of the keys (where N is the number of nodes), instead of nearly all of them.

**The virtual nodes improvement:**

With few physical nodes, the hash ring can be unbalanced — one node might own a much larger arc than others. Virtual nodes solve this:

- Each physical node gets multiple positions on the ring (e.g., 150-200 virtual nodes per physical node).
- Instead of "Node A" at one point, you have "Node A-1", "Node A-2", ..., "Node A-150" scattered around the ring.
- This distributes the ring more evenly, so each physical node handles roughly 1/N of the keyspace.
- When a node is removed, its virtual nodes' keys distribute across many other nodes (instead of all going to one neighbor), preventing a single node from being overwhelmed.

**Where consistent hashing appears in practice:**

- **Distributed caches (Memcached, Redis Cluster):** Determines which cache node holds a given key. Adding a cache node only invalidates ~1/N of cached data instead of all of it. This is the canonical use case (as discussed in question 7).
- **Load balancers:** Consistent hashing-based routing ensures the same client/key always hits the same backend, enabling server-side caching and stateful connections without session affinity cookies.
- **Partitioned databases (Cassandra, DynamoDB):** Determines which node owns a partition. Cassandra uses consistent hashing with virtual nodes (called "vnodes") to distribute data across the cluster.
- **CDNs:** Determine which edge server caches a given piece of content, so requests for the same content consistently hit the same server.
- **Kafka:** Assigns partitions to consumers in a consumer group using a similar ring-based approach.

</details>

## Availability & Failure Modes

<details>
<summary>24. What are network partitions and single points of failure in distributed systems -- why do network partitions happen even within a single data center, how do they force the consistency-vs-availability tradeoff, and how do you identify and eliminate single points of failure in a system design?</summary>

**Network partitions:** A network partition occurs when nodes in a distributed system can't communicate with each other, even though each node is individually healthy. The system splits into isolated groups that can't coordinate.

**Why partitions happen even within a single data center:**
- **Switch/router failures:** A top-of-rack switch fails, isolating all servers in that rack from the rest of the datacenter
- **Network congestion:** Overloaded network links drop packets, causing timeouts that look like partitions to the application
- **Misconfiguration:** A firewall rule change, VLAN misconfiguration, or BGP route change can partition segments of the network
- **Partial failures:** A NIC on a single server fails — that server can't reach anything, but everything else works. From that server's perspective, the world is partitioned.

These aren't theoretical — Google, AWS, and Azure all publish post-mortems involving intra-datacenter partitions.

**How partitions force the consistency-vs-availability tradeoff:**

During a partition, nodes on each side of the split can receive client requests but can't coordinate with nodes on the other side. You have two choices (as covered in question 16 — CAP theorem):

- **Choose consistency:** Nodes that can't reach a quorum refuse to serve requests. The system is correct but partially unavailable.
- **Choose availability:** All nodes continue serving requests independently. The system is available but may serve stale or conflicting data.

**Single points of failure (SPOF):**

A SPOF is any component whose failure takes down the entire system. Common SPOFs:

| Component | Why it's a SPOF | How to eliminate it |
|-----------|----------------|---------------------|
| Single database instance | All reads/writes fail | Primary + replica with automated failover |
| Single load balancer | No traffic reaches backends | Active-passive or active-active LB pair |
| Single DNS provider | Domain doesn't resolve | Multiple DNS providers (Route53 + Cloudflare) |
| Single region deployment | Regional outage takes everything down | Multi-region with failover |
| Single message broker | Async processing halts | Clustered broker (Kafka cluster, RabbitMQ cluster) |
| Coordination service (ZooKeeper) | Leader election, config, locks all fail | 3 or 5 node cluster (tolerates minority failures) |

**How to identify SPOFs in a design:**

Walk through your architecture diagram and for each component ask: "If this dies right now, what breaks?" If the answer is "everything" or "a critical user-facing flow," it's a SPOF.

**Elimination patterns:**
- **Redundancy:** Run 2+ instances of every critical component
- **Failover:** Automated health checks + promotion (database failover, LB health checks removing unhealthy backends)
- **Graceful degradation:** If a non-critical dependency fails, the system continues with reduced functionality rather than failing entirely (e.g., show cached recommendations instead of real-time ones)
- **Avoid centralized coordination when possible:** Designs that require a single coordinator (directory-based sharding, centralized lock service) inherit that coordinator as a SPOF

**In interviews:** After drawing your high-level design, proactively point out "this is a single point of failure, and here's how I'd address it." This signals production-readiness thinking.

</details>

<details>
<summary>25. How do cascading failures propagate through a system and what design patterns prevent them — explain how a single slow dependency can take down an entire service fleet, why timeouts alone are insufficient, and how circuit breakers, bulkheads, and graceful degradation work together to contain failures?</summary>

**How cascading failures propagate:**

Imagine Service A calls Service B, which calls Service C. Service C becomes slow (not down — slow). Here's the cascade:

1. Service C responds in 30 seconds instead of 100ms
2. Service B's threads/connections are blocked waiting for C. B's connection pool fills up.
3. New requests to B can't get a connection — they queue up and time out
4. Service A is now waiting for B. A's connection pool fills up.
5. A's callers (users, other services) start timing out
6. The entire request chain is down — triggered by one slow service

The key insight: **slow is worse than down.** A dead service fails fast — callers get an error immediately. A slow service holds resources hostage, consuming connections, threads, and memory while doing nothing useful.

**Why timeouts alone are insufficient:**

Timeouts help — a 2-second timeout on calls to Service C means you wait 2 seconds instead of 30. But:

- During those 2 seconds, you're still holding a connection. At 1,000 QPS, you accumulate 2,000 in-flight requests waiting for the timeout.
- Every request still hits the failing service, adding load to an already struggling dependency.
- Timeouts don't prevent retries — clients often retry on timeout, multiplying the load (retry storm).

Timeouts limit the damage per request but don't limit the total damage to your system.

**Circuit breakers:**

A circuit breaker wraps calls to a dependency and tracks failures. When failures exceed a threshold, the circuit "opens" — subsequent calls fail immediately without even attempting the call. After a cooldown period, the circuit enters "half-open" state and allows a test request through. If it succeeds, the circuit closes (normal operation). If it fails, the circuit stays open.

```
Closed → (failures exceed threshold) → Open → (cooldown expires) → Half-Open
  ↑                                                                      |
  └────── (test request succeeds) ──────────────────────────────────────┘
```

**What this achieves:** The failing dependency gets relief (no more traffic), callers fail fast (no waiting for timeouts), and recovery is automatic (half-open probes).

**Bulkheads:**

Named after ship compartments that prevent flooding from spreading. In software, bulkheads isolate failure domains:

- **Connection pool isolation:** Give each dependency its own connection pool. If Service C's pool is exhausted, calls to Service D still have their own pool and work fine.
- **Thread pool isolation:** Dedicate separate thread pools to different dependencies. A slow dependency saturates its own pool but doesn't affect others.
- **Service isolation:** Deploy different critical paths on separate infrastructure. The checkout flow and the recommendation flow use different server groups — a recommendation failure can't affect checkout.

Without bulkheads, all dependencies share the same resources, so one bad dependency can exhaust everything.

**Graceful degradation:**

Design the system to provide reduced functionality instead of complete failure when a dependency is unavailable:

- **Recommendation service is down?** Show popular items instead of personalized recommendations.
- **Search service is slow?** Serve results from a stale cache.
- **Payment processor timeout?** Queue the payment for retry and show "processing" to the user instead of an error.

This requires identifying which dependencies are critical (must succeed for the request to be meaningful) vs non-critical (nice to have, can be skipped or approximated).

**How they work together:**

| Pattern | What it does | Scope |
|---------|-------------|-------|
| Timeouts | Limits wait time per request | Per-request |
| Circuit breakers | Stops calling a failing dependency entirely | Per-dependency |
| Bulkheads | Prevents one failure from consuming shared resources | Per-dependency or per-flow |
| Graceful degradation | Returns partial results instead of errors | Per-feature |

A robust system uses all four: timeouts on every external call, circuit breakers on each dependency, bulkheaded connection pools, and graceful degradation paths for non-critical features.

</details>

## Applied Design Reasoning

<details>
<summary>26. Given a system that needs to handle 10,000 writes per second with each record averaging 1 KB — walk through the back-of-the-envelope estimation for daily storage, monthly storage, read QPS assuming a 10:1 read-to-write ratio, bandwidth requirements, and how these numbers influence whether you need sharding, caching, or a CDN. Show your math and explain which numbers matter most for the design.</summary>

**Given:** 10,000 writes/sec, 1 KB per record, 10:1 read-to-write ratio.

**Storage:**

- Daily writes: 10,000/sec * 86,400 sec/day = 864,000,000 = ~864M records/day
- Daily storage: 864M * 1 KB = 864 GB/day (~0.86 TB/day)
- Monthly storage: 864 GB * 30 = ~25.9 TB/month
- Yearly storage: ~310 TB/year

These numbers don't include indexes (typically 1.5-3x raw data size for B-tree indexes), replication (2-3x for replicas), or backups.

**QPS:**

- Write QPS: 10,000 (given)
- Read QPS: 10,000 * 10 = 100,000 reads/sec
- Peak (2-3x average): 200,000-300,000 reads/sec

**Bandwidth:**

- Write bandwidth: 10,000/sec * 1 KB = 10 MB/sec = 80 Mbps
- Read bandwidth: 100,000/sec * 1 KB = 100 MB/sec = 800 Mbps
- Peak read bandwidth: ~200 MB/sec = 1.6 Gbps

**What these numbers tell us about the design:**

**Sharding is likely necessary for writes.** 10K writes/sec is approaching the limit for a single PostgreSQL instance (which handles roughly 5K-50K writes/sec depending on record size and indexes). With indexes and replication overhead, a single node would be under pressure. You'd want at least 2-4 write shards with room to grow.

**Caching is essential for reads.** 100K reads/sec is well beyond a single database's capacity. Options:
- Redis handles 100K+ ops/sec easily on a single instance. A Redis cluster could absorb most of the read traffic.
- With a 90% cache hit rate, only 10K reads/sec hit the database — easily manageable with a few read replicas.

**A CDN helps only if the data is cacheable and user-facing.** If these are API responses that vary per user, a CDN doesn't help. If these are public, static-ish records (product listings, articles), a CDN can absorb a significant portion of the 100K reads/sec.

**Storage drives the cost discussion.** 25 TB/month means you need a data retention policy. Storing everything forever costs ~300 TB/year in primary storage alone (plus replicas and indexes). Consider:
- Archiving old data to cheaper storage (S3) after N months
- Compacting or aggregating old records
- TTL-based expiration if the data has limited useful lifetime

**Which numbers matter most:**

1. **Write QPS (10K/sec)** — This determines whether you need sharding. It's the hardest bottleneck to solve because you can't cache writes.
2. **Read QPS (100K/sec)** — This determines your caching and replication strategy. High but solvable with Redis.
3. **Daily storage (864 GB)** — This determines whether a single node can hold the data and how quickly you'll outgrow disk. At this rate, retention policy is a day-one decision, not an afterthought.
4. **Bandwidth (800 Mbps reads)** — Significant but within the capacity of modern network infrastructure. Becomes the bottleneck only if records are much larger than 1 KB.

</details>

<details>
<summary>27. You're designing a system where users upload images and other users view them — walk through how you would combine building blocks (blob storage, CDN, metadata database, message queue for processing) into a coherent architecture, explain why each building block was chosen over alternatives, and identify where caching belongs and which consistency model applies to each data path.</summary>

**High-level architecture:**

```
User Upload Flow:
  Client → API Server → Blob Storage (S3)
                      → Metadata DB (PostgreSQL)
                      → Message Queue → Image Processing Workers
                                              ↓
                                        Blob Storage (processed versions)

User View Flow:
  Client → CDN → Blob Storage (cache miss)
  Client → API Server → Cache (Redis) → Metadata DB (cache miss)
```

**Building blocks and why each was chosen:**

**1. Blob storage (S3/GCS) for images:**
- Images are large binary files (100 KB - 10 MB). Storing them in a database is expensive and slow — databases are optimized for structured queries, not serving binary blobs.
- Object storage gives you: virtually unlimited capacity, built-in redundancy (11 nines durability), HTTP-accessible URLs, and cost-effective storage tiers for old images.
- Alternative rejected: storing images on local disk — not scalable, single point of failure, no CDN integration.

**2. Metadata database (PostgreSQL) for image records:**
- Each image has metadata: uploader ID, upload timestamp, title, tags, dimensions, processing status, URLs for each processed version. This is structured, relational, and queryable (e.g., "all images by user X, sorted by date").
- PostgreSQL gives ACID transactions (important for ensuring metadata and processing status stay consistent), flexible querying, and straightforward scaling with read replicas.
- Alternative rejected: storing metadata in S3 object tags — no querying, no transactions, limited to 10 tags per object.

**3. Message queue (SQS/Kafka) for async processing:**
- After upload, the original image needs processing: generate thumbnails (150px, 300px, 600px), optimize for web (WebP conversion, compression), extract EXIF metadata, run content moderation.
- This takes 2-10 seconds per image. The user shouldn't wait for it — the upload API returns immediately, and processing happens asynchronously.
- The queue enables: decoupling (upload service doesn't know about processing logic), load leveling (burst uploads don't overwhelm processors), retry (failed processing is retried automatically), and scaling (add more workers during peak).
- Alternative rejected: synchronous processing during upload — adds seconds of latency to the upload response, and a processing failure would fail the entire upload.

**4. CDN for serving images:**
- Images are read-heavy (uploaded once, viewed thousands of times) and immutable (a processed image never changes — you create a new version with a new URL).
- CDN caches images at edge locations worldwide, reducing latency from ~100ms to ~10ms for most users. Since images are immutable, you can set very long TTLs (1 year) with versioned URLs.
- The CDN also offloads bandwidth from your blob storage — S3 egress costs are significant at scale.
- Alternative rejected: serving directly from S3 — works at low scale but adds latency for distant users and S3 egress costs add up.

**Where caching belongs:**

| Data | Cache layer | Strategy | TTL |
|------|------------|----------|-----|
| Image files | CDN + browser cache | Immutable with versioned URLs | 1 year |
| Image metadata (for viewing) | Redis (cache-aside) | Invalidate on update | 5-15 min |
| User's image list | Redis (cache-aside) | Invalidate on upload/delete | 1-5 min |
| Processing status | No cache | Always read from DB | N/A |

**Consistency model per data path:**

- **Image upload → metadata creation:** Strong consistency. The user uploads an image and immediately sees it in their gallery. The metadata write goes to the PostgreSQL primary, and the user's reads route to the primary for a few seconds (read-your-writes, as discussed in question 15).
- **Image viewing by other users:** Eventual consistency. Other users see the image after CDN caching, Redis caching, and replication converge — a few seconds of delay is invisible to them.
- **Processing status:** Eventual consistency. The upload page polls for processing completion. A few seconds of delay between "processing done" and the user seeing "ready" is fine.
- **Image deletion:** The metadata delete should be strongly consistent (the uploader expects it gone immediately). CDN invalidation is eventually consistent — stale cached copies may be served for a short window (CDN TTL or explicit purge propagation time).

</details>