# Redis

> **20 questions** — 9 theory, 8 practical, 3 experience

- Core data structures: strings, hashes, lists, sets, sorted sets, streams
- Single-threaded execution model, event loop, and sub-millisecond latency
- Persistence: RDB snapshots vs AOF, and when to use both or neither
- Caching strategies: cache-aside, read-through, write-through, write-behind, refresh-ahead, cross-service invalidation patterns
- TTL, eviction policies (allkeys-lru, volatile-lru, allkeys-lfu, noeviction)
- Thundering herd / cache stampede: causes and solutions
- Distributed locks: SETNX with expiry, failure modes, Redlock debate (Kleppmann vs Antirez)
- High availability: Sentinel for failover, Cluster for hash-slot sharding
- Common patterns: rate limiting with sorted sets, leaderboards, counters, session storage with TTL, job queues (BullMQ)

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

</details>## Practical — Data Structures & Caching

<details>
<summary>10. Implement cache-aside for a Node.js service that caches database query results in Redis — show the lookup → miss → fetch → populate flow, how to handle cache invalidation when data changes (including across multiple services), and what consistency issues can arise between cache and database</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Implement rate limiting using Redis sorted sets — show the sliding window approach where each request adds a timestamped entry, expired entries are trimmed, and the count determines whether to allow or reject. Explain why sorted sets are better than simple counters for sliding window rate limiting</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Implement a real-time leaderboard using Redis sorted sets — show how ZADD adds scores, ZRANK/ZREVRANK retrieves a player's rank, and ZRANGE/ZREVRANGE fetches top-N players. Explain how to handle ties, how to get a player's rank among millions of entries, and what the time complexity is for each operation</summary>

<!-- Answer will be added later -->

</details><details>
<summary>13. Implement session storage with automatic expiry — show how to store session data in a Redis hash with TTL, how to extend the TTL on activity, and what happens when Redis evicts sessions under memory pressure with different eviction policies</summary>

<!-- Answer will be added later -->

</details><details>
<summary>14. Implement cache stampede prevention for a heavily-read cache key — show the mutex/lock approach (only one request repopulates while others wait or serve stale data), the probabilistic early expiration approach, and explain the tradeoffs between them</summary>

<!-- Answer will be added later -->

</details>

## Practical — Messaging & Coordination

<details>
<summary>15. Implement a distributed lock using Redis for a critical section that must not run concurrently — show the SETNX with EX approach, implement proper unlock (check-then-delete with Lua or conditional SET), demonstrate what happens when the lock holder crashes before releasing, and explain why you should set the lock TTL carefully</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Implement a job queue pattern using BullMQ with Redis -- show the basic producer/consumer setup in Node.js, explain how BullMQ uses Redis data structures under the hood (lists, sorted sets for delayed jobs, streams for events), how to handle failed jobs (retry strategies, dead letter queues), and what Redis configuration matters for reliable job processing (persistence, maxmemory-policy)</summary>

<!-- Answer will be added later -->

</details>

## Practical — Production Setup

<details>
<summary>17. Configure a managed Redis instance for a production Node.js application — show the key configuration decisions (maxmemory-policy, persistence settings, connection limits), the Node.js client setup with ioredis (connection pooling, retry strategy, error handling), and explain what breaks in production with each misconfiguration</summary>

<!-- Answer will be added later -->

</details>---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time you implemented Redis caching for a production application — what caching strategy did you choose, how did you handle cache invalidation, and what issues did you encounter (thundering herd, stale data, memory pressure)?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>19. Describe a time you debugged a Redis performance issue in production — what were the symptoms (latency spikes, connection timeouts, memory issues), how did you diagnose the root cause, and what was the fix?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>20. Tell me about a time you had to decide between Redis and another technology (Kafka, PostgreSQL, Memcached) for a specific use case — what were the requirements, what drove your decision, and how did it work out?</summary>

<!-- Answer framework will be added later -->

</details>

