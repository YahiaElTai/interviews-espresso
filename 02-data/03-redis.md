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

Redis provides specialized data structures because operating on data server-side is fundamentally more efficient than fetching raw strings, deserializing client-side, modifying, re-serializing, and writing back. Each structure has optimized commands that execute atomically in O(1) or O(log N), avoiding round trips and race conditions.

**Core structures and when to use each:**

| Structure | Use case | Key commands |
|---|---|---|
| **String** | Simple cache values, counters, flags | `GET`, `SET`, `INCR`, `SETNX` |
| **Hash** | Object-like data (user profiles, sessions) — update individual fields without rewriting the whole value | `HGET`, `HSET`, `HMGET`, `HINCRBY` |
| **List** | Ordered sequences, message queues, activity feeds | `LPUSH`, `RPOP`, `LRANGE`, `BRPOP` (blocking) |
| **Set** | Unique collections, tagging, membership checks | `SADD`, `SISMEMBER`, `SINTER`, `SUNION` |
| **Sorted Set** | Rankings, leaderboards, rate limiting, priority queues — anything needing ordered data with fast rank lookups | `ZADD`, `ZRANK`, `ZRANGE`, `ZRANGEBYSCORE` |
| **Stream** | Event logs, consumer groups, audit trails — append-only log with consumer group support (like a lightweight Kafka) | `XADD`, `XREAD`, `XREADGROUP`, `XACK` |

**Performance and memory implications:**

- **Hashes vs JSON strings**: A hash with 5 fields uses `HGET` to fetch one field in O(1). A JSON string requires fetching the entire blob, parsing it, and writing it back for any update. For small hashes (< ~128 entries with short values), Redis uses a compact `ziplist` encoding that's more memory-efficient than individual keys.
- **Sorted sets**: O(log N) for insert and rank lookup via a skip list internally. This means leaderboards with millions of entries still return ranks in microseconds.
- **Lists vs streams**: Lists are simpler but lack consumer groups. Streams add message IDs, acknowledgment, and consumer group semantics — use them when multiple consumers need to process the same stream independently.

**The wrong structure costs you**: Picking the wrong structure means paying for full serialization round trips, O(N) rewrites, and lost atomicity — the performance and memory differences above compound quickly at scale.

</details>

<details>
<summary>2. Why is Redis single-threaded yet capable of handling hundreds of thousands of operations per second with sub-millisecond latency — how does the event loop model work, what operations can still block Redis despite being single-threaded, and when does this model become a limitation?</summary>

Redis is fast despite being single-threaded because its bottleneck was never CPU — it's memory access and network I/O. A single thread avoids all locking overhead, context switching, and cache-line contention that multi-threaded systems pay for.

**How the event loop works:**

Redis uses an event-driven model (similar in spirit to Node.js) built on OS-level I/O multiplexing (`epoll` on Linux, `kqueue` on macOS). A single thread:

1. Waits for events on all client sockets simultaneously via `epoll_wait`
2. Reads a complete command from a ready socket
3. Executes the command against in-memory data structures (microseconds)
4. Writes the response back to the socket buffer
5. Loops back to step 1

Since all data is in memory and most operations are O(1) or O(log N), each command completes in microseconds. The event loop can process 100K+ commands/second because it never blocks waiting for disk or network — it just cycles through ready sockets.

**Note on I/O threading (Redis 6+):** Redis 6 introduced multi-threaded I/O — multiple threads handle reading from and writing to sockets. But command execution is still single-threaded. This improves throughput on high-bandwidth workloads without changing the single-threaded execution model.

**Operations that block Redis:**

- `KEYS *` — Scans the entire keyspace. Use `SCAN` instead (cursor-based, non-blocking).
- `FLUSHALL`, `FLUSHDB` — Synchronous deletion of all keys (use `ASYNC` variant).
- Lua scripts — Execute atomically, so a long-running script blocks everything.
- `SORT` on large datasets, `LRANGE` on huge lists.
- `SAVE` — Synchronous RDB snapshot (use `BGSAVE` which forks).
- Large key deletion — Deleting a key holding millions of elements (sorted set, hash) blocks while freeing memory. Use `UNLINK` (async delete) instead of `DEL`.

**When single-threaded becomes a limitation:**

- **CPU-bound workloads**: If you're doing heavy Lua computation or have extremely high throughput needs (500K+ ops/sec), a single core is the ceiling. The solution is running multiple Redis instances and sharding (Redis Cluster).
- **Large key operations**: Any single command that touches millions of elements will block all other clients for the duration.
- **Multi-core utilization**: A single Redis instance uses one core. On a 16-core machine, you're leaving 15 cores idle. Redis Cluster or multiple instances per host solves this.

</details>

## Conceptual Depth

<details>
<summary>3. What are Redis's persistence options (RDB snapshots and AOF) — what tradeoffs does each make between data safety and performance, when would you use both together, and when should you run Redis with no persistence at all?</summary>

**RDB (Redis Database) snapshots:**

Redis forks the process and writes a point-in-time snapshot of all data to disk as a compact binary file. Configured with rules like `save 900 1` (snapshot if at least 1 key changed in 900 seconds).

- **Pros**: Compact file, fast restarts (load the binary dump), minimal runtime performance impact (fork + copy-on-write).
- **Cons**: Data loss between snapshots. If Redis crashes 10 minutes after the last snapshot, those 10 minutes of writes are gone. The fork itself can cause latency spikes on large datasets (the OS must copy page tables).

**AOF (Append Only File):**

Every write command is appended to a log file. On restart, Redis replays the log to reconstruct state. Configurable fsync policy:

- `appendfsync always` — fsync after every command. Safest, but slowest (kills throughput).
- `appendfsync everysec` — fsync once per second. Lose at most 1 second of data. **Best default for most workloads.**
- `appendfsync no` — Let the OS flush when it wants. Fast, but you could lose 30+ seconds of data.

- **Pros**: Much smaller data loss window. Human-readable log (useful for debugging).
- **Cons**: Larger file size than RDB. Slower restarts (replaying commands). AOF rewrite (compaction) can cause I/O spikes.

**Using both together:**

This is the recommended production setup when data durability matters. RDB gives you fast restarts and compact backups. AOF gives you minimal data loss. On restart, Redis prefers the AOF file (more complete) but falls back to RDB if AOF is missing.

**When to run with no persistence:**

- **Pure caching**: If Redis is only a cache in front of a database, losing all data just means a cold cache — the source of truth is elsewhere. No persistence avoids fork overhead and disk I/O entirely.
- **Ephemeral data**: Session tokens, rate-limit counters, or real-time metrics where losing data on restart is acceptable.
- **Performance-critical**: When you need every microsecond and can't tolerate fork latency spikes.

**Practical note**: Most managed Redis services (ElastiCache, Cloud Memorystore) default to RDB snapshots. If you need AOF-level durability on managed services, check if it's supported — some don't expose AOF configuration.

</details>

<details>
<summary>4. How do caching strategies differ — what are the mechanics and tradeoffs of cache-aside, read-through, write-through, write-behind, and refresh-ahead, how do you choose between them for different use cases, and what are the cross-service cache invalidation patterns when multiple services share cached data?</summary>

**Cache-Aside (Lazy Loading):**

The application manages the cache directly. On read: check cache, on miss fetch from DB, populate cache. On write: update DB, then invalidate (delete) the cache key.

- **Pros**: Simple, only caches data that's actually requested, application controls everything.
- **Cons**: First request is always a cache miss. Cache and DB can become inconsistent if the invalidation fails (write succeeds, delete fails — stale data persists).
- **Best for**: Most general-purpose caching. The default choice.

**Read-Through:**

The cache layer itself fetches from the DB on a miss — the application only talks to the cache. The cache acts as the primary read interface.

- **Pros**: Simpler application code (no cache-miss logic). Cache library handles population.
- **Cons**: Requires a cache library/proxy that supports this pattern. First request is still a miss.
- **Best for**: When you want to abstract caching entirely from business logic.

**Write-Through:**

Every write goes through the cache to the DB synchronously. The cache is always up to date.

- **Pros**: Cache is never stale. Reads are always fast after the first write.
- **Cons**: Write latency increases (two writes on the write path). Caches data that may never be read.
- **Best for**: Read-heavy workloads where staleness is unacceptable (e.g., user profiles that are read 100x for every write).

**Write-Behind (Write-Back):**

Writes go to the cache immediately, then asynchronously flush to the DB in batches.

- **Pros**: Very fast writes. Can batch/coalesce multiple writes into one DB operation.
- **Cons**: Risk of data loss if the cache crashes before flushing. Complex failure handling. Eventual consistency by design.
- **Best for**: Write-heavy workloads where some data loss risk is acceptable (analytics counters, view counts).

**Refresh-Ahead:**

The cache proactively refreshes entries before they expire, based on access patterns. If a key is accessed and is within a "refresh window" (e.g., 80% of TTL elapsed), trigger an async background refresh.

- **Pros**: Hot keys never expire — zero cache-miss latency for frequently accessed data.
- **Cons**: Wastes resources refreshing data that might not be requested again. Complex to implement.
- **Best for**: Hot keys with predictable access patterns (product catalog, feature flags).

**Choosing a strategy:**

| Scenario | Strategy |
|---|---|
| General purpose, simple setup | Cache-aside |
| Read-heavy, staleness unacceptable | Write-through + cache-aside |
| Write-heavy, some loss OK | Write-behind |
| Hot keys, predictable access | Refresh-ahead |

**Cross-service cache invalidation patterns:**

When multiple services read from a shared cache or maintain their own caches over the same data:

1. **Event-driven invalidation**: The service that owns the data publishes a change event (via Kafka, SNS, Redis Pub/Sub). Consumer services listen and invalidate their local cache entries. Most scalable approach.
2. **TTL-based eventual consistency**: Don't explicitly invalidate — rely on short TTLs (30-60 seconds). Simple but means stale reads for the TTL duration. Works when slight staleness is acceptable.
3. **Shared cache with versioned keys**: All services read from the same Redis instance. The owning service bumps a version counter on writes, and cache keys include the version. Old keys expire naturally. Avoids explicit invalidation.
4. **Redis Pub/Sub for invalidation signals**: The writing service publishes to a Redis channel. Other services subscribe and delete their cached copies. Simpler than Kafka but messages are fire-and-forget (no persistence — if a subscriber is down, it misses the invalidation).

</details>

<details>
<summary>5. How do TTL and eviction policies work in Redis — what does each eviction policy (allkeys-lru, volatile-lru, allkeys-lfu, noeviction) do, how do you choose the right one for your use case, and what are the consequences of choosing wrong (unexpected data loss vs OOM crashes)?</summary>

**TTL (Time To Live):**

Set with `EXPIRE key seconds` or `SET key value EX seconds`. Redis lazily deletes expired keys in two ways:
1. **Passive expiration**: When a client tries to access a key, Redis checks if it's expired and deletes it.
2. **Active expiration**: Redis periodically samples 20 random keys with TTL set, deletes expired ones, and repeats if more than 25% were expired. This prevents memory from filling with never-accessed expired keys.

**Eviction policies** kick in when Redis hits `maxmemory`. Redis must decide what to remove to make room:

| Policy | Scope | Algorithm | Behavior |
|---|---|---|---|
| `noeviction` | N/A | N/A | Returns errors on writes when memory is full. Reads still work. |
| `allkeys-lru` | All keys | Least Recently Used | Evicts the least recently accessed key. **Best general-purpose cache policy.** |
| `volatile-lru` | Only keys with TTL | LRU | Only evicts keys that have an expiry set. Non-TTL keys are safe. |
| `allkeys-lfu` | All keys | Least Frequently Used | Evicts keys accessed least often. Better than LRU for skewed access patterns (a few hot keys, many cold). |
| `volatile-lfu` | Only keys with TTL | LFU | LFU but only among keys with expiry. |
| `allkeys-random` | All keys | Random | Random eviction. Simple but unpredictable. |
| `volatile-random` | Only keys with TTL | Random | Random eviction among TTL keys only. |
| `volatile-ttl` | Only keys with TTL | Shortest TTL | Evicts keys closest to expiration. |

**How to choose:**

- **Pure cache (all data is replaceable)**: `allkeys-lru` or `allkeys-lfu`. LFU is better when some keys are much hotter than others (e.g., popular product pages).
- **Mixed workload (cache + persistent data)**: `volatile-lru`. Your persistent data (session state, counters) has no TTL and won't be evicted. Your cache data has TTL and gets evicted when memory is tight.
- **Job queues, locks, critical data**: `noeviction`. You'd rather get write errors and alert than silently lose a distributed lock or queued job.

**Consequences of choosing wrong:**

- **`noeviction` on a cache**: Redis stops accepting writes when full. Your application starts throwing errors instead of gracefully evicting stale cache entries. A cache that can't write is useless.
- **`allkeys-lru` when you have persistent data**: Redis will evict your session data, lock keys, or BullMQ job metadata to make room for new cache entries. Silent data loss in production.
- **`volatile-lru` but you forget to set TTL on cache keys**: Keys without TTL are never eviction candidates. Memory fills up, and Redis can't evict anything — behaves like `noeviction` and returns errors.

**Practical note**: Redis LRU is approximate, not exact. It samples 5 keys (configurable via `maxmemory-samples`) and evicts the least recently used among the sample. Increasing the sample size improves accuracy at a small CPU cost.

</details>

<details>
<summary>6. What is the thundering herd / cache stampede problem — why does it happen when a popular cache key expires, what makes it dangerous at scale, and what are the different solutions (mutex locking, probabilistic early expiration, stale-while-revalidate)?</summary>

**The problem:**

A popular cache key (e.g., homepage product list, cached at `products:featured`) is read 10,000 times per second. When its TTL expires, all 10,000 concurrent requests see a cache miss simultaneously. All of them hit the database to regenerate the same data at the same time. The database gets crushed under redundant identical queries.

**Why it's dangerous at scale:**

- The database goes from 0 queries/sec (everything was cached) to 10,000 identical queries in the same millisecond.
- If the query is expensive (joins, aggregations), it can saturate the DB connection pool or cause timeouts.
- While the DB is overwhelmed, other unrelated queries also fail — cascading failure.
- The irony: caching was supposed to protect the database, but the expiry event creates a worse spike than no cache at all.

**Solutions:**

**1. Mutex locking (single-flight):**

Only one request regenerates the cache. Others wait or get stale data.

```typescript
async function getWithMutex(key: string, fetchFn: () => Promise<string>, ttl: number): Promise<string> {
  const cached = await redis.get(key);
  if (cached) return cached;

  const lockKey = `lock:${key}`;
  // Try to acquire lock (SET NX EX = set if not exists, with expiry)
  const acquired = await redis.set(lockKey, '1', 'EX', 10, 'NX');

  if (acquired) {
    try {
      const fresh = await fetchFn();
      await redis.set(key, fresh, 'EX', ttl);
      return fresh;
    } finally {
      await redis.del(lockKey);
    }
  }

  // Lock not acquired — wait briefly and retry (or serve stale data)
  await new Promise((r) => setTimeout(r, 100));
  return getWithMutex(key, fetchFn, ttl);
}
```

- **Pros**: Only one DB hit. Simple to understand.
- **Cons**: Other requests are delayed while waiting. If the lock holder crashes, the lock TTL determines how long everyone waits.

**2. Probabilistic early expiration (XFetch / early recompute):**

Before the key actually expires, randomly refresh it early. Each request rolls a random chance to refresh based on how close the key is to expiring:

```typescript
// Concept: if the key has 10% of its TTL remaining, there's a chance
// we proactively refresh it before it expires
const ttlRemaining = await redis.ttl(key);
const shouldRefresh = Math.random() < Math.exp(-ttlRemaining / beta);
```

- **Pros**: No locking, no waiting. Stampede is probabilistically eliminated because one request will refresh the key before it expires.
- **Cons**: Slightly more complex. May cause occasional duplicate refreshes (but far fewer than a full stampede).

**3. Stale-while-revalidate:**

Store the data with a logical expiry (in the value) but a longer physical TTL. When the logical expiry passes, serve the stale value immediately while triggering an async background refresh.

```typescript
// Store with logical and physical TTL
const entry = { data: value, logicalExpiry: Date.now() + 60_000 };
await redis.set(key, JSON.stringify(entry), 'EX', 120); // physical TTL = 2x logical

// On read
const raw = await redis.get(key);
const entry = JSON.parse(raw);
if (Date.now() > entry.logicalExpiry) {
  // Serve stale immediately, refresh in background
  refreshInBackground(key);
}
return entry.data;
```

- **Pros**: Zero latency impact — users always get a response. Background refresh prevents stampede.
- **Cons**: Users may see stale data for a brief window. Slightly more complex data format.

**Which to use**: Mutex locking is the simplest for moderate traffic. Stale-while-revalidate is best for user-facing latency-sensitive paths. Probabilistic early expiration is elegant but harder to tune. In practice, combining stale-while-revalidate with a mutex on the background refresh gives you the best of both.

</details>

<details>
<summary>7. How do Redis pipelining and MULTI/EXEC transactions work — what does each optimize (network round trips vs atomicity), what are the atomicity guarantees and limitations of MULTI/EXEC compared to SQL transactions (no rollback on individual command failure, no conditional logic), and when is each appropriate?</summary>

**Pipelining:**

Sends multiple commands to Redis without waiting for each response. The client buffers commands and sends them in one batch; Redis processes them sequentially and sends all responses back together.

```typescript
const pipeline = redis.pipeline();
pipeline.set('user:1:name', 'Alice');
pipeline.set('user:1:email', 'alice@example.com');
pipeline.incr('stats:user-updates');
const results = await pipeline.exec(); // one round trip for all 3 commands
```

- **Optimizes**: Network round trips. Instead of 3 round trips (each ~0.5ms), you get 1. On high-latency connections, this is a massive win.
- **Does NOT provide**: Atomicity. Another client can interleave commands between your pipelined ones. Each command succeeds or fails independently.

**MULTI/EXEC transactions:**

Groups commands into an atomic block. After `MULTI`, commands are queued (not executed). `EXEC` executes them all atomically — no other client's commands can interleave.

```typescript
const multi = redis.multi();
multi.decrby('account:1:balance', 100);
multi.incrby('account:2:balance', 100);
const results = await multi.exec(); // atomic: both happen or neither
```

- **Optimizes**: Atomicity. The commands in the MULTI block execute as a single, uninterrupted sequence.
- **Also reduces**: Round trips (like pipelining — commands are sent together).

**Critical limitations vs SQL transactions:**

1. **No rollback on individual command failure**: If one command in the MULTI block fails (e.g., wrong type error), the others still execute. SQL transactions roll back everything. You must check each result in the `exec()` response.

2. **No conditional logic inside the transaction**: You can't read a value and branch on it within MULTI. All commands are queued blindly. For read-then-write patterns, use `WATCH`:

```typescript
await redis.watch('account:1:balance');
const balance = await redis.get('account:1:balance');

if (parseInt(balance) < 100) {
  await redis.unwatch();
  throw new Error('Insufficient funds');
}

// If account:1:balance changed since WATCH, EXEC returns null (transaction aborted)
const result = await redis.multi()
  .decrby('account:1:balance', 100)
  .incrby('account:2:balance', 100)
  .exec();

if (!result) {
  // Another client modified the watched key — retry
}
```

3. **WATCH is optimistic locking**: It doesn't prevent other clients from modifying the key — it just causes your transaction to fail if they do. You must implement retry logic yourself.

**When to use each:**

- **Pipelining**: Batch independent operations for throughput (bulk imports, fetching multiple unrelated keys, writing metrics).
- **MULTI/EXEC**: When operations must be atomic (transferring balances, updating related keys that must stay consistent).
- **WATCH + MULTI**: When you need read-then-write atomicity (check-and-set, conditional updates).
- **Lua scripts**: When you need conditional logic AND atomicity. A Lua script runs atomically and can contain `if/else` branching that MULTI cannot.

</details>

<details>
<summary>8. How do distributed locks work in Redis using SETNX with expiry — what are the failure modes (clock drift, GC pauses, split brain), what is the Redlock algorithm and why did Martin Kleppmann argue it's fundamentally flawed, and what's the practical takeaway for teams that need distributed coordination?</summary>

**Basic distributed lock with SETNX:**

```typescript
// Acquire: SET with NX (only if not exists) and EX (expiry)
const lockId = crypto.randomUUID(); // unique per lock holder
const acquired = await redis.set('lock:payment-process', lockId, 'EX', 10, 'NX');

if (acquired) {
  try {
    await doExclusiveWork();
  } finally {
    // Release: only delete if we still own the lock (Lua for atomicity)
    await redis.eval(
      `if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`,
      1, 'lock:payment-process', lockId
    );
  }
}
```

The expiry prevents deadlocks — if the holder crashes, the lock auto-releases after the TTL.

**Failure modes:**

1. **GC pauses / process pauses**: Client A acquires lock with 10s TTL. A long GC pause (or network delay) takes 12 seconds. The lock expires. Client B acquires the lock. A's GC finishes — it thinks it still holds the lock and proceeds. Both clients are in the "critical section" simultaneously.

2. **Clock drift**: If the lock holder's clock runs fast, it may think it has more time than it does. The Redis server's clock determines actual expiry, but the client uses its own clock to decide whether it's safe to proceed.

3. **Split brain**: In a Redis Sentinel setup, if the primary fails and Sentinel promotes a replica, the replica may not have replicated the lock key yet. Client A's lock is on the old primary. Client B acquires the same lock on the new primary. Two holders.

**The Redlock algorithm:**

Designed by Salvatore Sanfilippo (Antirez) to address single-instance failures. Uses N independent Redis instances (typically 5):

1. Get current time
2. Try to acquire the lock on all N instances with a short timeout
3. If you acquired the lock on a majority (N/2 + 1) within the validity time, the lock is held
4. Lock validity = initial TTL minus time spent acquiring

**Kleppmann's critique (2016):**

Martin Kleppmann argued Redlock is fundamentally flawed:

- **Process pauses defeat it**: Even with Redlock, a GC pause after acquiring the lock but before doing work can outlast the lock TTL. Another client acquires it. Redlock doesn't solve this — it's inherent to any lock with a timeout.
- **Clock assumptions**: Redlock assumes bounded clock drift across nodes. If clocks jump (NTP correction, VM migration), the timing assumptions break.
- **Fencing tokens are the real solution**: Instead of trying to make locks unbreakable, use a monotonically increasing fencing token. The storage system rejects writes with older tokens. This is how ZooKeeper-based locks work with `zxid`.

Antirez responded that Kleppmann's scenarios are unlikely in practice and that Redlock provides "good enough" safety for most use cases. The debate essentially boils down to: Redlock is an engineering tradeoff, not a formal proof of correctness.

**Practical takeaway:**

- **For most applications** (rate limiting, deduplication, avoiding duplicate cron runs): A single-instance Redis lock with `SET NX EX` is fine. The failure modes are rare and the consequences are usually tolerable (a duplicate email, two reports generated).
- **For financial/safety-critical mutual exclusion**: Don't use Redis locks. Use a consensus system (ZooKeeper, etcd) that provides fencing tokens. Or redesign the system to be idempotent so that concurrent execution doesn't cause harm.
- **If you must use Redlock**: Understand that it raises the bar significantly over single-instance but does NOT provide the same guarantees as a consensus protocol. It's a best-effort distributed lock.

</details>

<details>
<summary>9. How does Redis provide high availability — what does Sentinel do for automatic failover and how does it detect failures, how does Redis Cluster partition data across nodes using hash slots, and when would you choose Sentinel (replication + failover) vs Cluster (sharding + failover)?</summary>

**Redis Sentinel:**

Sentinel is a monitoring and failover system for Redis replication. You run 3+ Sentinel processes alongside your Redis primary and replicas.

**How it works:**
1. Sentinels continuously `PING` the primary and replicas.
2. If a Sentinel doesn't get a response within `down-after-milliseconds`, it marks the node as **subjectively down (SDOWN)**.
3. It asks other Sentinels if they agree. If a quorum (configurable, typically majority) agrees, the node is marked **objectively down (ODOWN)**.
4. One Sentinel is elected leader (via Raft-like election) and performs failover: promotes the best replica to primary, reconfigures other replicas to follow the new primary, and updates clients.

**Client integration**: ioredis connects to Sentinels, asks which node is the current primary, and automatically reconnects to the new primary after failover. Clients subscribe to Sentinel notifications for topology changes.

**Limitations**: All data lives on a single primary. No sharding — your dataset must fit in one node's memory.

**Redis Cluster:**

Cluster distributes data across multiple primary nodes using **hash slots**. The keyspace is divided into 16,384 slots. Each key is assigned to a slot via `CRC16(key) % 16384`, and each primary node owns a range of slots.

**How it works:**
- A 3-primary cluster might have: Node A owns slots 0-5460, Node B owns 5461-10922, Node C owns 10923-16383.
- Each primary has one or more replicas. If a primary fails, the cluster promotes its replica automatically.
- If a client sends a command to the wrong node, it gets a `MOVED` redirect to the correct node. Smart clients (like ioredis Cluster) cache the slot map to avoid redirects.

**Key constraints:**
- Multi-key operations (`MGET`, transactions) only work if all keys are on the same node. Use **hash tags** `{user:1}:profile` and `{user:1}:settings` to force keys to the same slot.
- No `SELECT` (database numbers) — only db 0.
- Cross-slot operations require application-level coordination.

**When to choose which:**

| Factor | Sentinel | Cluster |
|---|---|---|
| Data size | Fits in one node's memory | Exceeds single node memory |
| Throughput | One primary handles the write load | Need write scalability across nodes |
| Complexity | Simpler — standard replication | More complex — hash slots, redirects, hash tags |
| Multi-key operations | Full support | Only within same hash slot |
| Use case | Small-medium datasets needing HA | Large datasets or high write throughput |

**Practical guidance**: Start with Sentinel. It's simpler to operate and sufficient for most applications. Move to Cluster when you outgrow single-node memory (usually > 25-50 GB) or need higher write throughput than one node can handle. Many managed Redis services (ElastiCache, Cloud Memorystore) offer cluster mode as a checkbox — the operational complexity is largely abstracted away.

</details>

## Practical — Data Structures & Caching

<details>
<summary>10. Implement cache-aside for a Node.js service that caches database query results in Redis — show the lookup → miss → fetch → populate flow, how to handle cache invalidation when data changes (including across multiple services), and what consistency issues can arise between cache and database</summary>

**Basic cache-aside implementation:**

```typescript
import Redis from 'ioredis';
import { Pool } from 'pg';

const redis = new Redis();
const db = new Pool();

async function getProduct(productId: string): Promise<Product> {
  const cacheKey = `product:${productId}`;

  // 1. Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. Cache miss — fetch from DB
  const { rows } = await db.query('SELECT * FROM products WHERE id = $1', [productId]);
  if (!rows[0]) throw new NotFoundError(`Product ${productId} not found`);

  // 3. Populate cache with TTL
  await redis.set(cacheKey, JSON.stringify(rows[0]), 'EX', 300); // 5 min TTL

  return rows[0];
}
```

**Invalidation on write:**

```typescript
async function updateProduct(productId: string, updates: Partial<Product>): Promise<Product> {
  // 1. Update the database (source of truth)
  const { rows } = await db.query(
    'UPDATE products SET name = COALESCE($2, name), price = COALESCE($3, price) WHERE id = $1 RETURNING *',
    [productId, updates.name, updates.price]
  );

  // 2. Invalidate cache (delete, don't update — simpler and avoids race conditions)
  await redis.del(`product:${productId}`);

  return rows[0];
}
```

**Why delete instead of update the cache**: If two concurrent updates happen (Update A, Update B), the DB will serialize them correctly. But cache updates can arrive out of order — Update A might overwrite the cache after Update B, leaving stale data. Deleting is safe: the next read will fetch fresh data from the DB.

**Cross-service invalidation:**

When Service B also caches products that Service A writes:

```typescript
// Service A — after writing to DB
import { Kafka } from 'kafkajs';

async function updateProduct(productId: string, updates: Partial<Product>) {
  await db.query(/* update */);
  await redis.del(`product:${productId}`);

  // Publish event for other services
  await kafka.producer.send({
    topic: 'product-changes',
    messages: [{ key: productId, value: JSON.stringify({ event: 'product.updated', productId }) }],
  });
}

// Service B — consumer
kafka.consumer.subscribe({ topic: 'product-changes' });
kafka.consumer.run({
  eachMessage: async ({ message }) => {
    const { productId } = JSON.parse(message.value.toString());
    await redis.del(`product:${productId}`); // invalidate local cache
  },
});
```

**Consistency issues:**

1. **DB write succeeds, cache delete fails**: The cache serves stale data until TTL expires. Mitigation: short TTLs as a safety net, retry the delete, or use a background sweeper.

2. **Race between read and write**:
   - Thread A reads from DB (gets v1), is about to write to cache
   - Thread B updates DB to v2, deletes cache
   - Thread A writes v1 to cache — stale data cached until TTL

   This window is small but real. The TTL is your safety net. For critical consistency, use shorter TTLs (30-60s).

3. **Cache warming after deployment**: A new service instance starts with an empty cache. Thousands of requests simultaneously miss cache and hit DB (mini thundering herd). Mitigation: warm the cache on startup for known hot keys, or use mutex locking (as covered in question 6).

</details>

<details>
<summary>11. Implement rate limiting using Redis sorted sets — show the sliding window approach where each request adds a timestamped entry, expired entries are trimmed, and the count determines whether to allow or reject. Explain why sorted sets are better than simple counters for sliding window rate limiting</summary>

**Sliding window rate limiter with sorted sets:**

```typescript
import Redis from 'ioredis';

const redis = new Redis();

async function isRateLimited(
  userId: string,
  limit: number,
  windowMs: number
): Promise<boolean> {
  const key = `ratelimit:${userId}`;
  const now = Date.now();
  const windowStart = now - windowMs;

  // Use a pipeline for atomicity and single round trip
  const pipeline = redis.pipeline();

  // 1. Remove entries older than the window
  pipeline.zremrangebyscore(key, 0, windowStart);

  // 2. Add the current request (score = timestamp, member = unique ID)
  pipeline.zadd(key, now.toString(), `${now}:${Math.random()}`);

  // 3. Count entries in the current window
  pipeline.zcard(key);

  // 4. Set TTL so the key auto-cleans (slightly longer than window)
  pipeline.expire(key, Math.ceil(windowMs / 1000) + 1);

  const results = await pipeline.exec();
  const currentCount = results![2][1] as number;

  return currentCount > limit;
}

// Usage: 100 requests per 60 seconds
const limited = await isRateLimited('user:123', 100, 60_000);
if (limited) {
  // Return 429 Too Many Requests
}
```

**Why sorted sets beat simple counters for sliding windows:**

A **fixed window counter** (`INCR ratelimit:user:123:minute:42`) creates a new key per time bucket. The problem is boundary bursts: a user can send 100 requests at the end of minute 42 and 100 more at the start of minute 43 — 200 requests in 2 seconds while staying under the 100/minute limit in each window.

A **sorted set sliding window** tracks every individual request with its timestamp as the score. At any point, you remove entries outside the window and count what remains. There are no boundary edges — the window literally slides with the current time.

| Approach | Boundary bursts | Memory | Complexity |
|---|---|---|---|
| Fixed window counter | Allows 2x burst at boundary | O(1) per key | Simple |
| Sorted set sliding window | No boundary bursts | O(N) where N = requests in window | Moderate |

**Tradeoff**: Sorted sets use more memory (one entry per request). For high-volume endpoints (thousands of requests per user per window), this adds up. A common compromise is the **sliding window log** for strict rate limits and fixed window counters for loose ones.

**Lua script alternative for strict atomicity:**

The pipeline above is nearly atomic but not perfectly so (another client could read between commands). For strict guarantees, wrap it in a Lua script:

```typescript
const luaScript = `
  local key = KEYS[1]
  local now = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local limit = tonumber(ARGV[3])

  redis.call('zremrangebyscore', key, 0, now - window)
  redis.call('zadd', key, now, ARGV[4])
  local count = redis.call('zcard', key)
  redis.call('expire', key, math.ceil(window / 1000) + 1)

  return count > limit and 1 or 0
`;

// Pass a unique member ID from the client side — Redis Lua's math.random() is
// deterministic (seeded for replication safety) and can produce duplicate members.
const memberId = `${Date.now()}:${Math.random()}`;
const isLimited = await redis.eval(luaScript, 1, `ratelimit:${userId}`, Date.now(), windowMs, limit, memberId);
```

</details>

<details>
<summary>12. Implement a real-time leaderboard using Redis sorted sets — show how ZADD adds scores, ZRANK/ZREVRANK retrieves a player's rank, and ZRANGE/ZREVRANGE fetches top-N players. Explain how to handle ties, how to get a player's rank among millions of entries, and what the time complexity is for each operation</summary>

**Core leaderboard operations:**

```typescript
import Redis from 'ioredis';

const redis = new Redis();
const LEADERBOARD = 'leaderboard:season1';

// Add or update a player's score
await redis.zadd(LEADERBOARD, 1500, 'player:alice');
await redis.zadd(LEADERBOARD, 2300, 'player:bob');
await redis.zadd(LEADERBOARD, 1500, 'player:charlie'); // same score as alice

// Increment score atomically (e.g., player earns 100 points)
await redis.zincrby(LEADERBOARD, 100, 'player:alice'); // alice now at 1600

// Get top 10 players (highest score first) with scores
const top10 = await redis.zrevrange(LEADERBOARD, 0, 9, 'WITHSCORES');
// Returns: ['player:bob', '2300', 'player:alice', '1600', 'player:charlie', '1500']

// Get a specific player's rank (0-indexed, highest score = rank 0)
const rank = await redis.zrevrank(LEADERBOARD, 'player:alice');
// Returns: 1 (second place)

// Get a player's score
const score = await redis.zscore(LEADERBOARD, 'player:alice');

// Get players ranked 50-59 (pagination)
const page = await redis.zrevrange(LEADERBOARD, 50, 59, 'WITHSCORES');

// Get total number of players
const total = await redis.zcard(LEADERBOARD);
```

**Time complexity:**

| Operation | Complexity | Notes |
|---|---|---|
| `ZADD` / `ZINCRBY` | O(log N) | Skip list insertion/update |
| `ZREVRANK` / `ZRANK` | O(log N) | Rank lookup among millions is microseconds |
| `ZREVRANGE` (top N) | O(log N + M) | M = number of elements returned |
| `ZSCORE` | O(1) | Direct hash table lookup |
| `ZCARD` | O(1) | Stored metadata |

With 10 million players, `ZREVRANK` still returns in microseconds because skip lists provide O(log N) rank queries — log2(10M) is roughly 23 steps.

**Handling ties:**

Redis breaks ties by lexicographic order of member names. Players with the same score are ordered alphabetically, which is arbitrary and usually undesirable. Common strategies:

**1. Composite score (time-based tiebreaker):**

Encode a timestamp in the score so earlier achievements rank higher:

```typescript
// Use decimal portion for time-based tiebreaker
// Higher base score = better. For same base score, earlier time = higher composite score.
function compositeScore(baseScore: number): number {
  // Invert timestamp so earlier times produce higher decimal values
  const timeFactor = (Number.MAX_SAFE_INTEGER - Date.now()) / Number.MAX_SAFE_INTEGER;
  return baseScore + timeFactor; // e.g., 1500.9999...
}

await redis.zadd(LEADERBOARD, compositeScore(1500), 'player:alice');
```

**2. Secondary sorted set:**

Maintain a second sorted set keyed by timestamp for players at the same score. More complex but cleaner separation.

**Practical helper — get rank with "1-indexed" display:**

```typescript
async function getPlayerRank(playerId: string): Promise<{ rank: number; score: number } | null> {
  const pipeline = redis.pipeline();
  pipeline.zrevrank(LEADERBOARD, playerId);
  pipeline.zscore(LEADERBOARD, playerId);
  const results = await pipeline.exec();

  const rank = results![0][1];
  const score = results![1][1];

  if (rank === null) return null;
  return { rank: (rank as number) + 1, score: parseFloat(score as string) }; // 1-indexed for display
}
```

</details>

<details>
<summary>13. Implement session storage with automatic expiry — show how to store session data in a Redis hash with TTL, how to extend the TTL on activity, and what happens when Redis evicts sessions under memory pressure with different eviction policies</summary>

**Session storage with Redis hashes:**

```typescript
import Redis from 'ioredis';
import crypto from 'crypto';

const redis = new Redis();
const SESSION_TTL = 1800; // 30 minutes

async function createSession(userId: string, metadata: Record<string, string>): Promise<string> {
  const sessionId = crypto.randomUUID();
  const key = `session:${sessionId}`;

  // Store session as a hash — individual fields are readable/updatable
  await redis.hset(key, {
    userId,
    createdAt: Date.now().toString(),
    lastActivity: Date.now().toString(),
    ...metadata,
  });

  // Set TTL for automatic expiry
  await redis.expire(key, SESSION_TTL);

  return sessionId;
}

async function getSession(sessionId: string): Promise<Record<string, string> | null> {
  const key = `session:${sessionId}`;
  const session = await redis.hgetall(key);

  if (!session || Object.keys(session).length === 0) return null;

  // Extend TTL on activity (sliding expiry)
  const pipeline = redis.pipeline();
  pipeline.hset(key, 'lastActivity', Date.now().toString());
  pipeline.expire(key, SESSION_TTL);
  await pipeline.exec();

  return session;
}

async function updateSessionField(sessionId: string, field: string, value: string): Promise<void> {
  const key = `session:${sessionId}`;
  // HSET only updates the specified field, doesn't touch others
  await redis.hset(key, field, value);
  await redis.expire(key, SESSION_TTL); // refresh TTL
}

async function destroySession(sessionId: string): Promise<void> {
  await redis.del(`session:${sessionId}`);
}
```

**Why hashes over JSON strings**: With a hash, you can read a single field (`HGET session:abc userId`) without deserializing the entire session. You can update one field without rewriting everything. This matters when sessions carry many fields and you're reading/writing specific ones on each request.

**Eviction behavior under memory pressure:**

| Policy | Effect on sessions |
|---|---|
| `volatile-lru` | Sessions have TTL, so they're eviction candidates. Least recently accessed sessions get evicted first. **Best choice** — inactive sessions are evicted while other non-TTL data (locks, queues) stays safe. |
| `allkeys-lru` | Sessions are evicted, but so is everything else. Your distributed locks and queue metadata can also be evicted — dangerous if Redis holds mixed workloads. |
| `noeviction` | Redis returns errors on writes when full. Sessions pile up (even expired ones take time to be cleaned). Users can't log in because new sessions can't be created. |
| `volatile-ttl` | Sessions closest to expiration get evicted first. Newly created sessions survive longer, which is reasonable — but a session about to expire in 5 minutes might still be in active use. |

**Key insight**: If Redis is used solely for sessions, `allkeys-lru` is fine. If Redis is shared with caches, queues, or locks, use `volatile-lru` and ensure only disposable data (sessions, caches) has TTL set — your critical data (locks, counters) should have no TTL and won't be evicted.

</details>

<details>
<summary>14. Implement cache stampede prevention for a heavily-read cache key — show the mutex/lock approach (only one request repopulates while others wait or serve stale data), the probabilistic early expiration approach, and explain the tradeoffs between them</summary>

The stampede problem and its dangers were covered in question 6. This answer focuses on production-ready implementations.

**Mutex approach with stale data fallback:**

The version in question 6 retries in a loop, adding latency. A better production approach: serve stale data to waiters while one request refreshes.

```typescript
import Redis from 'ioredis';

const redis = new Redis();

async function getWithStaleOnLock<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  // Try fresh cache first
  const cached = await redis.get(key);
  if (cached) {
    const entry = JSON.parse(cached) as { data: T; refreshAt: number };
    if (Date.now() < entry.refreshAt) return entry.data;

    // Past refresh time — try to acquire lock for refresh
    const lockKey = `lock:${key}`;
    const acquired = await redis.set(lockKey, '1', 'EX', 30, 'NX');

    if (acquired) {
      // We hold the lock — refresh in background, serve stale now
      refreshInBackground(key, lockKey, fetchFn, ttlSeconds);
    }
    // Either way, serve the stale data immediately
    return entry.data;
  }

  // True cache miss (no stale data available) — must wait for fetch
  const lockKey = `lock:${key}`;
  const acquired = await redis.set(lockKey, '1', 'EX', 30, 'NX');

  if (acquired) {
    try {
      const fresh = await fetchFn();
      await setWithRefreshWindow(key, fresh, ttlSeconds);
      return fresh;
    } finally {
      await redis.del(lockKey);
    }
  }

  // Another request is fetching — wait briefly and retry
  await new Promise((r) => setTimeout(r, 100));
  return getWithStaleOnLock(key, fetchFn, ttlSeconds);
}

async function setWithRefreshWindow<T>(key: string, data: T, ttlSeconds: number): Promise<void> {
  const entry = {
    data,
    refreshAt: Date.now() + ttlSeconds * 800, // 80% of TTL in ms (seconds * 1000 * 0.8)
  };
  // Physical TTL is 2x the logical TTL — stale data available as fallback
  await redis.set(key, JSON.stringify(entry), 'EX', ttlSeconds * 2);
}

async function refreshInBackground<T>(
  key: string, lockKey: string, fetchFn: () => Promise<T>, ttlSeconds: number
): Promise<void> {
  try {
    const fresh = await fetchFn();
    await setWithRefreshWindow(key, fresh, ttlSeconds);
  } catch (err) {
    // Refresh failed — stale data continues to serve. Log and alert.
    console.error(`Background refresh failed for ${key}:`, err);
  } finally {
    await redis.del(lockKey);
  }
}
```

**Probabilistic early expiration (XFetch):**

```typescript
async function getWithXFetch<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttlSeconds: number,
  beta: number = 1 // higher beta = more aggressive early refresh
): Promise<T> {
  const raw = await redis.get(key);

  if (raw) {
    const entry = JSON.parse(raw) as { data: T; storedAt: number; delta: number };
    const age = (Date.now() - entry.storedAt) / 1000;
    const ttlRemaining = ttlSeconds - age;

    // XFetch formula: refresh probability increases as TTL approaches 0
    const shouldRefresh = ttlRemaining - entry.delta * beta * Math.log(Math.random()) <= 0;

    if (!shouldRefresh) return entry.data;
    // Fall through to refresh
  }

  const start = Date.now();
  const fresh = await fetchFn();
  const delta = (Date.now() - start) / 1000; // computation time in seconds

  const entry = { data: fresh, storedAt: Date.now(), delta };
  await redis.set(key, JSON.stringify(entry), 'EX', ttlSeconds);

  return fresh;
}
```

The `delta` is the time it took to compute the value. Expensive computations (high delta) trigger earlier refreshes — the logic being that if regeneration is slow, you want more lead time.

**Tradeoffs:**

| Factor | Mutex + stale | Probabilistic (XFetch) |
|---|---|---|
| Implementation complexity | Moderate — lock management, background refresh | Moderate — math, stores extra metadata |
| DB hits on expiry | Exactly 1 (lock holder) | Usually 1, occasionally 2-3 (probabilistic) |
| Latency impact | Zero for hot path (stale served immediately) | Zero for most requests, one request does the fetch |
| Cold start (no stale data) | Must wait for fetch | Must wait for fetch |
| Failure handling | Lock TTL prevents deadlock; stale data serves as fallback | No locks to get stuck; failed refresh means next request tries again |

**Recommendation**: Use the mutex + stale approach for user-facing hot keys where latency matters and you always want instant responses. Use XFetch when you want a simpler, lock-free design and can tolerate the occasional duplicate fetch.

</details>

## Practical — Messaging & Coordination

<details>
<summary>15. Implement a distributed lock using Redis for a critical section that must not run concurrently — show the SETNX with EX approach, implement proper unlock (check-then-delete with Lua or conditional SET), demonstrate what happens when the lock holder crashes before releasing, and explain why you should set the lock TTL carefully</summary>

The theory behind distributed locks and their failure modes was covered in question 8. This answer focuses on a production-ready implementation.

**Complete distributed lock implementation:**

```typescript
import Redis from 'ioredis';
import crypto from 'crypto';

const redis = new Redis();

// Lua script: only delete the lock if we still own it
const UNLOCK_SCRIPT = `
  if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
  else
    return 0
  end
`;

interface LockOptions {
  ttlMs: number;       // Lock expiry
  retryCount: number;  // Max acquisition attempts
  retryDelayMs: number; // Delay between retries
}

const DEFAULT_OPTS: LockOptions = { ttlMs: 10_000, retryCount: 5, retryDelayMs: 200 };

async function acquireLock(
  resource: string,
  opts: Partial<LockOptions> = {}
): Promise<{ lockId: string; release: () => Promise<boolean> } | null> {
  const { ttlMs, retryCount, retryDelayMs } = { ...DEFAULT_OPTS, ...opts };
  const lockKey = `lock:${resource}`;
  const lockId = crypto.randomUUID();

  for (let attempt = 0; attempt < retryCount; attempt++) {
    const acquired = await redis.set(lockKey, lockId, 'PX', ttlMs, 'NX');

    if (acquired === 'OK') {
      return {
        lockId,
        release: async () => {
          const result = await redis.eval(UNLOCK_SCRIPT, 1, lockKey, lockId);
          return result === 1;
        },
      };
    }

    // Wait before retry (with small jitter to avoid thundering herd)
    const jitter = Math.random() * retryDelayMs * 0.2;
    await new Promise((r) => setTimeout(r, retryDelayMs + jitter));
  }

  return null; // Failed to acquire after all retries
}

// Usage
async function processPayment(paymentId: string): Promise<void> {
  const lock = await acquireLock(`payment:${paymentId}`, { ttlMs: 30_000 });

  if (!lock) {
    throw new Error('Could not acquire lock — another process is handling this payment');
  }

  try {
    await doPaymentProcessing(paymentId);
  } finally {
    const released = await lock.release();
    if (!released) {
      // Lock expired before we finished — another process may have acquired it.
      // Our work may have overlapped. Log a warning for investigation.
      console.warn(`Lock for payment:${paymentId} expired before release — possible overlap`);
    }
  }
}
```

**Why the Lua unlock script matters:**

Without Lua, a naive unlock is `GET` then `DEL` — two separate commands. Between them, the lock could expire and be acquired by another client. Your `DEL` then removes THEIR lock. The Lua script is atomic: it checks ownership and deletes in a single operation that can't be interrupted.

**What happens when the lock holder crashes:**

1. Client A acquires `lock:payment:123` with 30s TTL
2. Client A crashes (process killed, network partition, OOM)
3. Client A never calls `release()`
4. After 30 seconds, Redis expires the key automatically
5. Client B can now acquire the lock and proceed

The TTL is the safety net — without it, a crash creates a permanent deadlock.

**Why TTL choice matters:**

- **Too short** (e.g., 2 seconds for a 10-second operation): The lock expires while you're still working. Another client acquires it. Two processes run concurrently — exactly what the lock was supposed to prevent. Your `release()` call then returns `false` (lock no longer yours).
- **Too long** (e.g., 5 minutes for a 2-second operation): If the holder crashes, everyone waits 5 minutes before the lock auto-releases. The system is effectively down for that resource.

**Guideline**: Set TTL to 3-5x the expected operation time. If the operation takes 5 seconds, use a 15-25 second TTL. Monitor lock `release()` returning `false` — this means your TTL is too short or your operation is slower than expected.

**Lock extension for long-running tasks:**

If you can't predict operation duration, periodically extend the lock:

```typescript
async function extendLock(resource: string, lockId: string, ttlMs: number): Promise<boolean> {
  const EXTEND_SCRIPT = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("pexpire", KEYS[1], ARGV[2])
    else
      return 0
    end
  `;
  const result = await redis.eval(EXTEND_SCRIPT, 1, `lock:${resource}`, lockId, ttlMs);
  return result === 1;
}
```

</details>

<details>
<summary>16. Implement a job queue pattern using BullMQ with Redis -- show the basic producer/consumer setup in Node.js, explain how BullMQ uses Redis data structures under the hood (lists, sorted sets for delayed jobs, streams for events), how to handle failed jobs (retry strategies, dead letter queues), and what Redis configuration matters for reliable job processing (persistence, maxmemory-policy)</summary>

**Producer/consumer setup:**

```typescript
import { Queue, Worker, Job } from 'bullmq';

const connection = { host: 'localhost', port: 6379 };

// Producer — adds jobs to the queue
const emailQueue = new Queue('email-notifications', { connection });

// Add a simple job
await emailQueue.add('welcome-email', {
  to: 'user@example.com',
  templateId: 'welcome',
});

// Add a delayed job (send in 1 hour)
await emailQueue.add('reminder-email',
  { to: 'user@example.com', templateId: 'reminder' },
  { delay: 3600_000 }
);

// Add a job with retry configuration
await emailQueue.add('transactional-email',
  { to: 'user@example.com', templateId: 'receipt', orderId: '12345' },
  {
    attempts: 5,
    backoff: { type: 'exponential', delay: 1000 }, // 1s, 2s, 4s, 8s, 16s
    removeOnComplete: { count: 1000 },  // keep last 1000 completed
    removeOnFail: { count: 5000 },      // keep last 5000 failed for inspection
  }
);

// Consumer — processes jobs
const worker = new Worker('email-notifications',
  async (job: Job) => {
    console.log(`Processing ${job.name} for ${job.data.to}`);

    const result = await sendEmail(job.data);

    // Update progress for long-running jobs
    await job.updateProgress(100);

    return result; // stored as job.returnvalue
  },
  {
    connection,
    concurrency: 5, // process 5 jobs simultaneously
  }
);

// Event listeners
worker.on('completed', (job) => console.log(`Job ${job.id} completed`));
worker.on('failed', (job, err) => console.error(`Job ${job?.id} failed: ${err.message}`));
```

**How BullMQ uses Redis data structures:**

BullMQ manages job lifecycle using several Redis structures per queue (all prefixed with `bull:<queue-name>:`):

| Redis structure | Purpose |
|---|---|
| **List** (`wait`, `active`) | `wait` list holds job IDs ready to be processed. Workers move jobs from `wait` to `active` using Lua scripts that atomically transition job state — this prevents two workers from grabbing the same job. |
| **Sorted set** (`delayed`) | Delayed jobs are stored with their execution timestamp as the score. A timer checks for jobs whose score is <= now and moves them to `wait`. |
| **Sorted set** (`prioritized`) | Jobs with priority are stored here. Lower priority number = processed first. |
| **Hash** (`<job-id>`) | Each job's data, options, progress, and result are stored as a hash. |
| **Stream** (`events`) | Job lifecycle events (completed, failed, progress) are published to a Redis stream. Used by `QueueEvents` for monitoring. |

**Failed job handling — retry strategies:**

```typescript
// Built-in backoff strategies
await queue.add('job', data, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 1000 }, // 1s, 2s, 4s, 8s, 16s
});

await queue.add('job', data, {
  attempts: 5,
  backoff: { type: 'fixed', delay: 3000 }, // 3s every time
});

// Custom backoff strategy — defined on the worker
const worker = new Worker('email-notifications',
  async (job) => { /* ... */ },
  {
    connection,
    settings: {
      backoffStrategy: (attemptsMade, type, err, job) => {
        if (err.message.includes('rate limit')) {
          return attemptsMade * 5000; // back off more for rate limits
        }
        return attemptsMade * 1000; // default linear backoff
      },
    },
  }
);
```

**Dead letter queue pattern:**

BullMQ doesn't have a built-in DLQ, but you implement it by moving permanently failed jobs to a separate queue:

```typescript
worker.on('failed', async (job, err) => {
  if (job && job.attemptsMade >= job.opts.attempts!) {
    // All retries exhausted — move to dead letter queue
    const dlq = new Queue('dead-letter', { connection });
    await dlq.add('failed-email', {
      originalJob: job.data,
      error: err.message,
      failedAt: new Date().toISOString(),
      originalQueue: 'email-notifications',
      jobId: job.id,
    });
  }
});
```

**Redis configuration for reliable job processing:**

| Setting | Required value | Why |
|---|---|---|
| `maxmemory-policy` | `noeviction` | BullMQ stores job state in Redis. If Redis evicts job hashes or list entries, jobs silently disappear — workers will fail with confusing errors. You must never lose queue metadata. |
| Persistence | AOF with `appendfsync everysec` | Without persistence, a Redis restart loses all queued jobs. AOF ensures at most 1 second of jobs are lost on crash. |
| `maxmemory` | Set appropriately | Monitor queue depth. Each job is a hash + list entry. Thousands of retained completed/failed jobs add up. Use `removeOnComplete` and `removeOnFail` to limit retention. |
| `timeout` | 0 (disabled) | Don't let Redis close idle worker connections. Workers use blocking commands (`BRPOPLPUSH`) that hold connections open. |

</details>

## Practical — Production Setup

<details>
<summary>17. Configure a managed Redis instance for a production Node.js application — show the key configuration decisions (maxmemory-policy, persistence settings, connection limits), the Node.js client setup with ioredis (connection pooling, retry strategy, error handling), and explain what breaks in production with each misconfiguration</summary>

**Redis instance configuration decisions:**

| Setting | Recommendation | What breaks if wrong |
|---|---|---|
| `maxmemory-policy` | `allkeys-lru` for pure cache; `noeviction` for queues/locks (see question 5) | Wrong policy = silent data loss or OOM errors |
| Persistence | AOF `everysec` + RDB for durable data; none for pure cache | No persistence + restart = cold cache + thundering herd on DB |
| `maxmemory` | Set to ~75% of available memory (leave room for fork overhead) | 100% = RDB/AOF fork fails due to copy-on-write memory doubling |
| `timeout` | 300 seconds for general use; 0 for worker connections using blocking commands | Too short = active connections dropped; too long = connection leaks pile up |
| `tcp-keepalive` | 60 seconds | Without keepalive, idle connections through load balancers get silently dropped |
| `maxclients` | Default 10000; ensure it exceeds your app's total connection count | Exceeded = new connections rejected, app throws errors |

**ioredis client setup for production:**

```typescript
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  tls: process.env.REDIS_TLS === 'true' ? {} : undefined,

  // Retry strategy: exponential backoff, capped at 2 seconds, give up after ~30s
  retryStrategy(times) {
    if (times > 15) {
      console.error('Redis: max retries exceeded, giving up');
      return null; // stop retrying — triggers 'error' event
    }
    return Math.min(times * 200, 2000);
  },

  // Don't queue commands forever when disconnected
  maxRetriesPerRequest: 3,

  // Reconnect on failover (Sentinel/ElastiCache promotes replica)
  reconnectOnError(err) {
    if (err.message.includes('READONLY')) {
      return 2; // reconnect AND resend failed command
    }
    return false;
  },

  // Use lazy connect for graceful startup
  lazyConnect: true,

  // Enable keepalive to detect dead connections
  keepAlive: 30_000,
});

// Error handling — without this, unhandled Redis errors crash the process
redis.on('error', (err) => {
  console.error('Redis connection error:', err.message);
  // Don't crash — ioredis will retry automatically
});

redis.on('connect', () => console.log('Redis connected'));
redis.on('reconnecting', () => console.log('Redis reconnecting...'));

// Explicit connect (because lazyConnect: true)
await redis.connect();
```

**Sentinel setup (for self-managed HA):**

```typescript
const redis = new Redis({
  sentinels: [
    { host: 'sentinel-1.internal', port: 26379 },
    { host: 'sentinel-2.internal', port: 26379 },
    { host: 'sentinel-3.internal', port: 26379 },
  ],
  name: 'mymaster', // Sentinel group name
  sentinelRetryStrategy(times) {
    return Math.min(times * 100, 3000);
  },
  reconnectOnError(err) {
    return err.message.includes('READONLY') ? 2 : false;
  },
});
```

**Connection pooling note:** ioredis doesn't use traditional connection pools — a single connection handles many concurrent commands via pipelining (Redis is single-threaded anyway). Creating a "pool" of ioredis instances only helps if you need to isolate blocking operations (`BRPOP`) from regular commands. For most apps, one client instance per application is sufficient.

**Common production misconfigurations:**

| Misconfiguration | Symptom | Fix |
|---|---|---|
| No `retryStrategy` | App crashes on first Redis hiccup | Always configure retry with backoff |
| `maxRetriesPerRequest` left at default (20) | Requests hang for ~30s during Redis outage instead of failing fast | Set to 3 for latency-sensitive paths |
| No `reconnectOnError` for READONLY | After ElastiCache failover, writes fail until manual restart | Return `2` for READONLY errors |
| No error event listener | `error` event crashes Node.js process | Always attach `redis.on('error', ...)` |
| Using `redis://` URL without TLS | Credentials sent in plaintext across the network | Use `rediss://` (TLS) in production |
| Missing `keepAlive` | Connections silently drop behind NAT/load balancer, commands timeout | Set keepAlive to 30s |

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time you implemented Redis caching for a production application — what caching strategy did you choose, how did you handle cache invalidation, and what issues did you encounter (thundering herd, stale data, memory pressure)?</summary>

**What the interviewer is looking for:** Evidence that you've made caching decisions in production, not just followed a tutorial. They want to hear about tradeoffs you considered, problems you hit, and how you solved them.

**Suggested structure (STAR-ish):**

1. **Context**: What was the application? What was the performance problem that led to caching?
2. **Strategy choice**: Which caching strategy and why? What alternatives did you consider?
3. **Implementation details**: Key design decisions — TTLs, key structure, invalidation approach.
4. **Issues encountered**: What went wrong? Stale data? Stampede? Memory growth?
5. **Resolution and lessons**: How did you fix it? What would you do differently?

**Key points to hit:**

- Name the caching strategy explicitly (cache-aside, write-through, etc.) and why it fit your use case
- Explain your invalidation approach — TTL-based, event-driven, explicit delete — and the consistency tradeoff you accepted
- Describe at least one real problem: stale data incident, memory pressure, thundering herd after a deploy, or cache key design issues
- Show you monitored the cache (hit rates, memory usage, latency) — not just "set and forget"

**Example outline to personalize:**

> Our API was hitting PostgreSQL for product catalog data on every request — p99 latency was 200ms. We added Redis cache-aside with 5-minute TTLs. Invalidation was event-driven: when the product service updated data, it published to Kafka, and consumer services deleted their cached copies.
>
> The first issue was after deployments — cold caches across all pods caused a thundering herd that spiked DB connections. We added a mutex lock on cache-miss so only one request per key hit the DB.
>
> The second issue was memory — we cached everything including rarely-accessed long-tail products. We switched to `allkeys-lfu` so hot products stayed cached while cold ones were evicted, dropping memory usage by 40%.
>
> Lesson: Start with short TTLs and monitor hit rates before optimizing. The 80/20 rule applies — a small percentage of keys serve most traffic.

</details>

<details>
<summary>19. Describe a time you debugged a Redis performance issue in production — what were the symptoms (latency spikes, connection timeouts, memory issues), how did you diagnose the root cause, and what was the fix?</summary>

**What the interviewer is looking for:** Systematic debugging methodology — not just "I restarted Redis." They want to see you correlate symptoms with root causes and use Redis-specific diagnostic tools.

**Suggested structure:**

1. **Symptoms**: What alerted you? (Latency dashboards, error rate spike, OOM alerts)
2. **Investigation**: What tools/commands did you use? (`SLOWLOG`, `INFO`, `CLIENT LIST`, `MEMORY USAGE`, monitoring dashboards)
3. **Root cause**: What was actually wrong?
4. **Fix**: What did you change? Short-term mitigation vs long-term fix.
5. **Prevention**: What monitoring or guardrails did you add?

**Key points to hit:**

- Mention specific Redis diagnostic commands: `SLOWLOG GET` (shows slow commands), `INFO memory` (memory stats), `INFO clients` (connection count), `CLIENT LIST` (what clients are doing), `MONITOR` (live command stream — use sparingly in prod)
- Show you checked the obvious first: network latency? Connection count? Memory pressure? Slow commands?
- Connect the Redis issue to application impact (p99 latency, error rates, user-facing degradation)

**Example outline to personalize:**

> We noticed p99 latency spikes to 500ms every few minutes on our API. Redis response times (measured client-side) correlated exactly with the spikes.
>
> First I checked `SLOWLOG GET 20` — several `KEYS pattern:*` commands were showing up, each taking 100-200ms. Traced them to a background job that was iterating over keys to clean up expired sessions. This blocked the entire Redis event loop during each scan.
>
> Short-term fix: Replaced `KEYS` with `SCAN` (cursor-based, non-blocking). Long-term: Switched to TTL-based expiry instead of manual cleanup — let Redis handle it.
>
> Added a `SLOWLOG` threshold alert and a dashboard tracking Redis command latency by type so we'd catch this pattern earlier.

**Common root causes to draw from:**

- `KEYS *` or `SORT` blocking the event loop
- Large key deletion (`DEL` on a sorted set with 1M members) — fix with `UNLINK`
- Connection exhaustion (too many clients, no pooling)
- Memory fragmentation after high churn — `INFO memory` shows `mem_fragmentation_ratio` > 1.5
- Lua scripts running too long
- RDB/AOF fork causing latency spike on write-heavy workloads with large datasets

</details>

<details>
<summary>20. Tell me about a time you had to decide between Redis and another technology (Kafka, PostgreSQL, Memcached) for a specific use case — what were the requirements, what drove your decision, and how did it work out?</summary>

**What the interviewer is looking for:** Engineering judgment — the ability to evaluate technologies against specific requirements rather than defaulting to what you know. They want to hear your decision framework, not just the outcome.

**Suggested structure:**

1. **Requirements**: What did the system need? (Throughput, latency, durability, ordering, complexity budget)
2. **Options considered**: What 2-3 technologies did you evaluate? What were the key differentiators?
3. **Decision criteria**: What factors mattered most? What tradeoffs did you accept?
4. **Outcome**: Did it work? What would you reconsider?

**Key points to hit:**

- Show you evaluated based on requirements, not familiarity
- Demonstrate understanding of what each technology is actually good at (not just surface-level)
- Acknowledge what you gave up with your choice — every decision has a cost
- If the decision turned out to be wrong or required adjustment, say so — that shows maturity

**Decision framework to reference:**

| Factor | Redis | Kafka | PostgreSQL | Memcached |
|---|---|---|---|---|
| Latency | Sub-millisecond | Low but higher (ms range) | ms range (depends on query) | Sub-millisecond |
| Durability | Optional (configurable) | Durable by default (replicated log) | Fully ACID | None (volatile) |
| Data structures | Rich (sorted sets, streams, hashes) | Append-only log | Full relational | Simple key-value only |
| Ordering | Per-key with streams | Partition-ordered, strong guarantees | Transaction-ordered | None |
| Throughput | 100K+ ops/sec single instance | Millions of messages/sec (distributed) | 10K-50K queries/sec (depends on complexity) | 100K+ ops/sec |
| Best for | Caching, sessions, rate limiting, real-time data | Event streaming, decoupling services, audit logs | Source of truth, complex queries, transactions | Simple caching at scale |

**Example outline to personalize:**

> We needed a job queue for processing webhook deliveries — retry on failure, delayed retries with backoff, visibility into failed jobs. Initial proposal was Kafka.
>
> I pushed back: Kafka is designed for event streaming and durable log replay, not job queue semantics. We'd have to build retry logic, delay mechanisms, and dead letter handling ourselves on top of Kafka consumers. Redis + BullMQ gave us all of that out of the box — retries, exponential backoff, delayed jobs, a dashboard for monitoring.
>
> The tradeoff: if Redis goes down, we lose queued jobs (mitigated with AOF persistence). With Kafka, durability is built in. We accepted this because webhook deliveries could be replayed from the source system if needed — perfect durability wasn't required.
>
> It worked well — simpler to operate and reason about than a Kafka-based solution would have been for this use case.

</details>
