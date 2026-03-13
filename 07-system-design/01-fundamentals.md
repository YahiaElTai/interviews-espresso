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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Why is back-of-the-envelope estimation important in a design interview, and how do you calculate QPS, storage, and bandwidth from product requirements -- walk through a concrete example showing how you go from "10 million daily active users" to specific numbers, why should you memorize powers of two (2^10 through 2^40) as estimation anchors, and what mistakes do candidates commonly make in estimation?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What are the key latency numbers every engineer should know (memory read, SSD seek, network round-trip, disk sequential read) and how do these numbers influence design decisions -- when do they tell you "this must be cached," "this can hit the database on every request," or "this needs to be colocated"?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. How do you define APIs during a system design interview — why should you lock down the API contract early, how do you decide between REST endpoints vs RPC-style vs event-driven interfaces, and how does the API definition expose assumptions about read/write ratios and access patterns that shape the rest of the design?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What does "designing for the common case" mean in system design — how do you identify the dominant access patterns (read-heavy vs write-heavy, latency-sensitive vs throughput-sensitive), why is it dangerous to optimize for edge cases too early, and how do latency vs throughput tradeoffs influence which building blocks you choose?</summary>

<!-- Answer will be added later -->

</details>

## Building Blocks — Traffic & Networking

<details>
<summary>6. Why do systems need load balancers, what is the difference between L4 (transport) and L7 (application) load balancing, how do you decide which layer to operate at, and how does a load balancer differ from a reverse proxy — when do the two roles overlap and when are they distinct?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do load balancing algorithms work and when would you choose each one — compare round-robin, weighted round-robin, least connections, and consistent hashing, explain the tradeoffs of each (simplicity vs fairness vs stickiness), and why consistent hashing is critical for caching layers and sharded datastores?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why are CDNs a fundamental building block in system design — how do push vs pull CDN models work, when would you choose each, and how do you handle cache invalidation at the CDN layer (TTL expiry, explicit purge, versioned URLs)? What goes wrong when CDN caching strategy is poorly designed?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How do rate limiting algorithms (token bucket, sliding window, fixed window) work, what are the tradeoffs of each -- when would you apply rate limiting at the client vs the server, and what additional challenges does distributed rate limiting introduce when you have multiple API server instances?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How do you choose the right communication protocol for a given system design scenario — compare HTTP/REST, gRPC, WebSockets, SSE, and UDP in terms of latency, connection overhead, bidirectionality, and browser support, and explain the specific scenarios where each one is the right choice and where it would be a poor fit?</summary>

<!-- Answer will be added later -->

</details>

## Building Blocks — Data & Storage

<details>
<summary>11. What are the caching strategies (cache-aside, write-through, write-behind, refresh-ahead), how does each one work, what consistency tradeoffs does each introduce, and how do you decide which strategy fits a given access pattern — why is cache-aside the most common default, and when would write-through or write-behind be a better choice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Where can caching layers exist in a system (client, CDN, application, database) and why does the position of the cache matter — how do you reason about which layer to cache at, what happens when you have multiple caching layers that interact, and what are the common pitfalls (stale data, cache stampede, cold start)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why are message queues a fundamental building block in system design — explain how they enable decoupling, load leveling, and retry, how ordering guarantees and backpressure work, how task queues differ from message queues (dedicated workers processing jobs vs general pub/sub), and when introducing a queue adds unnecessary complexity vs when it's essential for the architecture?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Why is idempotency a critical design requirement in distributed systems -- explain what idempotency keys are, how they enable safe retries, the difference between at-least-once and exactly-once delivery semantics, and how you would design an idempotent API endpoint that handles duplicate requests gracefully?</summary>

<!-- Answer will be added later -->

</details>

## Consistency, Replication & Data Models

<details>
<summary>15. What are the consistency models you need to reason about in system design (strong, eventual, causal, read-your-writes) — explain what guarantees each model provides, why stronger consistency costs more in latency and availability, and how do you choose the right consistency model for different parts of the same system?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. What does the CAP theorem actually say and why is it frequently misunderstood — explain the real constraint (you must choose between consistency and availability during a network partition), how PACELC extends CAP to cover the non-partition case (latency vs consistency tradeoff), and how these theorems guide real design decisions rather than being academic trivia?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How do the three database replication models (single-leader, multi-leader, leaderless) work and what are the tradeoffs of each in terms of consistency, write availability, and latency -- when would you choose each one in a system design, and what failure modes are unique to each model?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. How do you decide between SQL and NoSQL in a system design — what are the real selection criteria (data model, query patterns, consistency requirements, scale characteristics, team expertise) beyond the surface-level "SQL for relational, NoSQL for everything else," and what are common mistakes teams make when choosing, and what are the consequences?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. How do distributed coordination primitives (leader election, distributed locks, consensus) work at a high level — why are protocols like Raft and Paxos necessary, what problems do tools like ZooKeeper and etcd solve, and when would you reach for a distributed lock vs an alternative approach in a system design?</summary>

<!-- Answer will be added later -->

</details>

## Scaling & Partitioning

<details>
<summary>20. How do you decide between vertical and horizontal scaling in a system design -- what are the limits of each approach, why is vertical scaling often the right first step, and what signals tell you it is time to move to horizontal scaling?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. What are the database-specific scaling strategies (read replicas, federation, denormalization) -- explain when each approach is appropriate, why read replicas only help read-heavy workloads, and how federation and denormalization trade normalization for scale?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. How do sharding strategies (hash-based, range-based, directory-based) work, what are the tradeoffs of each — when does hash-based sharding cause hot spots, when does range-based sharding suffer from uneven distribution, and why is directory-based sharding more flexible but introduces a single point of failure?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. How does consistent hashing work and why is it critical for distributed systems — explain the problem it solves (minimizing data movement when nodes are added or removed), how virtual nodes improve load distribution, and where consistent hashing appears in practice (load balancers, distributed caches, partitioned databases)?</summary>

<!-- Answer will be added later -->

</details>

## Availability & Failure Modes

<details>
<summary>24. What are network partitions and single points of failure in distributed systems -- why do network partitions happen even within a single data center, how do they force the consistency-vs-availability tradeoff, and how do you identify and eliminate single points of failure in a system design?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. How do cascading failures propagate through a system and what design patterns prevent them — explain how a single slow dependency can take down an entire service fleet, why timeouts alone are insufficient, and how circuit breakers, bulkheads, and graceful degradation work together to contain failures?</summary>

<!-- Answer will be added later -->

</details>

## Applied Design Reasoning

<details>
<summary>26. Given a system that needs to handle 10,000 writes per second with each record averaging 1 KB — walk through the back-of-the-envelope estimation for daily storage, monthly storage, read QPS assuming a 10:1 read-to-write ratio, bandwidth requirements, and how these numbers influence whether you need sharding, caching, or a CDN. Show your math and explain which numbers matter most for the design.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. You're designing a system where users upload images and other users view them — walk through how you would combine building blocks (blob storage, CDN, metadata database, message queue for processing) into a coherent architecture, explain why each building block was chosen over alternatives, and identify where caching belongs and which consistency model applies to each data path.</summary>

<!-- Answer will be added later -->

</details>