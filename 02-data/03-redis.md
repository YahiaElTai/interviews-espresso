# Redis

> **35 questions** — 12 theory, 23 practical

- Core data structures: strings, hashes, lists, sets, sorted sets, streams
- Single-threaded execution model, event loop, and sub-millisecond latency
- Persistence: RDB snapshots vs AOF, and when to use both or neither
- Caching strategies: cache-aside, read-through, write-through, write-behind, refresh-ahead, cross-service invalidation patterns
- Redis as cache vs primary store: durability tradeoffs, when Redis is the source of truth (sessions, rate limits, leaderboards), backup and recovery considerations
- TTL, eviction policies (allkeys-lru, volatile-lru, allkeys-lfu, noeviction)
- Memory management: key naming conventions for memory efficiency, estimating memory per data structure, MEMORY USAGE command, fragmentation ratio monitoring
- Thundering herd / cache stampede: causes and solutions
- Pub/sub: delivery guarantees, fire-and-forget, and failure modes
- Redis Streams: consumer groups, acknowledgment, persistence, and when to use Kafka instead
- Distributed locks: SETNX with expiry, failure modes, Redlock debate (Kleppmann vs Antirez)
- High availability: Sentinel for failover, Cluster for hash-slot sharding
- Common patterns: rate limiting with sorted sets, leaderboards, counters, session storage with TTL, job queues (BullMQ)
- Pipelining for throughput, MULTI/EXEC transactions — atomicity guarantees and limitations vs SQL transactions
- Client libraries and connection management: ioredis vs node-redis, connection pooling, reconnection strategies, serialization (JSON vs MessagePack)
- Production deployment: managed Redis (ElastiCache/Memorystore), maxmemory sizing, replica configuration
- Monitoring and diagnostics: SLOWLOG, INFO memory, CLIENT LIST, latency spikes, cache hit/miss ratio, key expiration and eviction metrics

---

## Foundational

<details>
<summary>1. What data structures does Redis provide (strings, hashes, lists, sets, sorted sets, streams) — why does Redis have so many specialized structures instead of just key-value strings, when would you choose each one for different application needs, and how does the choice of data structure affect both performance and memory usage?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Why is Redis single-threaded yet capable of handling hundreds of thousands of operations per second with sub-millisecond latency — how does the event loop model work, what operations can still block Redis despite being single-threaded, and when does this model become a limitation?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. What are Redis's persistence options (RDB snapshots and AOF) — what tradeoffs does each make between data safety and performance, when would you use both together, and when should you run Redis with no persistence at all?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. How do caching strategies differ — what are the mechanics and tradeoffs of cache-aside, read-through, write-through, write-behind, and refresh-ahead, how do you choose between them for different use cases, and what are the cross-service cache invalidation patterns when multiple services share cached data?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How do TTL and eviction policies work in Redis — what does each eviction policy (allkeys-lru, volatile-lru, allkeys-lfu, noeviction) do, how do you choose the right one for your use case, and what are the consequences of choosing wrong (unexpected data loss vs OOM crashes)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What is the thundering herd / cache stampede problem — why does it happen when a popular cache key expires, what makes it dangerous at scale, and what are the different solutions (mutex locking, probabilistic early expiration, stale-while-revalidate)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do Redis pipelining and MULTI/EXEC transactions work — what does each optimize (network round trips vs atomicity), what are the atomicity guarantees and limitations of MULTI/EXEC compared to SQL transactions (no rollback on individual command failure, no conditional logic), and when is each appropriate?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do distributed locks work in Redis using SETNX with expiry — what are the failure modes (clock drift, GC pauses, split brain), what is the Redlock algorithm and why did Martin Kleppmann argue it's fundamentally flawed, and what's the practical takeaway for teams that need distributed coordination?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How does Redis provide high availability — what does Sentinel do for automatic failover and how does it detect failures, how does Redis Cluster partition data across nodes using hash slots, and when would you choose Sentinel (replication + failover) vs Cluster (sharding + failover)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why is Redis pub/sub fire-and-forget -- what happens when a subscriber disconnects or falls behind on message consumption, what failure modes does this create, and when is fire-and-forget delivery acceptable (e.g., cache invalidation broadcasts, real-time UI updates) vs when is it dangerous?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How do Redis Streams address the reliability gaps of pub/sub -- how do consumer groups and acknowledgment work, what persistence guarantees do Streams provide, and when should you skip both pub/sub and Streams in favor of Kafka or RabbitMQ?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. When should Redis serve as the primary data store (sessions, rate limits, leaderboards) vs purely as a cache layer in front of a database -- what durability tradeoffs does each role require, how do persistence and replication requirements change, and what backup and recovery strategies apply when Redis holds data that doesn't exist elsewhere?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Data Structures & Caching

<details>
<summary>13. Implement cache-aside for a Node.js service that caches database query results in Redis — show the lookup → miss → fetch → populate flow, how to handle cache invalidation when data changes (including across multiple services), and what consistency issues can arise between cache and database</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Implement rate limiting using Redis sorted sets — show the sliding window approach where each request adds a timestamped entry, expired entries are trimmed, and the count determines whether to allow or reject. Explain why sorted sets are better than simple counters for sliding window rate limiting</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Implement a real-time leaderboard using Redis sorted sets — show how ZADD adds scores, ZRANK/ZREVRANK retrieves a player's rank, and ZRANGE/ZREVRANGE fetches top-N players. Explain how to handle ties, how to get a player's rank among millions of entries, and what the time complexity is for each operation</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Implement atomic counters and rate limiting using INCR with EXPIRE — show the basic INCR + EXPIRE approach for fixed-window rate limiting, explain the race condition where a key is incremented but the EXPIRE never runs, and demonstrate the Lua script or MULTI/EXEC solution that guarantees atomicity</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Implement session storage with automatic expiry — show how to store session data in a Redis hash with TTL, how to extend the TTL on activity, and what happens when Redis evicts sessions under memory pressure with different eviction policies</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Configure Redis eviction policies for two scenarios: a pure cache (where losing data is fine) and a session store (where losing data matters) — show the maxmemory and maxmemory-policy configuration for each, explain why the wrong policy for the wrong use case causes either OOM crashes or unexpected data loss</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Implement cache stampede prevention for a heavily-read cache key — show the mutex/lock approach (only one request repopulates while others wait or serve stale data), the probabilistic early expiration approach, and explain the tradeoffs between them</summary>

<!-- Answer will be added later -->

</details>

## Practical — Messaging & Coordination

<details>
<summary>20. Implement a pub/sub notification system where a service publishes events and multiple subscribers react — show the publisher and subscriber code, demonstrate what happens when a subscriber is temporarily disconnected (messages are lost), and explain when this fire-and-forget model is acceptable vs when you need Redis Streams or a proper message queue</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Implement a distributed lock using Redis for a critical section that must not run concurrently — show the SETNX with EX approach, implement proper unlock (check-then-delete with Lua or conditional SET), demonstrate what happens when the lock holder crashes before releasing, and explain why you should set the lock TTL carefully</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Implement a job queue pattern using BullMQ with Redis -- show the basic producer/consumer setup in Node.js, explain how BullMQ uses Redis data structures under the hood (lists, sorted sets for delayed jobs, streams for events), how to handle failed jobs (retry strategies, dead letter queues), and what Redis configuration matters for reliable job processing (persistence, maxmemory-policy)</summary>

<!-- Answer will be added later -->

</details>

## Practical — Production Setup

<details>
<summary>23. When configuring persistence for a managed Redis instance (ElastiCache, Memorystore) -- what are the persistence options available (RDB snapshots, AOF), what do the different fsync modes (always, everysec, no) trade off between durability and performance, how do you choose the right configuration for a typical web application, and what does a managed service handle for you vs what you still need to decide?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Configure a managed Redis instance for a production Node.js application — show the key configuration decisions (maxmemory-policy, persistence settings, connection limits), the Node.js client setup with ioredis (connection pooling, retry strategy, error handling), and explain what breaks in production with each misconfiguration</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Use Redis pipelining to batch multiple commands and reduce round-trip latency — show a before (sequential commands) and after (pipelined) implementation, demonstrate the throughput difference, and explain when pipelining helps vs when MULTI/EXEC is more appropriate</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Configure a Node.js application to handle Redis failover gracefully -- show the ioredis configuration for Sentinel-aware connections (or managed service equivalents), implement retry logic and connection event handling for failover scenarios, demonstrate what the application experiences during a failover (connection drops, read-after-write inconsistency), and show the monitoring/alerting configuration you would set up to detect failover events</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Configure Redis connection management in a Node.js application -- compare ioredis vs node-redis (API differences, Cluster/Sentinel support, reconnection behavior), show proper connection pooling and reconnection handling with ioredis, demonstrate using connection events for health monitoring, explain the tradeoffs of JSON vs MessagePack for value serialization (readability vs size vs speed), and what happens when your application creates too many connections or doesn't handle disconnections gracefully</summary>

<!-- Answer will be added later -->

</details>

## Practical — Monitoring & Troubleshooting

<details>
<summary>28. Set up Redis monitoring using SLOWLOG, INFO, and CLIENT LIST -- show the commands to configure SLOWLOG (threshold, max entries), interpret INFO memory output (used_memory, fragmentation ratio, evicted keys), use CLIENT LIST to identify connection issues (too many connections, blocked clients, idle connections), and explain what healthy vs unhealthy numbers look like for a production cache</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Your Redis instance is experiencing periodic latency spikes — walk through the diagnosis: check SLOWLOG for expensive commands, look for background save (BGSAVE) interference, check memory fragmentation, identify if the spikes correlate with eviction activity, and explain how each root cause manifests differently</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Your cache hit ratio has dropped from 95% to 60% — walk through the analysis: how to measure hit/miss ratio from INFO stats (keyspace_hits, keyspace_misses), identify whether it's caused by TTL misconfiguration, key space changes, eviction under memory pressure, or traffic pattern shifts, and what the fix is for each scenario</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. Demonstrate how to audit and optimize Redis memory usage -- show how to use MEMORY USAGE to measure the cost of individual keys, how key naming conventions affect memory (short prefixes vs verbose names, hash encoding thresholds), how to estimate memory consumption for different data structures (a hash with N fields vs N separate string keys), and what the memory fragmentation ratio in INFO memory tells you about allocation efficiency</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>32. Tell me about a time you implemented Redis caching for a production application — what caching strategy did you choose, how did you handle cache invalidation, and what issues did you encounter (thundering herd, stale data, memory pressure)?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>33. Describe a time you debugged a Redis performance issue in production — what were the symptoms (latency spikes, connection timeouts, memory issues), how did you diagnose the root cause, and what was the fix?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Tell me about a time you had to decide between Redis and another technology (Kafka, PostgreSQL, Memcached) for a specific use case — what were the requirements, what drove your decision, and how did it work out?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>35. Describe a time you set up or improved Redis high availability — what was the architecture, what failure scenarios did you plan for, and did you ever experience an actual failover?</summary>

<!-- Answer framework will be added later -->

</details>
