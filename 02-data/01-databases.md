# Databases

> **26 questions** — 12 theory, 10 practical, 4 experience

- SQL vs NoSQL: relational (PostgreSQL, MySQL) vs document (MongoDB) vs key-value (DynamoDB) vs wide-column (Cassandra) — schema flexibility, consistency, query power, and when each model wins
- ACID properties: atomicity, consistency, isolation, durability — what each guarantees and how different databases implement them
- Data modeling: normalization, denormalization, and schema design for multi-tenant systems
- Index internals: B-tree, hash, GIN/GiST, LSM trees, and write-amplification tradeoffs
- Query optimization: EXPLAIN ANALYZE, N+1 queries, composite index strategy
- Transactions, isolation levels, and MVCC (PostgreSQL vs MySQL)
- Caching patterns: cache-aside, write-through, write-behind, cache invalidation strategies, read replica as cache alternative
- Replication: single-leader, multi-leader, leaderless, sync vs async, multi-region topologies
- Sharding strategies: range, hash, directory-based — shard key selection, hotspot avoidance, cross-shard queries, resharding challenges; federation (functional partitioning) as a simpler alternative
- CAP theorem: what it actually means, why "pick two" is misleading, how PostgreSQL, MongoDB, DynamoDB, and CockroachDB make different tradeoffs during partitions
- Consistency models: strong, eventual, read-your-writes, causal — practical implications for application design and user experience
- Connection pooling: application-level (pg pool, Prisma) vs external (PgBouncer), SSL/TLS connections
- Database monitoring: slow queries, connection pool saturation, replication lag, bloat tracking
- Safe schema migrations: locking behavior, adding columns/indexes on large tables, versioned migration tooling (Prisma, Flyway), rollback strategies, data vs schema migrations

---

## Foundational

<details>
<summary>1. Why would you choose a relational database (PostgreSQL, MySQL) vs a document store (MongoDB) vs a key-value store (DynamoDB) -- what are the real tradeoffs in schema flexibility, consistency guarantees, and query power? Beyond these common categories, when would you reach for a specialized database type like time-series (TimescaleDB), columnar (ClickHouse), or graph (Neo4j) instead of forcing a relational database to do the job?</summary>

**Relational (PostgreSQL, MySQL)** — Best when your data has clear relationships and you need strong consistency. Schema is enforced at the database level, which prevents bad data but requires migrations for changes. Full SQL gives you JOINs, aggregations, window functions, CTEs — the most powerful query language available. PostgreSQL adds JSONB columns for semi-structured data, giving you document-like flexibility within a relational model. Choose relational when: complex queries across entities, transactional integrity matters, or your access patterns aren't fully known yet.

**Document store (MongoDB)** — Each record is a self-contained JSON document. Schema is flexible — different documents in the same collection can have different shapes. This makes rapid iteration easy but pushes validation into your application. Good for: content management, catalogs with varied attributes, or data that's naturally hierarchical. The tradeoff: joins are expensive (done client-side or with `$lookup`), so you denormalize aggressively, which means data duplication and update anomalies. MongoDB now supports multi-document transactions, but they're slower than in relational databases.

**Key-value store (DynamoDB)** — Optimized for known access patterns at massive scale. You design your table around your queries (partition key + sort key), and in return you get single-digit millisecond latency at any scale. The tradeoff is severe: ad-hoc queries are nearly impossible, and if your access patterns change, you may need to restructure the entire table. Choose DynamoDB when: you know your access patterns upfront, need predictable performance at high throughput, and can accept limited query flexibility.

**When to reach for specialized databases:**

- **Time-series (TimescaleDB, InfluxDB)** — When you're ingesting millions of timestamped data points (metrics, IoT, financial ticks). These optimize for time-range queries, automatic partitioning by time, downsampling old data, and aggregations over time windows. A relational DB can do this but becomes painful at scale — partitioning and retention policies are manual.
- **Columnar (ClickHouse, BigQuery)** — When your workload is analytical: scanning billions of rows but only a few columns. Columnar storage compresses better and reads only the columns needed. A row-oriented DB reads entire rows even if you only need 2 of 50 columns.
- **Graph (Neo4j)** — When relationships ARE the data: social networks, fraud detection, recommendation engines, permission hierarchies. A relational DB can model graphs with join tables, but traversing 5+ levels of relationships becomes exponentially slow with JOINs, while graph databases do this in constant time per hop.

**The practical rule:** Start with PostgreSQL unless you have a specific reason not to. It handles 90% of workloads well. Reach for specialized databases when you've identified a workload that PostgreSQL handles poorly at your scale, not before.

</details>

<details>
<summary>2. How does data modeling work in relational databases -- what are the tradeoffs between normalization and denormalization, when does each approach win in practice, what are the signs that you've over-normalized or over-denormalized, and how do these choices affect query performance and maintenance as the schema evolves?</summary>

**Normalization** means eliminating data duplication by splitting data into related tables. Each fact is stored once. Third Normal Form (3NF) is the typical target: every non-key column depends on the primary key, the whole key, and nothing but the key.

**Why normalize:** Updates are simple (change a customer's address in one row, not in every order), data integrity is enforced by the schema, and storage is efficient. This is the right default for transactional (OLTP) workloads.

**Why denormalize:** Reads are faster because you avoid JOINs. When your dashboard query joins 6 tables and runs 200 times per second, pre-computing that into a single table or materialized view can drop latency from 50ms to 2ms. Denormalization trades write complexity for read performance.

**Signs you've over-normalized:**
- Simple API responses require 4+ JOINs
- You have "lookup tables" with only an ID and a name that almost never change — these could be enums or check constraints
- Every query touches the same set of JOINs and you're fighting the ORM to express them
- Read performance is degrading and adding indexes isn't enough

**Signs you've over-denormalized:**
- The same data lives in 3+ places and gets out of sync (customer name in orders, invoices, and shipments)
- Updates require touching many rows across tables — and you've had bugs where one was missed
- Schema changes cascade unpredictably because the same column exists in multiple tables
- Your data has contradictions that are hard to track down

**Practical approach:**

Start normalized (3NF) for your core transactional tables. Denormalize deliberately for specific read-heavy paths:

```sql
-- Normalized: clean writes
SELECT o.id, o.created_at, u.name, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending';

-- Denormalized: materialized view for a dashboard
CREATE MATERIALIZED VIEW order_summary AS
SELECT o.id, o.created_at, o.status, u.name AS user_name, u.email,
       COUNT(oi.id) AS item_count, SUM(oi.price * oi.quantity) AS total
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.created_at, o.status, u.name, u.email;
```

**As the schema evolves:** Normalized schemas are easier to extend — adding a new entity is just a new table with a foreign key. Denormalized schemas are harder to change because the same data is embedded in multiple places. This is why "normalize first, denormalize for performance" is the standard advice — you keep a clean source of truth and build optimized read paths on top of it.

</details>

<details>
<summary>3. How does the CAP theorem actually apply to real database choices — what do PostgreSQL, MongoDB, DynamoDB, and CockroachDB each sacrifice during a network partition, and why is the "pick two out of three" framing misleading?</summary>

The CAP theorem states that during a network partition, a distributed system must choose between **consistency** (every read returns the most recent write) and **availability** (every request gets a response). You can't have both when nodes can't communicate.

**Why "pick two" is misleading:**

Partitions aren't a choice — they happen. Networks fail. So the real decision is: "When a partition occurs, do you sacrifice consistency or availability?" When there's NO partition (which is most of the time), you can have both. The theorem only forces a tradeoff during failure. Saying "CA" makes no sense because it implies you chose to never have partitions, which isn't possible in a distributed system.

Also, the tradeoff isn't binary — real databases offer a spectrum. You can have "mostly consistent with occasional stale reads" or "available but with conflict resolution."

**How real databases make this tradeoff:**

- **PostgreSQL (single-node or primary-replica)** — CP in practice. During a partition, the primary keeps serving writes but replicas can't receive updates. If the primary goes down, you lose availability until failover completes. It prioritizes consistency — you never get stale reads from the primary.

- **MongoDB** — CP by default. Reads and writes go to the primary. If the primary is unreachable during a partition, the replica set elects a new one, but writes are unavailable during the election (typically 10-12 seconds). You can opt into reading from secondaries (`readPreference: 'secondary'`), which trades consistency for availability — reads may be stale.

- **DynamoDB** — AP by default. It's designed for availability. Writes are acknowledged as soon as a quorum of storage nodes confirm. Reads are eventually consistent by default (may return stale data), but you can request strongly consistent reads (which hit the leader node, sacrificing some availability). During a partition, DynamoDB keeps serving requests even if some replicas are unreachable.

- **CockroachDB** — CP. It uses Raft consensus, so writes require a majority of replicas to agree. During a partition, the minority side can't serve writes (unavailable) but the majority side stays consistent. It explicitly chooses consistency over availability, similar to Google Spanner.

**The practical takeaway:** Most applications need consistency for their core data path (you don't want to sell the same seat twice) and can tolerate eventual consistency for ancillary features (showing slightly stale analytics is fine). Design for this per-feature, not as a system-wide binary choice.

</details>

<details>
<summary>4. What do the ACID properties (atomicity, consistency, isolation, durability) actually guarantee, how do relational databases like PostgreSQL enforce each one, and how do NoSQL databases like MongoDB and DynamoDB make different ACID tradeoffs -- why would you choose a database that relaxes some of these guarantees?</summary>

**Atomicity** — A transaction is all-or-nothing. If any part fails, the entire transaction is rolled back. PostgreSQL implements this with a Write-Ahead Log (WAL): all changes are written to the WAL first. If the transaction aborts, the WAL entries are discarded. If the system crashes mid-transaction, recovery replays the WAL and only applies committed transactions.

**Consistency** — The database moves from one valid state to another. Constraints (foreign keys, unique constraints, CHECK constraints) are enforced at commit time. If a transaction would violate a constraint, it's rejected. Note: this is different from CAP's "consistency" — ACID consistency means data integrity rules, not read-your-writes.

**Isolation** — Concurrent transactions don't interfere with each other. PostgreSQL uses MVCC (Multi-Version Concurrency Control) — each transaction sees a snapshot of the data, so readers don't block writers and vice versa. The isolation level (Read Committed, Repeatable Read, Serializable) controls what concurrent changes are visible. Covered in more depth in question 7.

**Durability** — Once a transaction commits, it survives crashes. PostgreSQL ensures this by flushing the WAL to disk before reporting commit success (`fsync`). Even if the server loses power, committed data is recoverable from the WAL.

**NoSQL tradeoffs:**

**MongoDB** — Supports full ACID transactions within a single document (which is atomic by default) and across multiple documents since v4.0. But multi-document transactions are slower and have a 60-second default timeout. The design encourages embedding related data in a single document to avoid needing transactions. This relaxes nothing in theory but in practice pushes you toward a data model where cross-document consistency is your problem.

**DynamoDB** — Single-item operations are atomic and durable. For multi-item transactions, DynamoDB Transactions provide ACID across up to 100 items, but at 2x the cost (reads and writes are doubled). There's no isolation level concept — transactions use optimistic concurrency control with condition checks. Eventually consistent reads (the default) relax the "I" in ACID: you might read data that a concurrent transaction hasn't fully committed.

**Why relax ACID guarantees?** Performance and scalability. Full ACID across distributed nodes requires coordination (consensus protocols, distributed locks), which adds latency. If your use case tolerates stale reads (analytics dashboards, social media feeds, product catalog browsing), relaxing isolation or consistency lets you serve requests faster with simpler infrastructure. The key is understanding which parts of your system need strict ACID (financial transactions, inventory) and which don't (recommendations, activity feeds).

</details>

## Conceptual Depth

<details>
<summary>5. How do database indexes work internally — what are the structural differences between B-tree, hash, GIN/GiST, and LSM tree indexes, why does each exist for different workloads, and what is write amplification and when does it matter?</summary>

**B-tree** — The default index type in PostgreSQL and MySQL. A balanced tree where each node contains sorted keys and pointers to child nodes. Leaf nodes are linked, making range scans efficient. Supports equality (`=`), range (`>`, `<`, `BETWEEN`), prefix `LIKE 'abc%'`, and `ORDER BY`. Lookup is O(log n). This is the right choice for most columns — primary keys, foreign keys, timestamps, status fields.

**Hash index** — Maps keys through a hash function to buckets. O(1) for equality lookups, but useless for range queries (hashed values lose ordering). In PostgreSQL, hash indexes are now WAL-logged and crash-safe (they weren't before v10), but they're rarely better than B-tree in practice. Use case: exact-match lookups on long string keys where you'd save space by not storing the full key in the index.

**GIN (Generalized Inverted Index)** — Designed for values that contain multiple elements: arrays, JSONB fields, full-text search. An inverted index maps each element to the rows containing it. Example: indexing a `tags` array column — GIN maps each tag to all rows with that tag, making `@>` (contains) queries fast. Slower to update than B-tree because inserting one row may add many index entries.

**GiST (Generalized Search Tree)** — Supports spatial and geometric data: PostGIS geographic queries, range types, nearest-neighbor searches. Unlike B-tree which handles one-dimensional ordering, GiST handles multi-dimensional data with bounding box overlap operations.

**LSM tree (Log-Structured Merge-tree)** — Used by write-optimized databases like Cassandra, RocksDB, LevelDB, and DynamoDB under the hood. Instead of updating an index in-place (like B-tree), writes go to an in-memory buffer (memtable). When full, it's flushed to an immutable sorted file on disk (SSTable). Background compaction merges SSTables periodically. This makes writes extremely fast (sequential I/O) but reads slower (may need to check multiple SSTables).

**Write amplification** — The ratio of actual bytes written to disk vs. the logical bytes of your data. B-trees have moderate write amplification: inserting one row may require updating and rebalancing multiple tree nodes. LSM trees have higher write amplification during compaction — the same data may be rewritten multiple times as SSTables merge. This matters when:

- You're on SSDs with limited write endurance
- Your workload is write-heavy and storage I/O is the bottleneck
- You're paying for provisioned IOPS (cloud)

**Practical index selection:**

| Workload | Index type |
|---|---|
| Most columns (equality, range, sort) | B-tree |
| JSONB fields, array columns, full-text search | GIN |
| Geographic/spatial queries | GiST |
| Write-heavy with mostly point lookups | LSM tree (choose the right database) |

</details>

<details>
<summary>6. How do you approach query optimization systematically — how do you read an EXPLAIN ANALYZE output to identify bottlenecks, what causes N+1 queries and why are they insidious at scale, and how do you design a composite index strategy that covers your most important queries?</summary>

### Reading EXPLAIN ANALYZE

`EXPLAIN` shows the query plan. `EXPLAIN ANALYZE` actually executes the query and shows real timing.

```sql
EXPLAIN ANALYZE
SELECT o.id, u.name FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending' AND o.created_at > '2024-01-01';
```

Key things to look for (read the plan bottom-up, innermost to outermost):

- **Seq Scan** — Full table scan. Fine on small tables, a red flag on large ones. Usually means a missing index on the filtered column.
- **Index Scan vs Index Only Scan** — Index Only Scan is faster because it reads only the index (no table access). Requires the index to contain all columns the query needs (a covering index).
- **Nested Loop vs Hash Join vs Merge Join** — Nested Loop is O(n*m), fine when the inner side is small or indexed. Hash Join builds a hash table from the smaller side — good for larger datasets. Merge Join needs both sides pre-sorted.
- **Rows estimated vs actual** — If the planner estimated 10 rows but got 100,000, the statistics are stale. Run `ANALYZE table_name`.
- **Sort** — `Sort Method: external merge Disk` means the sort spilled to disk because `work_mem` was too small.
- **Buffers** — `shared hit` = pages from cache, `shared read` = pages from disk. High `read` count means the working set doesn't fit in memory.

### N+1 queries

The N+1 problem: you query a list (1 query), then for each item, query related data (N queries).

```typescript
// N+1: 1 query for orders + N queries for users
const orders = await prisma.order.findMany(); // 1 query
for (const order of orders) {
  const user = await prisma.user.findUnique({  // N queries
    where: { id: order.userId }
  });
}
```

Why it's insidious: with 10 orders in development it's invisible. With 10,000 in production, that's 10,001 queries, each with network round-trip overhead. Fix covered in detail in question 15.

### Composite index strategy

Column order in a composite index matters — the index is sorted by the first column, then the second, then the third (like a phone book sorted by last name, then first name).

**Rules:**
1. **Equality columns first** — columns compared with `=` should lead
2. **Range/sort columns last** — columns used in `>`, `<`, `BETWEEN`, `ORDER BY`
3. **Most selective column first** (among equality columns) for maximum row elimination

```sql
-- Query: WHERE status = 'pending' AND created_at > '2024-01-01' ORDER BY created_at
-- Good: equality first, range/sort last
CREATE INDEX idx_orders_status_created ON orders (status, created_at);

-- Query: WHERE user_id = ? AND status = ? ORDER BY created_at DESC
CREATE INDEX idx_orders_user_status_created ON orders (user_id, status, created_at DESC);
```

**An index on (A, B, C) can satisfy queries on:**
- (A) — yes
- (A, B) — yes
- (A, B, C) — yes
- (B) — no (leftmost prefix rule)
- (A, C) — uses A for filtering, but can't skip B to use C for range scan

**Covering indexes** add columns the query SELECTs to avoid table lookups:

```sql
-- Covers: SELECT id, status, created_at FROM orders WHERE status = ? ORDER BY created_at
CREATE INDEX idx_orders_covering ON orders (status, created_at) INCLUDE (id);
```

Don't over-index — each index slows writes and uses memory. Profile your actual queries (`pg_stat_statements`), identify the top 5-10 by total time, and design indexes for those.

</details>

<details>
<summary>7. How do database transactions and isolation levels work — what does each isolation level (Read Uncommitted through Serializable) protect against, how does MVCC implement isolation differently in PostgreSQL vs MySQL InnoDB, and why does choosing the wrong isolation level cause either phantom reads or unnecessary blocking?</summary>

### Isolation levels and what they protect against

| Isolation Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible (MySQL) / Prevented (PG) |
| Serializable | Prevented | Prevented | Prevented |

- **Dirty read** — Reading uncommitted data from another transaction. If that transaction rolls back, you read data that never existed.
- **Non-repeatable read** — You read a row, another transaction modifies and commits it, you read it again and get a different value.
- **Phantom read** — You query a set of rows matching a condition, another transaction inserts a new row matching that condition, your second query returns a different set of rows.

### MVCC in PostgreSQL vs MySQL InnoDB

Both use Multi-Version Concurrency Control — readers don't block writers and writers don't block readers. Each row has multiple versions, and transactions see the version consistent with their snapshot. But the implementations differ significantly.

**PostgreSQL MVCC:**
- Every row has `xmin` (creating transaction ID) and `xmax` (deleting/updating transaction ID). Old versions live in the same table — updates create a new row version, the old one is marked dead.
- `VACUUM` is required to reclaim dead rows. This is PostgreSQL-specific overhead — if autovacuum falls behind, table bloat grows and performance degrades.
- **Read Committed** — Each statement gets a fresh snapshot. You see all data committed before your statement started.
- **Repeatable Read** — One snapshot for the entire transaction. PostgreSQL actually prevents phantom reads at this level (stricter than the SQL standard requires). If a concurrent transaction modifies rows you've read, your transaction gets a serialization error and must retry.
- **Serializable** — Uses Serializable Snapshot Isolation (SSI). Tracks read/write dependencies between transactions and aborts one if a cycle is detected. No locks on reads — it's optimistic.

**MySQL InnoDB MVCC:**
- Old row versions are stored in an **undo log** (separate from the table). The table always has the latest committed version.
- **Read Committed** — Same as PostgreSQL: each statement gets a new snapshot.
- **Repeatable Read** — Uses a snapshot from the first read, but handles phantoms differently. InnoDB uses **gap locks** and **next-key locks** to prevent inserts into ranges you've scanned. This is a locking approach, not a snapshot approach, for phantom prevention.
- **Serializable** — All `SELECT` statements implicitly become `SELECT ... LOCK IN SHARE MODE`, adding shared locks. This is a pessimistic approach — more blocking than PostgreSQL's SSI.

### Consequences of choosing wrong

**Too low (e.g., Read Committed for a financial calculation):** You run a multi-step transaction that reads balances, does calculations, and writes results. Between your reads, another transaction commits a change. Your calculation uses stale data — you get phantom reads or non-repeatable reads, causing incorrect results.

**Too high (e.g., Serializable for a read-heavy dashboard):** Every read acquires locks or goes through serialization conflict detection. Under high concurrency, you get frequent transaction aborts (PostgreSQL SSI) or lock waits (MySQL Serializable). Throughput drops, latency spikes, and the application retries flood the database further.

**Practical default:** Use Read Committed (PostgreSQL's default) for most work. Step up to Repeatable Read or Serializable only for specific transactions that need it — financial transfers, inventory reservations, anything where stale reads cause correctness issues. Always implement retry logic for serialization failures.

</details>

<details>
<summary>8. What are the main database caching patterns (cache-aside, write-through, write-behind) -- how does each work, what consistency tradeoffs does each make, why is cache invalidation considered one of the hardest problems, and when is using a read replica a better alternative to adding a caching layer?</summary>

### Cache-aside (lazy loading)

The application manages the cache explicitly. On read: check cache first, if miss, read from DB, write to cache. On write: write to DB, then invalidate (or update) the cache.

```typescript
async function getUser(id: string): Promise<User> {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.user.findUnique({ where: { id } });
  await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 3600); // 1 hour TTL
  return user;
}
```

**Tradeoffs:** Cache misses are slower (DB hit + cache write). Stale data is possible if another process updates the DB without invalidating the cache. TTL limits staleness but doesn't eliminate it. This is the most common pattern because it's simple and the application controls exactly what gets cached.

### Write-through

Every write goes to both the cache and the database synchronously. The cache is always up-to-date for data that's been written through it. Reads never get stale data for recently written items.

**Tradeoffs:** Write latency increases (two writes per operation). You cache data that may never be read — wasting memory. Works well when reads vastly outnumber writes and most written data is read soon after.

### Write-behind (write-back)

Writes go to the cache immediately, and the cache asynchronously flushes to the database in batches. The application sees fast writes because it only waits for the cache.

**Tradeoffs:** Risk of data loss if the cache crashes before flushing to the DB. Complex to implement reliably (need durability guarantees in the cache layer). Good for high-write-throughput scenarios where some data loss is acceptable (metrics, counters, activity streams).

### Why cache invalidation is hard

The fundamental problem: you have two sources of truth and need to keep them in sync.

- **Race conditions:** Two processes update the same key. Process A writes to DB, Process B writes to DB, Process B invalidates cache, Process A invalidates cache — but Process A's older value might have been re-cached between B's write and B's invalidation.
- **Distributed invalidation:** Multiple application servers share a cache. Ensuring all of them see the invalidation at the same time is non-trivial.
- **Cascading invalidation:** Updating a user's name might require invalidating the user cache, all order caches that display the user's name, the search index, etc. The dependency graph grows fast.
- **Thundering herd:** A popular cache key expires, and 1,000 requests simultaneously hit the database to rebuild it.

Mitigation strategies: short TTLs (accept some staleness), probabilistic early expiration (jitter), lock/coalesce on cache misses (only one request rebuilds), and event-driven invalidation (DB triggers or change data capture).

### When read replicas beat caching

A read replica is better when:

- Your queries are diverse and ad-hoc — caching requires knowing access patterns; a replica handles any query
- Data freshness matters — replication lag is typically seconds; cache TTL is often minutes
- You need the full SQL query power (JOINs, aggregations, filters) — caching individual keys doesn't help
- The operational complexity of a cache layer (Redis/Memcached cluster, invalidation logic) isn't worth it for the traffic level

A cache is better when: latency requirements are sub-millisecond, access patterns are predictable (same keys hit repeatedly), or the database can't handle the read load even with replicas.

</details>

<details>
<summary>9. How does database replication work and when would you choose each topology — what are the tradeoffs between single-leader, multi-leader, and leaderless replication, how do synchronous vs asynchronous replication differ in durability and latency, and what challenges arise with multi-region topologies?</summary>

### Single-leader (primary-replica)

All writes go to one leader. Replicas receive a stream of changes (WAL in PostgreSQL, binlog in MySQL) and apply them. Reads can go to any replica.

**Pros:** Simple. No write conflicts — there's one authoritative copy. Strong consistency on the leader.
**Cons:** The leader is a write bottleneck and a single point of failure (until failover). Replicas may serve stale reads (replication lag).
**Use when:** Most applications. This is the default topology.

### Multi-leader

Multiple nodes accept writes. Each leader replicates its changes to all other leaders. Used for multi-datacenter setups where you want local write latency.

**Pros:** Writes are fast in each region (no cross-region round-trip). Tolerates one datacenter going down.
**Cons:** Write conflicts. Two leaders can modify the same row concurrently, and you need a conflict resolution strategy: last-write-wins (data loss risk), merge (application logic), or conflict-free replicated data types (CRDTs). The conflict resolution complexity is significant.
**Use when:** Multi-region deployments where write latency matters and you can handle conflict resolution. Common in collaborative editing (Google Docs-style), multi-region SaaS.

### Leaderless

Any node can accept reads and writes. Writes go to multiple nodes in parallel; reads query multiple nodes and use version numbers or timestamps to determine the latest value. DynamoDB and Cassandra use this model.

**Pros:** High availability — no single leader to fail. No failover needed.
**Cons:** Eventual consistency by default. Quorum reads/writes (W + R > N) can provide strong consistency but at higher latency. Stale reads are possible if quorum isn't met. No global ordering of writes.
**Use when:** Applications that need extreme availability and can tolerate eventual consistency. High-throughput workloads with simple key-value access patterns.

### Synchronous vs asynchronous replication

**Synchronous:** The leader waits for the replica to confirm before acknowledging the write to the client. Zero data loss on failover (the replica has every committed write). But write latency increases — especially across regions. If the replica is down, writes block.

**Asynchronous:** The leader acknowledges writes immediately and streams changes to replicas in the background. Lower write latency, but on failover you can lose recent writes that hadn't been replicated yet. This is the default in most setups.

**Semi-synchronous** (common compromise): One replica is synchronous (guaranteeing at least one copy), the rest are async. PostgreSQL supports this with `synchronous_standby_names`.

### Multi-region challenges

- **Latency:** Cross-region round-trips are 50-200ms. Synchronous replication across regions makes every write pay this cost. Async replication means regions can diverge.
- **Conflict resolution:** With multi-leader across regions, concurrent writes to the same data are common and conflicts must be handled deterministically on all nodes.
- **Failover complexity:** Promoting a replica in another region changes the write endpoint. All application servers need to discover the new leader. DNS-based failover has TTL delays.
- **Regulatory constraints:** Data residency laws may require certain data to stay in specific regions, complicating what gets replicated where.
- **Split-brain:** If regions lose connectivity, both sides may accept writes, creating divergent state that's hard to reconcile.

**Practical recommendation:** Start with single-leader + async replicas in the same region. Add read replicas in other regions for read latency. Only move to multi-leader when you have a proven need for low-latency writes in multiple regions and are prepared to handle conflict resolution.

</details>

<details>
<summary>10. When should you shard a database and what are the tradeoffs of each sharding strategy — how do range-based, hash-based, and directory-based sharding work, what happens to cross-shard queries and transactions, why is sharding usually the last resort after other scaling approaches, and how does federation (functional partitioning — splitting by domain into separate databases) compare as a simpler scaling step before full sharding?</summary>

### When to shard

Shard when you've exhausted simpler scaling options and a single database can no longer handle the workload. The scaling ladder before sharding:

1. **Query optimization and indexing** — often buys 10-100x improvement
2. **Connection pooling** — handle more concurrent connections
3. **Read replicas** — offload read traffic
4. **Caching** — reduce database load for hot data
5. **Vertical scaling** — bigger machine (more CPU, RAM, faster disks)
6. **Table partitioning** — split large tables within the same database (e.g., partition orders by month)
7. **Federation** — split by domain (see below)
8. **Sharding** — horizontal partitioning across multiple database servers

Shard when: a single write node can't keep up, the dataset exceeds what fits on one machine, or you need geographic data distribution.

### Sharding strategies

**Range-based:** Assign rows to shards based on a key range (e.g., users A-M on shard 1, N-Z on shard 2; or orders from 2023 on shard 1, 2024 on shard 2).
- **Pro:** Range queries are efficient — you know which shard to hit. Good for time-series data.
- **Con:** Hotspots. If most activity is in the current date range, one shard gets all the traffic while others sit idle. Requires careful range selection and periodic rebalancing.

**Hash-based:** Apply a hash function to the shard key (e.g., `hash(user_id) % num_shards`) to distribute rows evenly.
- **Pro:** Even distribution — no hotspots if the hash function is good.
- **Con:** Range queries become scatter-gather (must hit all shards). Adding/removing shards requires rehashing and moving data (consistent hashing mitigates this).

**Directory-based:** A lookup service maps each key to its shard. The mapping is stored in a separate table or service.
- **Pro:** Maximum flexibility — move tenants between shards, handle uneven data distribution, route large tenants to dedicated shards.
- **Con:** The directory is a single point of failure and a potential bottleneck. Every query needs a lookup first. More operational complexity.

### Cross-shard problems

- **Cross-shard queries:** A query that needs data from multiple shards (e.g., "all orders across all users") must scatter to every shard and gather results. This is slow and complex. You often build denormalized views or search indexes (Elasticsearch) for cross-shard queries.
- **Cross-shard transactions:** Two-phase commit (2PC) across shards is possible but slow, fragile, and adds a coordinator as a failure point. Most sharded systems avoid cross-shard transactions by designing the shard key so related data lives together (e.g., all of a tenant's data on one shard).
- **Cross-shard JOINs:** Not supported natively. You either denormalize, do application-level joins, or redesign queries.

### Why it's a last resort

Sharding adds permanent operational complexity: schema changes must be applied to all shards, backups and monitoring multiply, debugging is harder (which shard has the problem?), and application code must be shard-aware. You can't easily un-shard. Every simpler option should be exhausted first.

### Federation (functional partitioning)

Instead of splitting the same table across servers, you split by domain: users database, orders database, products database. Each database handles its own domain independently.

**Pros:** Simpler than sharding — each database is a normal single-node setup. Failures are isolated. Teams can scale and maintain their databases independently. No shard key design or cross-shard query problems within a domain.
**Cons:** Cross-domain JOINs require application-level joins or API calls. Cross-domain transactions need saga patterns or eventual consistency. Schema coupling between services becomes an issue.

Federation is a natural fit for microservice architectures (each service owns its database) and is often the right scaling step before sharding. You only shard within a domain if that single domain outgrows one database.

</details>

<details>
<summary>11. What are the practical differences between consistency models (strong, eventual, read-your-writes, causal) -- how does each one affect what a user experiences in a distributed application, when is eventual consistency actually fine and when does it cause real bugs, and how do you choose the right consistency model for different parts of the same system?</summary>

### The models and what users experience

**Strong consistency** — Every read returns the most recent write, globally. The user never sees stale data. A user updates their profile and immediately sees the change, regardless of which server handles the next request. Cost: every read/write requires coordination between nodes, increasing latency. PostgreSQL primary reads and DynamoDB strongly consistent reads provide this.

**Eventual consistency** — After a write, replicas will converge to the same value *eventually* (typically milliseconds to seconds). Until then, different users — or the same user hitting different nodes — may see different values. DynamoDB default reads and Cassandra with low consistency levels use this.

**Read-your-writes (session consistency)** — The user who made a write always sees their own write in subsequent reads. Other users may still see stale data. This is the minimum consistency level that feels correct to an end user. Implementation: route reads after a write to the leader, use sticky sessions, or track a version/timestamp and wait for replicas to catch up.

**Causal consistency** — If operation A causally precedes operation B (A happened-before B), then everyone sees A before B. But concurrent operations (no causal relationship) can appear in any order. Example: if user A posts a message and user B replies, everyone sees the post before the reply — but two unrelated posts can appear in different orders on different nodes.

### When eventual consistency is fine

- **Product catalog browsing** — Seeing a price that's 2 seconds stale doesn't cause harm
- **Analytics dashboards** — Slightly behind real-time is acceptable
- **Social media feeds** — A post appearing a few seconds late to other users is invisible
- **Search indexes** — Full-text search being slightly behind the source of truth is normal

### When eventual consistency causes real bugs

- **Inventory/booking systems** — Two users both see 1 item in stock, both purchase it. Overselling.
- **Financial balances** — Reading a stale balance before a debit, allowing overdraft
- **Access control** — Revoking permissions but the user can still access for seconds because the replica hasn't caught up
- **Read-after-write flows** — User submits a form, gets redirected to a confirmation page that reads from a replica that hasn't received the write yet. User sees "not found" and resubmits.

### Choosing per-feature

The right approach is per-feature consistency, not system-wide:

| Feature | Consistency needed | Why |
|---|---|---|
| Account balance, payments | Strong | Incorrect reads cause real financial harm |
| User profile after edit | Read-your-writes | The editing user must see their changes; others can lag |
| Comment threads | Causal | Replies must appear after the comment they reply to |
| Product listings, search | Eventual | Staleness is harmless for a few seconds |
| Notification counts | Eventual | Approximate is fine |

**Implementation pattern:** Use the primary for writes and critical reads (account balance check before debit). Route read-heavy, stale-tolerant traffic to replicas. For read-your-writes, pass a `lastWriteTimestamp` from the write response and have the read path wait for the replica to reach that point (or fall back to the primary).

</details>

<details>
<summary>12. Why are schema migrations dangerous on production databases — what locking behavior do DDL operations (ALTER TABLE, CREATE INDEX) trigger, how do you add columns and indexes to large tables without downtime, and what strategies prevent migrations from causing outages?</summary>

### Why DDL is dangerous

Most DDL operations in PostgreSQL acquire an **ACCESS EXCLUSIVE** lock on the table, which blocks ALL other operations — reads and writes — until the DDL completes. On a 50-million-row table, an `ALTER TABLE ADD COLUMN ... DEFAULT ...` (pre-PG11) would rewrite the entire table while holding this lock. Even a few seconds of full table lock can cascade: queries queue up, connection pool fills, application timeouts trigger, and your API goes down.

### Locking behavior by operation

| Operation | Lock level | Risk |
|---|---|---|
| `CREATE INDEX` | `SHARE` lock — blocks writes, allows reads | Minutes on large tables |
| `CREATE INDEX CONCURRENTLY` | Minimal locking — allows reads and writes | Safe but takes longer, can't run inside a transaction |
| `ALTER TABLE ADD COLUMN` (nullable, no default) | `ACCESS EXCLUSIVE` — but instant | Brief lock, safe |
| `ALTER TABLE ADD COLUMN ... DEFAULT x` (PG 11+) | `ACCESS EXCLUSIVE` — but instant (default stored in catalog) | Brief lock, safe |
| `ALTER TABLE ADD COLUMN ... DEFAULT x` (pre-PG 11) | `ACCESS EXCLUSIVE` — full table rewrite | Dangerous |
| `ALTER TABLE SET NOT NULL` | `ACCESS EXCLUSIVE` — full table scan to validate | Dangerous on large tables |
| `ALTER TABLE ADD CONSTRAINT (FK)` | `ACCESS EXCLUSIVE` — validates all rows | Use `NOT VALID` + `VALIDATE CONSTRAINT` |
| `DROP COLUMN` | `ACCESS EXCLUSIVE` — but instant (marks column as dropped) | Brief lock, safe |

### Safe migration strategies

**Adding a column safely:**
```sql
-- Step 1: Add nullable column (instant, brief lock)
ALTER TABLE orders ADD COLUMN priority int;

-- Step 2: Backfill in batches (no lock)
UPDATE orders SET priority = 0 WHERE id BETWEEN 1 AND 100000;
UPDATE orders SET priority = 0 WHERE id BETWEEN 100001 AND 200000;
-- ... continue in batches

-- Step 3: Add NOT NULL constraint safely (PG 12+)
ALTER TABLE orders ADD CONSTRAINT orders_priority_not_null
  CHECK (priority IS NOT NULL) NOT VALID;
-- Then validate (only needs SHARE UPDATE EXCLUSIVE lock, allows reads/writes)
ALTER TABLE orders VALIDATE CONSTRAINT orders_priority_not_null;

-- Step 4: Optionally set column default for future inserts
ALTER TABLE orders ALTER COLUMN priority SET DEFAULT 0;
```

**Adding an index safely:**
```sql
-- CONCURRENTLY avoids locking the table for writes
CREATE INDEX CONCURRENTLY idx_orders_priority ON orders (priority);
-- Takes longer but doesn't block production traffic
-- Cannot run inside a transaction block — run it as a standalone statement
```

**Adding a foreign key safely:**
```sql
-- Add constraint without validating existing rows (instant)
ALTER TABLE orders ADD CONSTRAINT fk_orders_user
  FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

-- Validate existing rows (allows concurrent reads/writes)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_user;
```

### General strategies

- **Use migration tooling** (Prisma Migrate, Flyway, golang-migrate) for version tracking, but review the generated SQL — the tool may not use `CONCURRENTLY` or `NOT VALID` by default
- **Set lock timeouts** (`SET lock_timeout = '3s'`) so a migration fails fast instead of queuing up behind long transactions
- **Run migrations during low-traffic windows** when possible
- **Separate schema migrations from data migrations** — schema changes should be fast and reversible; data backfills are separate, batched operations
- **Deploy in phases**: add new column (nullable) → deploy code that writes to both old and new → backfill → deploy code that reads from new → drop old column

</details>

## Practical — Query Writing & Optimization

<details>
<summary>13. Given a users table, orders table, and products table — design a composite index strategy that covers the 5 most common queries (user lookup, recent orders, orders by status, product search, user's order history). Show the CREATE INDEX statements, explain the column order choices, and demonstrate when a query would vs wouldn't use each index</summary>

### Schema assumptions

```sql
-- users: id (PK), email, name, created_at
-- orders: id (PK), user_id (FK), status, total, created_at
-- products: id (PK), name, category, price, description
-- order_items: order_id (FK), product_id (FK), quantity, price
```

### The 5 indexes

```sql
-- 1. User lookup by email (exact match, unique)
CREATE UNIQUE INDEX idx_users_email ON users (email);
-- Covers: SELECT * FROM users WHERE email = 'user@example.com'
-- Column order: single column, must be email (the lookup key)

-- 2. Recent orders (all orders sorted by date, optionally filtered by status)
CREATE INDEX idx_orders_created_at ON orders (created_at DESC);
-- Covers: SELECT * FROM orders ORDER BY created_at DESC LIMIT 20
-- Why DESC: most queries want recent-first; the index stores it pre-sorted

-- 3. Orders by status with date ordering
CREATE INDEX idx_orders_status_created ON orders (status, created_at DESC);
-- Covers: SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC
-- Column order: status is equality (goes first), created_at is range/sort (goes last)

-- 4. Product search by category with price sorting
CREATE INDEX idx_products_category_price ON products (category, price);
-- Covers: SELECT * FROM products WHERE category = 'electronics' ORDER BY price ASC
-- Column order: category is equality, price is sort

-- 5. User's order history (specific user's orders, recent first)
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);
-- Covers: SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC
-- Column order: user_id is equality, created_at is sort
```

### When each index IS used

```sql
-- Uses idx_users_email (exact equality match)
SELECT * FROM users WHERE email = 'john@example.com';

-- Uses idx_orders_status_created (equality on status + sort on created_at)
SELECT * FROM orders WHERE status = 'shipped' ORDER BY created_at DESC LIMIT 50;

-- Uses idx_orders_user_created (equality on user_id + sort on created_at)
SELECT * FROM orders WHERE user_id = 42 ORDER BY created_at DESC;

-- Uses idx_products_category_price (equality on category + range on price)
SELECT * FROM products WHERE category = 'books' AND price < 30.00;
```

### When each index is NOT used

```sql
-- Does NOT use idx_orders_status_created (no status filter, just sorting by created_at)
-- The planner might use idx_orders_created_at instead, or seq scan
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20;

-- Does NOT use idx_orders_user_created (filtering by status, not user_id)
-- The leftmost prefix rule: index on (user_id, created_at) can't help with status lookups
SELECT * FROM orders WHERE status = 'pending';

-- Does NOT use idx_products_category_price (searching by name, not category)
SELECT * FROM products WHERE name ILIKE '%wireless%';
-- This needs a different approach: full-text search with GIN index or trigram index

-- Does NOT efficiently use idx_orders_status_created for aggregation across all statuses
SELECT status, COUNT(*) FROM orders GROUP BY status;
-- Might seq scan since it needs all rows anyway
```

### Practical notes

- Index 3 (`status, created_at`) makes index 2 (`created_at`) partially redundant for filtered queries, but index 2 is still useful for "all recent orders" without a status filter
- Index 5 (`user_id, created_at`) also covers simple `WHERE user_id = ?` queries (leftmost prefix)
- For product text search, add a GIN index: `CREATE INDEX idx_products_search ON products USING GIN (to_tsvector('english', name || ' ' || description))`
- Use `pg_stat_user_indexes` to verify indexes are actually being used — drop unused ones

</details>

<details>
<summary>14. Given a slow query that joins three tables and filters on multiple conditions — walk through the EXPLAIN ANALYZE output line by line, identify the bottleneck (sequential scan, nested loop join, large sort), propose a fix (index, query rewrite, or materialized view), and verify the improvement with EXPLAIN ANALYZE again</summary>

### The slow query

```sql
-- "Show me all pending orders from the last 30 days with user and item details"
SELECT u.name, u.email, o.id AS order_id, o.created_at, o.total,
       p.name AS product_name, oi.quantity
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '30 days'
ORDER BY o.created_at DESC;
```

### EXPLAIN ANALYZE output (before fix)

```
Sort  (cost=45230..45280 rows=20000 width=120) (actual time=3842.15..3843.50 rows=18500 loops=1)
  Sort Key: o.created_at DESC
  Sort Method: external merge  Disk: 4096kB
  ->  Hash Join  (cost=12500..42000 rows=20000 width=120) (actual time=1205.00..3750.00 rows=18500 loops=1)
        Hash Cond: (oi.product_id = p.id)
        ->  Hash Join  (cost=8200..35000 rows=20000 width=88) (actual time=800.00..3200.00 rows=18500 loops=1)
              Hash Cond: (o.user_id = u.id)
              ->  Nested Loop  (cost=0..28000 rows=20000 width=52) (actual time=0.50..2800.00 rows=18500 loops=1)
                    ->  Seq Scan on orders o  (cost=0..18500 rows=5000 width=36) (actual time=0.30..1500.00 rows=5000 loops=1)
                          Filter: (status = 'pending' AND created_at > ...)
                          Rows Removed by Filter: 995000
                    ->  Index Scan using order_items_order_id_idx on order_items oi  (cost=0..1.90 rows=4 width=16) (actual time=0.01..0.20 rows=3.7 loops=5000)
              ->  Hash  (cost=5000..5000 rows=100000 width=36) (actual time=600.00..600.00 rows=100000 loops=1)
                    ->  Seq Scan on users u  (cost=0..5000 rows=100000 width=36) (actual time=0.10..400.00 rows=100000 loops=1)
        ->  Hash  (cost=3000..3000 rows=50000 width=32) (actual time=300.00..300.00 rows=50000 loops=1)
              ->  Seq Scan on products p  (cost=0..3000 rows=50000 width=32) (actual time=0.10..200.00 rows=50000 loops=1)
Planning Time: 2.5ms
Execution Time: 3845.00ms
```

### Reading the plan (bottom-up, innermost first)

**Bottleneck 1 — Seq Scan on orders:** Scanning 1,000,000 rows, keeping only 5,000. The filter removes 995,000 rows. This is the biggest time sink (1500ms). Missing index on `(status, created_at)`.

**Bottleneck 2 — Seq Scan on users:** Building a hash table of all 100,000 users (600ms). The join only needs the ~4,000 distinct users who placed pending orders. A Nested Loop with index lookup on `users.id` would be faster.

**Bottleneck 3 — Sort spilling to disk:** `Sort Method: external merge Disk: 4096kB` means `work_mem` is too small to sort 18,500 rows in memory. Not the main bottleneck, but adds latency.

**The Nested Loop on order_items is fine** — it uses an index and loops only 5,000 times with ~4 rows per loop.

### The fix

```sql
-- Fix bottleneck 1: composite index for the WHERE clause + sort
CREATE INDEX CONCURRENTLY idx_orders_status_created
  ON orders (status, created_at DESC);

-- Fix bottleneck 2: ensure users.id has an index (it's the PK, so it does)
-- The planner should now switch from Hash Join to Nested Loop + Index Scan
-- because the orders side is much smaller after the index narrows it

-- Fix bottleneck 3: increase work_mem for this session (or globally if justified)
SET work_mem = '16MB';
```

### EXPLAIN ANALYZE after fix

```
Sort  (cost=850..860 rows=5000 width=120) (actual time=45.00..46.50 rows=18500 loops=1)
  Sort Key: o.created_at DESC
  Sort Method: quicksort  Memory: 2048kB
  ->  Nested Loop  (cost=0..800 rows=5000 width=120) (actual time=0.10..38.00 rows=18500 loops=1)
        ->  Nested Loop  (cost=0..600 rows=5000 width=88) (actual time=0.08..25.00 rows=18500 loops=1)
              ->  Nested Loop  (cost=0..350 rows=5000 width=52) (actual time=0.05..12.00 rows=5000 loops=1)
                    ->  Index Scan using idx_orders_status_created on orders o  (cost=0..200 rows=5000 width=36) (actual time=0.03..5.00 rows=5000 loops=1)
                          Index Cond: (status = 'pending' AND created_at > ...)
                    ->  Index Scan using order_items_order_id_idx on order_items oi  (...)
              ->  Index Scan using users_pkey on users u  (cost=0..0.50 rows=1 width=36) (actual time=0.005..0.005 rows=1 loops=5000)
        ->  Index Scan using products_pkey on products p  (cost=0..0.50 rows=1 width=32) (actual time=0.005..0.005 rows=1 loops=18500)
Planning Time: 1.8ms
Execution Time: 48.00ms
```

**Result:** 3845ms down to 48ms — an 80x improvement. The Seq Scan on orders became an Index Scan (reading 5,000 rows directly instead of scanning 1,000,000). Hash Joins became Nested Loops with index lookups (efficient since the driving set is small). Sort stayed in memory.

</details>

<details>
<summary>15. You discover your Node.js API is executing 50 SQL queries for a single endpoint that returns a list of orders with their items — show the N+1 problem in ORM code (Prisma or TypeORM), demonstrate the fix using eager loading or joins, and explain what to watch for when the fix over-fetches data</summary>

### The N+1 problem in Prisma

```typescript
// BAD: N+1 — 1 query for orders + N queries for items + N queries for products
async function getRecentOrders() {
  const orders = await prisma.order.findMany({
    where: { status: 'pending' },
    take: 50,
  });
  // 1 query executed ^^

  const result = [];
  for (const order of orders) {
    const items = await prisma.orderItem.findMany({
      where: { orderId: order.id },
    });
    // 50 queries executed (one per order) ^^

    for (const item of items) {
      item.product = await prisma.product.findUnique({
        where: { id: item.productId },
      });
      // ~150 more queries (avg 3 items per order) ^^
    }

    result.push({ ...order, items });
  }
  return result; // Total: ~201 queries for 50 orders
}
```

### Fix 1: Prisma `include` (eager loading)

```typescript
// GOOD: 3 queries total (Prisma batches each relation into a single IN query)
async function getRecentOrders() {
  return prisma.order.findMany({
    where: { status: 'pending' },
    take: 50,
    include: {
      items: {
        include: {
          product: true, // nested eager load
        },
      },
    },
  });
}
// Prisma generates:
// 1. SELECT * FROM orders WHERE status = 'pending' LIMIT 50
// 2. SELECT * FROM order_items WHERE order_id IN (1, 2, 3, ..., 50)
// 3. SELECT * FROM products WHERE id IN (101, 102, 103, ...)
```

### Fix 2: Prisma `select` (eager loading with field selection)

```typescript
// BETTER: same 3 queries but only fetches needed fields
async function getRecentOrders() {
  return prisma.order.findMany({
    where: { status: 'pending' },
    take: 50,
    select: {
      id: true,
      createdAt: true,
      total: true,
      items: {
        select: {
          quantity: true,
          price: true,
          product: {
            select: { name: true, category: true },
          },
        },
      },
    },
  });
}
```

### Fix 3: Raw SQL join (single query)

```typescript
// ALTERNATIVE: one query when you need maximum control
const orders = await prisma.$queryRaw`
  SELECT o.id, o.created_at, o.total,
         oi.quantity, oi.price AS item_price,
         p.name AS product_name
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.id
  JOIN products p ON oi.product_id = p.id
  WHERE o.status = 'pending'
  ORDER BY o.created_at DESC
  LIMIT 50
`;
// Returns flat rows — you'll need to group by order in application code
```

### Over-fetching pitfalls

**1. Loading unnecessary relations:** `include: { user: true, items: { include: { product: { include: { reviews: true } } } } }` pulls the entire object graph. Only include what the API response actually needs.

**2. Missing `select`:** Using `include` without `select` fetches ALL columns on every table. If `products` has a 10KB `description` column and you don't need it, you're transferring megabytes of unused data.

**3. Unbounded eager loads:** If an order can have 1,000 items, eagerly loading them all for a list endpoint is wasteful. Paginate the child relation or only load a count:

```typescript
// Load item count instead of all items for a list view
const orders = await prisma.order.findMany({
  where: { status: 'pending' },
  include: {
    _count: { select: { items: true } }, // just the count
  },
});
```

**4. Duplicate data in joins:** A raw JOIN returns the order data repeated for each item row. With 50 orders and 3 items each, you get 150 rows with duplicate order columns. For large payloads, Prisma's `include` approach (separate queries with IN) is often more efficient than a single large JOIN because it avoids this duplication.

</details>

<details>
<summary>16. Implement cursor-based pagination for a table with 10 million rows sorted by created_at — show the API contract, the SQL query using keyset pagination (WHERE created_at < ?), how to encode/decode the cursor, and demonstrate the performance difference vs OFFSET with EXPLAIN ANALYZE</summary>

### API contract

```typescript
// Request
GET /api/orders?limit=20&cursor=eyJjIjoiMjAyNC0wNi0xNVQxMDozMDowMFoiLCJpIjo5OTk1MH0

// Response
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJjIjoiMjAyNC0wNi0xNFQwODoxNTowMFoiLCJpIjo5OTkzMH0",  // null if no more pages
    "hasMore": true
  }
}
```

### Cursor encoding/decoding

The cursor encodes the last seen values for the sort columns. Using `created_at` alone isn't sufficient — multiple rows can share the same timestamp. Use `(created_at, id)` as a tiebreaker.

```typescript
interface Cursor {
  c: string; // created_at ISO string
  i: number; // id (tiebreaker)
}

function encodeCursor(createdAt: Date, id: number): string {
  return Buffer.from(
    JSON.stringify({ c: createdAt.toISOString(), i: id })
  ).toString('base64url');
}

function decodeCursor(cursor: string): Cursor {
  return JSON.parse(Buffer.from(cursor, 'base64url').toString());
}
```

### The SQL query

```typescript
async function getOrders(limit: number, cursor?: string) {
  const take = Math.min(limit, 100); // cap page size

  if (cursor) {
    const { c, i } = decodeCursor(cursor);
    // Keyset pagination: "give me rows after the cursor position"
    // Note: row value comparison (created_at, id) < (...) is PostgreSQL-specific syntax
    const orders = await prisma.$queryRaw<Order[]>`
      SELECT id, user_id, status, total, created_at
      FROM orders
      WHERE (created_at, id) < (${new Date(c)}::timestamptz, ${i})
      ORDER BY created_at DESC, id DESC
      LIMIT ${take + 1}
    `;
    // Fetch one extra to determine hasMore
    const hasMore = orders.length > take;
    const data = hasMore ? orders.slice(0, take) : orders;
    const last = data[data.length - 1];

    return {
      data,
      pagination: {
        nextCursor: hasMore ? encodeCursor(last.created_at, last.id) : null,
        hasMore,
      },
    };
  }

  // First page — no cursor
  const orders = await prisma.$queryRaw<Order[]>`
    SELECT id, user_id, status, total, created_at
    FROM orders
    ORDER BY created_at DESC, id DESC
    LIMIT ${take + 1}
  `;
  const hasMore = orders.length > take;
  const data = hasMore ? orders.slice(0, take) : orders;
  const last = data[data.length - 1];

  return {
    data,
    pagination: {
      nextCursor: hasMore ? encodeCursor(last.created_at, last.id) : null,
      hasMore,
    },
  };
}
```

### Required index

```sql
CREATE INDEX idx_orders_created_id ON orders (created_at DESC, id DESC);
```

### Performance: cursor vs OFFSET

```sql
-- OFFSET pagination: page 5000 (skipping 100,000 rows)
EXPLAIN ANALYZE
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
-- Limit  (actual time=850.00..850.05 rows=20)
--   ->  Sort  (actual time=700.00..800.00 rows=100020)
--         Sort Key: created_at DESC
--         ->  Seq Scan on orders  (actual time=0.01..500.00 rows=10000000)
-- Execution Time: 850ms
-- Problem: DB must sort and skip 100,000 rows to return 20. Gets slower with every page.

-- Cursor pagination: same logical position
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE (created_at, id) < ('2024-06-15T10:30:00Z', 99950)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Limit  (actual time=0.05..0.08 rows=20)
--   ->  Index Scan using idx_orders_created_id on orders  (actual time=0.04..0.07 rows=20)
--         Index Cond: (created_at, id) < (...)
-- Execution Time: 0.1ms
-- The index jumps directly to the cursor position. Constant time regardless of page depth.
```

| Page depth | OFFSET time | Cursor time |
|---|---|---|
| Page 1 | ~2ms | ~0.1ms |
| Page 100 | ~20ms | ~0.1ms |
| Page 5,000 | ~850ms | ~0.1ms |
| Page 50,000 | ~8,000ms | ~0.1ms |

### Tradeoffs

- **Cursor can't jump to page N** — you can only go forward/backward from a position. For "go to page 50" UIs, you need OFFSET or a hybrid approach.
- **Row tuple comparison** `(created_at, id) < (?, ?)` requires PostgreSQL. For MySQL, use: `WHERE created_at < ? OR (created_at = ? AND id < ?)`
- **Deletes/inserts between pages** — cursor pagination handles this gracefully (no skipped or duplicated rows), while OFFSET can skip rows when data changes between page fetches.

</details>

<details>
<summary>17. A financial application needs to transfer funds between two accounts — implement the transaction with the correct isolation level, show how you handle concurrent transfers to the same account, explain why you chose that isolation level over others, and what happens if you use a weaker one</summary>

### Implementation with Serializable isolation

```typescript
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function transferFunds(
  fromAccountId: string,
  toAccountId: string,
  amount: number
): Promise<void> {
  const MAX_RETRIES = 3;

  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');

      // Read both balances — Serializable tracks these reads
      const { rows: [from] } = await client.query(
        'SELECT balance FROM accounts WHERE id = $1',
        [fromAccountId]
      );
      const { rows: [to] } = await client.query(
        'SELECT balance FROM accounts WHERE id = $1',
        [toAccountId]
      );

      if (!from || !to) throw new Error('Account not found');
      if (from.balance < amount) throw new Error('Insufficient funds');

      // Debit and credit
      await client.query(
        'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
        [amount, fromAccountId]
      );
      await client.query(
        'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
        [amount, toAccountId]
      );

      // Record the transfer
      await client.query(
        `INSERT INTO transfers (from_account, to_account, amount, created_at)
         VALUES ($1, $2, $3, NOW())`,
        [fromAccountId, toAccountId, amount]
      );

      await client.query('COMMIT');
      return; // success — exit retry loop

    } catch (err: any) {
      await client.query('ROLLBACK');

      // PostgreSQL serialization failure error code
      if (err.code === '40001' && attempt < MAX_RETRIES) {
        // Retry with exponential backoff
        await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 50));
        continue;
      }
      throw err;
    } finally {
      client.release();
    }
  }
}
```

### Why Serializable

**Serializable** guarantees that concurrent transactions produce the same result as if they ran one at a time. PostgreSQL implements this with Serializable Snapshot Isolation (SSI) — it tracks read/write dependencies and aborts one transaction if a cycle is detected. No explicit row locks needed.

### Alternative: SELECT FOR UPDATE (pessimistic locking at Read Committed)

```typescript
await client.query('BEGIN'); // default Read Committed

// Lock both accounts — consistent ordering prevents deadlocks
const [first, second] = fromAccountId < toAccountId
  ? [fromAccountId, toAccountId]
  : [toAccountId, fromAccountId];

await client.query('SELECT * FROM accounts WHERE id = $1 FOR UPDATE', [first]);
await client.query('SELECT * FROM accounts WHERE id = $1 FOR UPDATE', [second]);

// Now proceed with balance check and updates...
await client.query('COMMIT');
```

`FOR UPDATE` explicitly locks the rows. Other transactions trying to lock the same rows will wait (not fail), so no retry loop needed. The tradeoff: blocking under contention vs. retries under contention.

### Why not weaker isolation levels?

**Read Committed (default) without explicit locks:**

```
Time    Transaction A (transfer $100)         Transaction B (transfer $100)
T1      BEGIN                                 BEGIN
T2      SELECT balance FROM accounts          SELECT balance FROM accounts
        WHERE id = 'X' → $500                WHERE id = 'X' → $500
T3      UPDATE SET balance = 500 - 100        UPDATE SET balance = 500 - 100
T4      COMMIT (balance = $400)               COMMIT (balance = $400)
                                              ^^ Lost update! Should be $300
```

Both transactions read $500 and subtract $100 independently. The second commit overwrites the first. $100 vanishes. This is a **lost update** — Read Committed allows it because each statement sees the latest committed data, but the transaction as a whole doesn't see the other transaction's write.

**Repeatable Read:** In PostgreSQL, Repeatable Read would detect this — Transaction B would get a serialization error at commit time because the row it read was modified. But it still requires retry logic, and it doesn't protect against all anomalies that Serializable does (like write skew).

### Practical recommendation

- For financial transfers: **Serializable + retry logic** (simplest correct approach, no explicit locks) or **Read Committed + SELECT FOR UPDATE** (pessimistic, no retries but potential blocking)
- Always order lock acquisition consistently (e.g., by account ID) to prevent deadlocks
- Log every transfer attempt (including retries) for audit trails

</details>

## Practical — Schema & Data Access

<details>
<summary>18. Design a multi-tenant database schema for a SaaS application — compare schema-per-tenant vs shared-schema with tenant_id column vs row-level security, show the SQL for your chosen approach, and explain the tradeoffs in isolation, performance, and operational complexity as tenant count grows</summary>

### Three approaches compared

| | Schema-per-tenant | Shared schema + tenant_id | Shared schema + RLS |
|---|---|---|---|
| **Isolation** | Strong — separate schemas, no cross-contamination possible | Weak — one bad WHERE clause leaks data | Strong — DB enforces isolation |
| **Performance** | Queries don't scan other tenants' data | Large tables with all tenants; indexes must include tenant_id | Same as shared, RLS adds small overhead |
| **Migrations** | Must apply to every schema (100 tenants = 100 migrations) | One migration | One migration |
| **Connection pooling** | Need separate pools or schema switching | One pool | One pool with session variables |
| **Tenant count scale** | Breaks down at ~100-500 tenants (too many schemas) | Scales to millions of tenants | Scales to millions of tenants |
| **Per-tenant customization** | Easy (different schema versions possible) | Hard | Hard |
| **Operational complexity** | High — backups, monitoring per schema | Low | Low-medium (RLS policies to manage) |

### Recommended: Shared schema + tenant_id + RLS

For most SaaS applications, shared schema with RLS gives the best balance: low operational overhead with database-enforced isolation.

```sql
-- Schema
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  email TEXT NOT NULL,
  name TEXT NOT NULL,
  UNIQUE (tenant_id, email)  -- email unique within tenant, not globally
);

CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  user_id UUID NOT NULL REFERENCES users(id),
  status TEXT NOT NULL DEFAULT 'pending',
  total NUMERIC(10, 2) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes must include tenant_id as the leading column
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status, created_at DESC);
CREATE INDEX idx_users_tenant_email ON users (tenant_id, email);
```

### Row-Level Security (RLS)

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: rows are visible only when tenant_id matches the session variable
CREATE POLICY tenant_isolation ON users
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Force RLS even for table owners (important for security)
ALTER TABLE users FORCE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

### Application integration

```typescript
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Middleware: set tenant context on every request
async function withTenant<T>(
  tenantId: string,
  fn: (client: PoolClient) => Promise<T>
): Promise<T> {
  const client = await pool.connect();
  try {
    // Set the session variable that RLS policies check
    await client.query("SET app.current_tenant_id = $1", [tenantId]);

    return await fn(client);
  } finally {
    // Reset to prevent tenant leakage on connection reuse
    await client.query("RESET app.current_tenant_id");
    client.release();
  }
}

// Usage — RLS automatically filters to the tenant's data
app.get('/orders', async (req, res) => {
  const orders = await withTenant(req.tenantId, async (client) => {
    // No WHERE tenant_id = ? needed — RLS handles it
    const result = await client.query(
      "SELECT * FROM orders WHERE status = $1 ORDER BY created_at DESC",
      ['pending']
    );
    return result.rows;
  });
  res.json(orders);
});
```

### Why this approach

- **Defense in depth:** Even if application code forgets `WHERE tenant_id = ?`, the database prevents cross-tenant data leakage. The application code literally cannot read another tenant's data.
- **Single pool:** No per-tenant connection management. The session variable switches context.
- **Single migration path:** Schema changes apply once, to one set of tables.
- **Performance:** With `tenant_id` as the leading column in all indexes, queries effectively partition the data. PostgreSQL's query planner combines the RLS filter with your query filters efficiently.

### When to choose schema-per-tenant instead

- Regulatory requirements mandate physical data separation (healthcare, finance)
- Tenants need different schema versions (enterprise customers with custom fields)
- You have fewer than ~100 tenants and each is large enough to justify dedicated resources
- You need per-tenant backup/restore capability

</details>

<details>
<summary>19. You need to add a non-null column with a default value and a new index to a table with 50 million rows in production — walk through the migration steps that avoid locking the table, show the SQL commands, explain why a naive ALTER TABLE would cause downtime, and what tools or techniques help (CREATE INDEX CONCURRENTLY, backfill strategies)</summary>

Building on the safe migration patterns from question 12 (which covers DDL locking behavior for each operation type), here's the full end-to-end sequence for this specific scenario.

### Why the naive approach causes downtime

```sql
-- DON'T DO THIS on a large table
ALTER TABLE orders ADD COLUMN priority INT NOT NULL DEFAULT 0;
CREATE INDEX idx_orders_priority ON orders (priority);
```

On PostgreSQL < 11, `ADD COLUMN ... DEFAULT` rewrites the entire table (50M rows) while holding an **ACCESS EXCLUSIVE** lock — blocking ALL reads and writes. On PG 11+, `ADD COLUMN type NOT NULL DEFAULT value` as a single statement is instant — the default is stored in the catalog and PostgreSQL knows all existing rows will have that value, so no NULL scan is needed. However, if you add a nullable column first and later try to add `NOT NULL` as a separate `ALTER COLUMN SET NOT NULL`, PostgreSQL must scan every row to verify no NULLs exist, holding an ACCESS EXCLUSIVE lock for the duration. The safe multi-step approach below is still preferred because the naive single-statement `CREATE INDEX` that follows it causes locking issues regardless.

`CREATE INDEX` (without `CONCURRENTLY`) takes a **SHARE** lock — blocks writes for the entire build duration (minutes on 50M rows).

### Safe migration steps

**Step 1: Add the column as nullable (instant, brief lock)**

```sql
-- PG 11+: instant, stores default in catalog metadata
-- Lock: ACCESS EXCLUSIVE but only for catalog update (milliseconds)
ALTER TABLE orders ADD COLUMN priority INT DEFAULT 0;
-- Note: no NOT NULL constraint yet
```

**Step 2: Deploy application code that writes the new column**

Update your application to write `priority` on all new inserts and updates. This prevents new NULL rows from appearing during the backfill window.

**Step 3: Backfill existing rows in batches**

```sql
-- Backfill in small batches to avoid long-running transactions and lock contention
-- Use a script that loops through ranges

-- Option A: batch by ID range
UPDATE orders SET priority = 0
WHERE id BETWEEN 1 AND 100000 AND priority IS NULL;

UPDATE orders SET priority = 0
WHERE id BETWEEN 100001 AND 200000 AND priority IS NULL;
-- ... continue in 100K batches with short pauses between

-- Option B: batch with a CTE (more elegant)
WITH batch AS (
  SELECT id FROM orders
  WHERE priority IS NULL
  LIMIT 10000
  FOR UPDATE SKIP LOCKED  -- skip rows locked by other transactions
)
UPDATE orders SET priority = 0
WHERE id IN (SELECT id FROM batch);
-- Run in a loop until 0 rows affected
```

```typescript
// Backfill script in Node.js
async function backfillPriority(pool: Pool) {
  let updated = 0;
  do {
    const result = await pool.query(`
      WITH batch AS (
        SELECT id FROM orders WHERE priority IS NULL LIMIT 10000
      )
      UPDATE orders SET priority = 0 WHERE id IN (SELECT id FROM batch)
    `);
    updated = result.rowCount ?? 0;
    console.log(`Updated ${updated} rows`);
    // Brief pause to let normal traffic through
    await new Promise(r => setTimeout(r, 100));
  } while (updated > 0);
}
```

**Step 4: Add NOT NULL constraint safely**

```sql
-- Add as NOT VALID first (instant, doesn't scan existing rows)
ALTER TABLE orders ADD CONSTRAINT orders_priority_not_null
  CHECK (priority IS NOT NULL) NOT VALID;

-- Validate existing rows (takes SHARE UPDATE EXCLUSIVE lock — allows reads AND writes)
ALTER TABLE orders VALIDATE CONSTRAINT orders_priority_not_null;
```

This is safer than `ALTER TABLE orders ALTER COLUMN priority SET NOT NULL` which scans the whole table under ACCESS EXCLUSIVE lock. The CHECK constraint with NOT VALID + VALIDATE achieves the same logical result with less disruptive locking.

**Step 5: Create the index concurrently**

```sql
-- CONCURRENTLY: no write lock, builds in background
-- Cannot run inside a transaction block
CREATE INDEX CONCURRENTLY idx_orders_priority ON orders (priority);
```

Takes longer than a regular `CREATE INDEX` (scans the table twice) but allows normal reads and writes throughout. If it fails partway, it leaves an INVALID index that you must drop and retry.

**Step 6: Verify**

```sql
-- Check for any remaining NULLs
SELECT COUNT(*) FROM orders WHERE priority IS NULL; -- should be 0

-- Verify index is valid (not left in INVALID state from a failed CONCURRENTLY)
SELECT indexrelid::regclass, indisvalid FROM pg_index
WHERE indexrelid = 'idx_orders_priority'::regclass;
```

### Summary of locks at each step

| Step | Lock | Duration | Blocks writes? |
|---|---|---|---|
| ADD COLUMN (nullable, with default, PG 11+) | ACCESS EXCLUSIVE | Milliseconds | Briefly |
| Backfill batches | ROW locks per batch | Seconds per batch | Only locked rows |
| ADD CHECK NOT VALID | ACCESS EXCLUSIVE | Milliseconds | Briefly |
| VALIDATE CONSTRAINT | SHARE UPDATE EXCLUSIVE | Minutes (scanning) | No |
| CREATE INDEX CONCURRENTLY | Minimal | Minutes | No |

</details>

## Practical — Production Operations

<details>
<summary>20. Your API response times have degraded from 50ms to 2 seconds over the past week — walk through the systematic database diagnosis: checking pg_stat_activity for blocked queries, identifying the slow query with pg_stat_statements, running EXPLAIN ANALYZE, checking for missing indexes, bloated tables, or lock contention, and applying the fix</summary>

### Step 1: Check for active problems — pg_stat_activity

```sql
-- What's happening RIGHT NOW? Look for blocked or long-running queries
SELECT pid, state, wait_event_type, wait_event,
       now() - query_start AS duration,
       left(query, 100) AS query_preview
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

What to look for:
- **`state = 'active'`** with long duration — long-running queries consuming resources
- **`wait_event_type = 'Lock'`** — queries blocked waiting for locks. Check what's holding the lock:

```sql
-- Find blocking chains
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks kl ON kl.locktype = bl.locktype
  AND kl.database IS NOT DISTINCT FROM bl.database
  AND kl.relation IS NOT DISTINCT FROM bl.relation
  AND kl.page IS NOT DISTINCT FROM bl.page
  AND kl.tuple IS NOT DISTINCT FROM bl.tuple
  AND kl.pid != bl.pid
  AND kl.granted
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid
WHERE NOT bl.granted;
```

- **Many `idle in transaction`** connections — someone opened a transaction and never committed. This holds locks and prevents VACUUM.

### Step 2: Find the slow query — pg_stat_statements

```sql
-- Top queries by total time (the ones consuming the most cumulative database time)
SELECT queryid,
       left(query, 80) AS query_preview,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(max_exec_time::numeric, 2) AS max_ms,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

If the degradation is gradual over a week, look for:
- A query whose `mean_exec_time` has increased (same query, slower execution)
- A query whose `calls` count is disproportionately high (N+1 pattern appearing)

### Step 3: Diagnose the slow query — EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ... -- paste the slow query from pg_stat_statements
```

Look for (as covered in question 14):
- **Seq Scan** on a large table — missing index
- **Rows estimated vs actual** wildly different — stale statistics, run `ANALYZE tablename`
- **Buffers: shared read** much higher than `shared hit` — working set doesn't fit in memory
- **Sort Method: external merge Disk** — `work_mem` too small

### Step 4: Check for missing indexes

```sql
-- Tables with the most sequential scans (candidates for missing indexes)
SELECT relname, seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 1000
ORDER BY seq_tup_read DESC
LIMIT 10;
-- High seq_scan with low idx_scan = missing index on commonly filtered columns

-- Unused indexes (candidates for removal — they slow writes for no benefit)
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Step 5: Check for table bloat

```sql
-- Estimate dead tuple ratio (bloat indicator)
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(n_dead_tup::numeric / GREATEST(n_live_tup, 1) * 100, 2) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

If `dead_pct` is high (>20%) and `last_autovacuum` is old:
- Autovacuum may be falling behind — check `autovacuum_max_workers`, `autovacuum_vacuum_cost_delay`
- A long-running transaction may be preventing VACUUM from reclaiming rows (MVCC — old transactions hold row visibility)
- Fix: kill the offending long transaction, then trigger manual vacuum:

```sql
-- Manual vacuum (doesn't lock the table for reads/writes)
VACUUM (VERBOSE) orders;

-- If bloat is severe, VACUUM FULL reclaims space but locks the table
-- Use pg_repack instead for zero-downtime bloat removal
```

### Step 6: Apply the fix and verify

Based on findings, the fix is typically one of:
1. **Add a missing index** (as covered in question 6 — use `CREATE INDEX CONCURRENTLY`)
2. **Fix N+1 queries** in application code (as covered in question 15)
3. **Kill long-running transactions** and tune autovacuum to prevent bloat recurrence
4. **Update statistics** with `ANALYZE` if the planner is making bad estimates
5. **Increase `work_mem`** if sorts are spilling to disk

After the fix, verify:
```sql
-- Reset pg_stat_statements to track from clean baseline
SELECT pg_stat_statements_reset();

-- After some traffic, check that the slow query is now fast
SELECT queryid, mean_exec_time, calls
FROM pg_stat_statements
WHERE queryid = <the_slow_query_id>;
```

Monitor the API p95/p99 latency to confirm the user-facing improvement.

</details>

<details>
<summary>21. How does connection pooling work at the application level (pg pool, Prisma connection pool) vs with an external pooler like PgBouncer — what are the tradeoffs between session mode, transaction mode, and statement mode, how do you size the pool correctly, and what goes wrong when you get it wrong (connection exhaustion, idle timeouts, SSL/TLS overhead)?</summary>

### How connection pooling works

A PostgreSQL connection is an OS process (~5-10 MB of memory). Opening one requires a TCP handshake, authentication, and if using SSL, a TLS handshake. This takes 20-100ms per connection. Connection pooling reuses a fixed set of open connections instead of creating/destroying them per request.

**Application-level pooling (pg pool, Prisma)**

The pool lives inside your Node.js process. You configure a `min` and `max` connection count, and the pool hands out connections from its internal queue.

```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host: 'db.example.com',
  max: 20,                    // max connections in this process
  idleTimeoutMillis: 30000,   // close idle connections after 30s
  connectionTimeoutMillis: 5000, // fail if no connection available in 5s
});

// Each query checks out a connection and returns it
const result = await pool.query('SELECT * FROM orders WHERE id = $1', [orderId]);
```

The problem: each Node.js instance has its own pool. If you have 10 pods with `max: 20`, that's up to 200 connections to PostgreSQL — and PostgreSQL's default `max_connections` is 100. Prisma has the same issue; its connection pool is per-process.

**External pooling (PgBouncer)**

PgBouncer sits between your application and PostgreSQL as a lightweight proxy. All application instances connect to PgBouncer (which handles thousands of connections cheaply), and PgBouncer maintains a much smaller pool to the actual database.

```
App (100 conns) → PgBouncer (100 → 20) → PostgreSQL (20 conns)
```

**PgBouncer modes:**

| Mode | How connections are shared | Supports | Limitations |
|---|---|---|---|
| **Session** | One PgBouncer-to-PG connection per client session (until disconnect) | Everything — prepared statements, SET, LISTEN/NOTIFY, temp tables | Minimal pooling benefit — similar to no pooler |
| **Transaction** | Connection returned to pool after each transaction commits | Most workloads | No prepared statements (across transactions), no SET that persists, no LISTEN/NOTIFY |
| **Statement** | Connection returned after each statement | Simple queries only | No multi-statement transactions at all |

**Transaction mode** is the standard choice for web applications — it gives the best pooling ratio because connections are only held during the brief transaction window, not for the entire session.

**Pool sizing:**

A good starting formula is:

```
pool_size = (number_of_CPU_cores * 2) + effective_spindle_count
```

For cloud databases on SSDs, this simplifies to roughly `cores * 2 + 1`. A 4-core PostgreSQL instance performs well with ~10-20 connections. More connections cause context switching and lock contention that actually *reduces* throughput.

For the application side:

```
app_pool_max = postgres_max_connections / number_of_app_instances
```

With PgBouncer, set `default_pool_size` in PgBouncer to match your PostgreSQL capacity, and let application instances use larger pools to PgBouncer (since PgBouncer connections are cheap).

**What goes wrong:**

- **Connection exhaustion**: All connections checked out, new requests queue and eventually timeout. Symptoms: `connection timeout` errors, request latency spikes. Cause is usually long-running queries or transactions holding connections, or pool `max` set too high (overwhelming PostgreSQL) or too low (can't handle burst traffic).
- **Idle connection bloat**: `min` set too high keeps unnecessary connections alive, wasting PostgreSQL memory. Each idle connection is ~5-10 MB on the PostgreSQL side.
- **SSL/TLS overhead**: Each new connection requires a TLS handshake (~30-50ms). Without pooling, this adds up fast. With pooling, the handshake happens once per pooled connection, amortized over thousands of requests. PgBouncer can terminate SSL from the app side and use a separate (or no) SSL connection to PostgreSQL, reducing overhead further.

</details>

<details>
<summary>22. What database metrics should you monitor in production and how do you act on them — walk through setting up alerts for slow queries (pg_stat_statements), connection pool saturation, replication lag, and table bloat, explain what each metric tells you, and what remediation steps you take when each threshold is breached?</summary>

### 1. Slow queries — pg_stat_statements

**What to track:**

```sql
-- Queries ranked by total execution time (biggest cumulative cost)
SELECT queryid, left(query, 80),
       calls, mean_exec_time, max_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;
```

**Key metrics to export (via Prometheus postgres_exporter or Datadog):**
- `mean_exec_time` per queryid — alert if a known query's average exceeds its baseline by 3x
- `calls` per queryid — sudden spike means new N+1 pattern or retry loop
- `total_exec_time` — the query consuming the most cumulative database time, even if each call is moderate

**Alert thresholds:**
- P95 query latency > 500ms → warning
- P95 query latency > 2s → critical
- Any single query > 30s → investigate (possible missing index or table lock)

**Remediation:** Run `EXPLAIN ANALYZE` on the offending query (as covered in question 20), check for missing indexes, stale statistics (`ANALYZE`), or plan regressions.

### 2. Connection pool saturation

**What to track:**

```sql
-- Current connections by state
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
-- Look for: active, idle, idle in transaction
```

**Key metrics:**
- `active_connections / max_connections` — utilization ratio
- `idle in transaction` count — connections held by uncommitted transactions
- Application pool wait queue depth (exposed by pg Pool's `waitingCount` or PgBouncer's `SHOW POOLS` → `cl_waiting`)

**Alert thresholds:**
- Connection utilization > 80% of `max_connections` → warning
- Any `idle in transaction` > 5 minutes → warning (holding locks, blocking VACUUM)
- PgBouncer `cl_waiting` > 0 sustained for 30s → pool is saturated

**Remediation:**
- Kill long `idle in transaction` sessions: `SELECT pg_terminate_backend(pid)`
- Set `idle_in_transaction_session_timeout` in PostgreSQL to auto-kill them
- If legitimately out of connections: add PgBouncer, reduce per-instance pool sizes, or scale horizontally

### 3. Replication lag

**What to track:**

```sql
-- On the primary: check how far behind each replica is
SELECT client_addr, state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

**Key metrics:**
- `replay_lag_bytes` — how far behind the replica is in applying WAL
- `replay_lag_seconds` (PostgreSQL 10+ via `pg_stat_replication` with `replay_lag` interval)

**Alert thresholds:**
- Replay lag > 1s → warning (stale reads from replicas may confuse users)
- Replay lag > 30s → critical (replica is falling behind, may indicate resource starvation on replica)
- Replication slot `active = false` → critical (if using slots, an inactive slot prevents WAL cleanup and disk fills up)

**Remediation:**
- Check replica for resource bottlenecks (CPU, I/O, long-running queries on replica blocking replay)
- If the replica is running analytical queries, set `max_standby_streaming_delay` to allow queries to delay replay briefly rather than being canceled
- If a replication slot is inactive and the replica is gone, drop the slot before the primary runs out of disk

### 4. Table bloat

**What to track:**

Use the same `pg_stat_user_tables` query from question 20 (Step 5) to check dead tuple ratios, adding `last_autoanalyze` for completeness:

```sql
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / GREATEST(n_live_tup, 1) * 100, 2) AS dead_pct,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;
```

**Key metrics:**
- `dead_pct` — ratio of dead tuples to live tuples
- `last_autovacuum` — when VACUUM last ran on this table
- Table size vs estimated live data size (use `pgstattuple` extension for accurate bloat estimate)

**Alert thresholds:**
- `dead_pct` > 20% → warning (autovacuum may be falling behind)
- `dead_pct` > 50% → critical (significant wasted space and degraded scan performance)
- `last_autovacuum` > 24 hours on a high-write table → investigate

**Remediation:**
- Check for long-running transactions preventing VACUUM from reclaiming tuples (as covered in question 20)
- Tune autovacuum to be more aggressive on hot tables: `autovacuum_vacuum_scale_factor = 0.01` (trigger at 1% dead rows instead of default 20%)
- For severe bloat where VACUUM alone isn't enough, use `pg_repack` for online table rewrite without locking

### Monitoring stack

A typical setup: **postgres_exporter** → **Prometheus** → **Grafana** dashboards with alert rules. Or a managed solution like Datadog's PostgreSQL integration, which collects all of the above automatically. The key is making these metrics visible and alerted on *before* they cause user-facing incidents.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>23. Tell me about a time you had to scale a database to handle significantly more traffic — what approach did you take (read replicas, caching, query optimization, sharding), what tradeoffs did you face, and what was the outcome?</summary>

**What the interviewer is looking for:**
- Methodical approach to diagnosing the bottleneck before jumping to solutions
- Understanding of the scaling ladder (optimize queries → add indexes → cache → read replicas → vertical scaling → sharding) and why order matters
- Awareness of tradeoffs, not just "we added more servers"
- Measurable outcomes (latency numbers, throughput improvement, cost)

**Key points to hit:**
- What triggered the scaling need (traffic growth, new feature, performance degradation)
- How you diagnosed the bottleneck (was it read-heavy? write-heavy? specific queries? connection limits?)
- Which approaches you evaluated and why you chose what you chose
- Tradeoffs you accepted (e.g., eventual consistency from read replicas, cache invalidation complexity, increased operational burden)
- What you'd do differently with hindsight

**Suggested structure (STAR):**

1. **Situation**: "Our service handled X requests/day, and after [event], traffic grew to Y. API latency degraded from Nms to Nms."
2. **Task**: "I was responsible for identifying the bottleneck and implementing a scaling solution within [timeframe]."
3. **Action**: Walk through your diagnostic steps (checking slow queries, connection pool usage, read/write ratio). Explain the solution you chose and WHY — e.g., "80% of our traffic was reads against the same dataset, so read replicas gave us the most leverage with the least application change" or "The queries themselves were inefficient — adding composite indexes and fixing N+1 patterns gave us a 10x improvement without any infrastructure changes."
4. **Result**: Concrete numbers. "P95 latency dropped from 2s to 150ms. We handled 5x the traffic on the same infrastructure."

**Example outline to personalize:**

"We had a multi-tenant service where one large tenant's query patterns were causing table scans on the orders table. I started by analyzing pg_stat_statements and found that the top query was doing a sequential scan on 50M rows because the index didn't cover the tenant's specific filter combination. I added a composite index and implemented query result caching with a 30-second TTL in Redis for the dashboard endpoint. This dropped the P99 from 4s to 200ms. The tradeoff was that dashboard data could be up to 30s stale, which we validated with the product team was acceptable."

</details>

<details>
<summary>24. Describe a time you debugged a production database issue (slow queries, connection exhaustion, replication lag, or data corruption) — what were the symptoms, how did you diagnose the root cause, and what was the fix?</summary>

**What the interviewer is looking for:**
- Systematic debugging approach under pressure (not random guessing)
- Familiarity with PostgreSQL diagnostic tools (pg_stat_activity, pg_stat_statements, EXPLAIN ANALYZE, pg_locks)
- Ability to distinguish symptoms from root causes
- Post-incident thinking: what you did to prevent recurrence

**Key points to hit:**
- How you noticed the problem (alerting, user reports, dashboard)
- Your diagnostic sequence — what you checked first and why
- The root cause and why it wasn't obvious
- The fix (both immediate mitigation and long-term prevention)
- What monitoring or process changes you made afterward

**Suggested structure (STAR):**

1. **Situation**: "We received alerts that API error rates spiked to N%. Users were seeing timeouts on [specific feature]."
2. **Task**: "As the on-call engineer, I needed to diagnose and resolve the issue."
3. **Action**: Walk through your debugging steps methodically. For example: "I checked pg_stat_activity and saw 95 of 100 connections were `idle in transaction`. I traced the PIDs back to a recently deployed endpoint that opened a transaction, made an external HTTP call, and only committed after the response — if the external service was slow or down, connections piled up." Show that you understand the *chain* from symptom to root cause.
4. **Result**: "I killed the stuck connections to restore service, then fixed the code to move the external call outside the transaction. I added `idle_in_transaction_session_timeout = 30s` as a safety net, and we added connection pool utilization to our dashboard."

**Example outline to personalize:**

"Our replication lag alert fired at 3 AM — the replica was 45 seconds behind. I checked the replica and found a long-running analytical query that was blocking WAL replay. The query was from a reporting job that had been moved to the replica but was running an unoptimized full table scan. I terminated the query to let replay catch up, then worked with the analytics team to optimize the query and schedule it during low-traffic windows. We also set `max_standby_streaming_delay` to 5 seconds so that replay would cancel long queries rather than falling dangerously behind."

</details>

<details>
<summary>25. Tell me about a time you chose between SQL and NoSQL for a project — what were the requirements, what influenced your decision, and looking back, was it the right call?</summary>

**What the interviewer is looking for:**
- Decision-making framework, not dogma ("we always use Postgres" is a red flag)
- Understanding of what each database type is actually good at
- Honesty about tradeoffs and whether the decision held up over time
- Awareness of how requirements evolve and how database choice affects future flexibility

**Key points to hit:**
- The specific requirements that drove the decision (data relationships, query patterns, scale, consistency needs, team expertise)
- What alternatives you considered and why you ruled them out
- Whether the choice was validated or challenged as the project evolved
- What you learned about making database decisions

**Suggested structure (STAR):**

1. **Situation**: Describe the project and its data characteristics. "We were building [feature/service] that needed to store [describe data shape and access patterns]."
2. **Task**: "I needed to recommend a database that would support [specific requirements — e.g., flexible schema, strong consistency, high write throughput, complex queries]."
3. **Action**: Explain your evaluation process. What factors did you weigh? Did you prototype? "I evaluated PostgreSQL and DynamoDB. The data had clear relationships between entities (orders → line items → products), and we needed ad-hoc reporting queries. DynamoDB would have required denormalizing heavily and duplicating data across multiple GSIs. PostgreSQL gave us the query flexibility we needed with JSONB for the few semi-structured fields."
4. **Result**: "The choice held up well — when product asked for a new reporting view six months later, we added it with a SQL query in an afternoon. With DynamoDB, that would have required a new GSI and a backfill migration."

**For the "was it the right call?" part**, be honest. If it was right, explain what validated it. If it was wrong, explain what you didn't anticipate and what you'd do differently. Interviewers value self-awareness over pretending every decision was perfect.

</details>

<details>
<summary>26. Describe a schema migration that went wrong or was particularly risky — what made it dangerous, how did you handle it, and what did you learn about safe migration practices?</summary>

**What the interviewer is looking for:**
- Understanding of why schema migrations are dangerous in production (locking, data loss, rollback difficulty)
- Knowledge of safe migration patterns (as covered in questions 18-19)
- Incident response skills and ability to stay calm under pressure
- Concrete lessons learned and process improvements

**Key points to hit:**
- What made the migration risky (table size, locking behavior, data transformation, no rollback path)
- What went wrong (or almost went wrong) and how you detected it
- How you mitigated the damage or prevented it
- Specific practices you adopted afterward

**Suggested structure (STAR):**

1. **Situation**: "We needed to [describe the migration — add a NOT NULL column, rename a table, change a column type, split a table] on a table with N million rows in a service handling N requests/second."
2. **Task**: "I was responsible for planning and executing the migration safely with zero downtime."
3. **Action**: Describe what happened. If it went wrong: "We ran `ALTER TABLE ADD COLUMN NOT NULL DEFAULT` on PostgreSQL 10, which rewrote the entire table and held an ACCESS EXCLUSIVE lock for 8 minutes. All writes to that table queued up, requests timed out, and the API returned 500s." If you caught it before production: "I tested the migration against a production-sized dataset in staging and discovered it would lock the table for 6 minutes. I rewrote the migration to add a nullable column first, backfill in batches, then add the NOT NULL constraint."
4. **Result**: Describe the outcome and the process changes. "After this, we adopted a migration checklist: test against production-sized data, check locking behavior, require a rollback plan, and use `CREATE INDEX CONCURRENTLY` and multi-step column additions as standard practice."

**Example outline to personalize:**

"We needed to add a `tenant_id` column to our largest table (200M rows) and make it NOT NULL with a foreign key. I broke it into 5 steps: add nullable column, backfill in 10k-row batches during off-peak hours, add a CHECK constraint with NOT VALID, validate the constraint, then create the index concurrently. The backfill took 4 hours but never blocked writes. The lesson was that any migration on a table over 1M rows needs to be multi-step — the 'obvious' single ALTER TABLE is almost never safe."

</details>
