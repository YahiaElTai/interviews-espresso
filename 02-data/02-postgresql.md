# PostgreSQL

> **28 questions** — 10 theory, 15 practical, 3 experience

- PostgreSQL vs MySQL: architectural differences (MVCC, extensibility, type system), strengths, weaknesses, and when MySQL is the better choice
- MVCC internals: tuple versioning, dead tuples, and the VACUUM dependency
- WAL (Write-Ahead Log): crash recovery, replication, and point-in-time restore
- Index types: B-tree, GIN, GiST, BRIN, partial, expression, covering (INCLUDE)
- VACUUM and autovacuum: transaction ID wraparound, tuning, and warning signs
- Locking: row-level locks, SELECT FOR UPDATE, advisory locks, deadlock detection, and pg_locks diagnostics
- Transaction isolation levels: Read Committed, Repeatable Read, Serializable
- Connection model: process-per-connection architecture, max_connections limits, application-level pool sizing, idle-in-transaction dangers
- JSONB: querying, GIN indexing, containment operators, when to use vs relational columns
- EXPLAIN ANALYZE: reading query plans, scan types, join strategies, and bottleneck identification
- Migrations: zero-downtime patterns (concurrent index creation, NOT NULL via constraints), tooling (node-pg-migrate, Prisma migrate), rollback strategies, data migrations
- Common SQL antipatterns: NOT IN with NULLs (use NOT EXISTS), BETWEEN with timestamps (double-counting at boundaries)
- Data type choices: timestamptz vs timestamp, text vs varchar(n), identity vs serial, UUID generation, money type antipattern (use integer cents or numeric)
- Table partitioning: declarative (range, list, hash), partition pruning, and maintenance challenges
- CTEs: recursive queries for hierarchical data, optimization fences, and inlining
- Window functions: ROW_NUMBER, RANK, LAG/LEAD, running totals

---

## Foundational

<details>
<summary>1. How does PostgreSQL differ from MySQL architecturally — what makes PostgreSQL's MVCC implementation, extensibility model, and type system different, when does PostgreSQL genuinely win over MySQL, and when would you pick MySQL or another database instead?</summary>

**MVCC implementation:**

PostgreSQL stores old and new row versions directly in the main table (heap). An UPDATE creates a brand new tuple and marks the old one as dead — readers see whichever version their snapshot allows. This means updates are essentially insert + mark-dead, and dead tuples accumulate until VACUUM cleans them.

MySQL (InnoDB) does MVCC differently: it updates rows in-place and writes old versions to an undo log. Undo entries are purged automatically in the background. This means MySQL doesn't suffer from table bloat the same way, but the undo log can grow large under long-running transactions.

**Extensibility:**

PostgreSQL was designed as an extensible database engine. You can create custom data types, operators, index access methods, and procedural languages. Extensions like PostGIS (geospatial), pg_trgm (fuzzy text search), and TimescaleDB (time-series) are first-class. MySQL's plugin system is more limited — you can add storage engines but not custom types or operators.

**Type system:**

PostgreSQL has a richer type system: arrays, JSONB with indexable operators, range types, composite types, enums, and network address types (inet, cidr). MySQL has JSON support but it's less mature — no containment operators, no GIN indexing on JSON.

**When PostgreSQL wins:**

- Complex queries: CTEs, window functions, lateral joins, recursive queries — PostgreSQL's query planner and SQL compliance are stronger
- Data integrity: full support for deferrable constraints, exclusion constraints, partial indexes
- JSONB workloads: when you need structured + semi-structured data in one database
- Extensions: PostGIS for geospatial, full-text search built-in

**When to pick MySQL instead:**

- Simple read-heavy workloads at high throughput — MySQL's simpler architecture and InnoDB's clustered index can be faster for primary key lookups
- Existing team expertise and tooling ecosystem
- Managed services where MySQL options are cheaper or more available
- When you don't need PostgreSQL's advanced features and want lower operational complexity (no VACUUM tuning)

</details>

<details>
<summary>2. How does PostgreSQL's MVCC (Multi-Version Concurrency Control) work under the hood — what is tuple versioning, how are dead tuples created by updates and deletes, why does this design enable readers to never block writers, and why does it create a dependency on VACUUM?</summary>

**Tuple versioning:**

Every row (tuple) in PostgreSQL has hidden system columns: `xmin` (the transaction ID that created it) and `xmax` (the transaction ID that deleted or updated it, 0 if still live). When a transaction reads a row, it checks these values against its own snapshot to determine visibility — "can I see this version?"

**How updates create dead tuples:**

An UPDATE in PostgreSQL doesn't modify the row in place. It:
1. Marks the old tuple's `xmax` with the current transaction ID (effectively "deleting" it for future transactions)
2. Inserts a brand new tuple with the updated values and a new `xmin`
3. The old tuple remains physically in the table as a "dead tuple"

A DELETE simply sets `xmax` on the existing tuple — the row stays in the heap until cleaned up.

**Why readers never block writers:**

Because multiple versions of the same row coexist physically in the table, a reader's SELECT uses its snapshot to find the version visible to it, while a writer creates a new version. They never contend for the same physical tuple. There's no read lock needed — the snapshot determines visibility, not locks.

**Why this creates a VACUUM dependency:**

Dead tuples accumulate with every UPDATE and DELETE. They consume disk space and slow down sequential scans (the database must skip over them). Without VACUUM:
- Table size grows unboundedly (table bloat)
- Index entries still point to dead tuples, degrading index performance
- Transaction IDs are 32-bit — after ~2 billion transactions, they wrap around. If old transaction IDs aren't "frozen" by VACUUM, PostgreSQL must shut down to prevent data corruption (transaction ID wraparound)

VACUUM reclaims dead tuple space, updates the visibility map (enabling index-only scans), and freezes old transaction IDs. It's not optional — it's a fundamental maintenance requirement of PostgreSQL's MVCC design.

</details>

<details>
<summary>3. What is the Write-Ahead Log (WAL) and why is it fundamental to PostgreSQL — how does WAL provide crash recovery guarantees, how does replication build on WAL, and how does WAL archiving enable point-in-time recovery?</summary>

**What WAL is:**

The Write-Ahead Log is an append-only log of every change before it's applied to the actual data files. The rule is simple: WAL records must be flushed to disk before the corresponding data page changes are written. This single rule provides durability and crash recovery.

**Crash recovery:**

When PostgreSQL starts after a crash, it reads the WAL from the last checkpoint forward and replays any changes that weren't yet written to the data files. This is why it's "write-ahead" — the log is always ahead of the data. Without WAL, a crash during a partial write to a data page would corrupt the database. With WAL, the database can always recover to a consistent state by replaying the log.

The checkpoint process periodically flushes dirty pages from shared buffers to disk and records a checkpoint in the WAL. Recovery only needs to replay from the last checkpoint, not the entire log.

**Replication via WAL:**

Streaming replication works by shipping WAL records from the primary to replicas, which replay them. The replica applies the same changes in the same order, producing an identical copy. This is physical replication — it operates at the byte level, not the SQL level.

- **Synchronous replication**: the primary waits for the replica to confirm WAL receipt before committing — zero data loss but higher latency
- **Asynchronous replication**: the primary doesn't wait — lower latency but the replica can lag, meaning some committed transactions could be lost if the primary crashes

**Point-in-time recovery (PITR):**

WAL archiving continuously copies completed WAL segments to an archive (S3, NFS, etc.). Combined with a base backup (a physical copy of the data directory), you can restore to any point in time:

1. Restore the base backup
2. Replay archived WAL segments up to the target timestamp

This is how you recover from accidental `DELETE FROM users` at 3:47 PM — restore to 3:46 PM. Without WAL archiving, your only option is restoring from the last full backup, losing all changes since then.

</details>

## Conceptual Depth

<details>
<summary>4. What PostgreSQL index types exist beyond B-tree (GIN, GiST, BRIN, partial, expression, covering with INCLUDE) — what problem does each solve, when would you choose each over a plain B-tree, and when do specialized indexes hurt more than they help?</summary>

**B-tree** (default) — Handles equality and range queries (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`). The right choice for 90% of cases. Supports unique constraints.

**GIN (Generalized Inverted Index)** — Designed for values that contain multiple elements: arrays, JSONB, full-text search (`tsvector`). Works like a book index — maps each element to the rows containing it. Use when you're querying "does this array/JSON contain X?" Slower to update than B-tree because inserting one row may add many index entries.

```sql
CREATE INDEX idx_tags ON articles USING gin (tags);  -- array column
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];

CREATE INDEX idx_metadata ON products USING gin (metadata jsonb_path_ops);
SELECT * FROM products WHERE metadata @> '{"color": "red"}';
```

**GiST (Generalized Search Tree)** — Supports geometric/spatial data, range types, and full-text search. PostGIS relies entirely on GiST. Use for "is this point within this polygon?" or "do these date ranges overlap?"

```sql
CREATE INDEX idx_location ON stores USING gist (coordinates);
CREATE INDEX idx_booking ON reservations USING gist (during);  -- tsrange column
SELECT * FROM reservations WHERE during && '[2024-01-01, 2024-01-07)';
```

**BRIN (Block Range Index)** — Extremely small index that stores min/max values per block range. Only useful when data is physically ordered on disk (e.g., append-only tables with a timestamp). A BRIN on a `created_at` column of a time-series table might be 1000x smaller than a B-tree, and still prune 99% of blocks for time-range queries.

```sql
CREATE INDEX idx_created ON events USING brin (created_at);
-- Only useful if rows are inserted roughly in created_at order
```

**Partial index** — Indexes only rows matching a WHERE condition. Smaller, faster, and more targeted.

```sql
CREATE INDEX idx_active_users ON users (email) WHERE active = true;
-- Only indexes active users — 10% of the table, 10% of the index size
```

**Expression index** — Indexes the result of a function, not the raw column value.

```sql
CREATE INDEX idx_lower_email ON users (lower(email));
SELECT * FROM users WHERE lower(email) = 'john@example.com';
```

**Covering index (INCLUDE)** — Adds non-key columns to a B-tree index so queries can be satisfied entirely from the index (index-only scan) without visiting the heap.

```sql
CREATE INDEX idx_orders_status ON orders (status) INCLUDE (total, created_at);
-- SELECT total, created_at FROM orders WHERE status = 'pending' → index-only scan
```

**When specialized indexes hurt:**

- GIN on a column that's updated frequently — every update rebuilds many index entries, crushing write performance
- BRIN on unordered data — it can't prune any blocks, so it scans everything anyway
- Too many partial indexes with overlapping conditions — maintenance overhead and planner confusion
- Any index on a tiny table — sequential scan is faster than index overhead

</details>

<details>
<summary>5. Why is VACUUM critical for PostgreSQL's health and what happens without it — how does autovacuum work, what is transaction ID wraparound and why is it an existential threat, what are the warning signs that VACUUM is falling behind, and how do you tune autovacuum for high-write tables?</summary>

**Why VACUUM is critical** (building on question 2's MVCC explanation):

Dead tuples from updates and deletes accumulate in the heap. VACUUM does three things:
1. **Reclaims dead tuple space** — marks it as reusable for future inserts (but doesn't return it to the OS — VACUUM FULL does, with a full table lock)
2. **Updates the visibility map** — tracks which pages are fully visible to all transactions, enabling index-only scans
3. **Freezes old transaction IDs** — prevents transaction ID wraparound

**How autovacuum works:**

Autovacuum is a background launcher that periodically checks each table. It triggers VACUUM when:

```
dead tuples > autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor * table_size)
```

Defaults: threshold = 50 rows, scale_factor = 0.2 (20%). So a 1M-row table gets vacuumed after ~200,000 dead tuples. For high-write tables, this is often too late.

Autovacuum is throttled by `autovacuum_vacuum_cost_delay` (default 2ms pause per cost unit) to avoid overwhelming I/O. This means it works slowly by design.

**Transaction ID wraparound:**

PostgreSQL uses 32-bit transaction IDs (~4.2 billion). It uses modular arithmetic to determine "past" vs "future" — a transaction can only see ~2 billion transactions in each direction. If a table has tuples with transaction IDs that haven't been frozen (marked as "definitely visible to everyone"), and the counter wraps around, those tuples would suddenly appear to be from the future and become invisible. Data loss without actual deletion.

When the database gets within ~10 million transactions of wraparound, it enters aggressive autovacuum mode. If that fails, PostgreSQL refuses to start new transactions — the database effectively goes read-only to protect itself. This is a production emergency.

**Warning signs VACUUM is falling behind:**

```sql
-- Dead tuple ratio
SELECT relname, n_dead_tup, n_live_tup,
       n_dead_tup::float / NULLIF(n_live_tup, 0) AS dead_ratio
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- How close to wraparound
SELECT datname, age(datfrozenxid) AS xid_age,
       2147483647 - age(datfrozenxid) AS remaining
FROM pg_database ORDER BY xid_age DESC;

-- Last vacuum time
SELECT relname, last_autovacuum, last_vacuum
FROM pg_stat_user_tables WHERE last_autovacuum IS NULL;
```

Red flags: dead_ratio > 0.2, xid_age > 500 million, tables that haven't been vacuumed in days.

**Tuning for high-write tables:**

```sql
ALTER TABLE hot_table SET (
  autovacuum_vacuum_scale_factor = 0.01,    -- trigger at 1% dead tuples instead of 20%
  autovacuum_vacuum_cost_delay = 0,          -- no throttling — vacuum as fast as possible
  autovacuum_vacuum_threshold = 1000         -- lower absolute threshold
);
```

You can also increase `autovacuum_max_workers` (default 3) globally if multiple large tables need concurrent vacuuming, and increase `autovacuum_work_mem` to allow VACUUM to track more dead tuples per pass.

</details>

<details>
<summary>6. How do PostgreSQL's transaction isolation levels (Read Committed, Repeatable Read, Serializable) differ in practice — what anomalies does each prevent, why is Read Committed the default, when do you need Serializable and what's the performance cost, and how does PostgreSQL implement these via MVCC snapshots?</summary>

**The three levels and what they prevent:**

| Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Serialization Anomalies |
|-------|------------|----------------------|---------------|------------------------|
| Read Committed | Prevented | Possible | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Prevented* | Possible |
| Serializable | Prevented | Prevented | Prevented | Prevented |

*PostgreSQL's Repeatable Read actually prevents phantoms too (stricter than SQL standard requires), but allows write skew.

**Read Committed** (default) — Takes a new snapshot at the start of each statement within a transaction. If another transaction commits between your two SELECTs, the second SELECT sees the new data. This is the default because it's the best tradeoff for most applications: no dirty reads, minimal conflict, and statements always see committed data. The downside: you can get inconsistent reads within a single transaction.

**Repeatable Read** — Takes a single snapshot at the start of the transaction. All queries within the transaction see the same data, regardless of what other transactions commit. If another transaction modifies a row you're also trying to modify, PostgreSQL throws a serialization error (`could not serialize access due to concurrent update`) and you must retry. Use for: read-heavy reports that need a consistent view, or any logic that reads a value and then writes based on it.

**Serializable** — The strictest level. PostgreSQL uses Serializable Snapshot Isolation (SSI) to detect dependency cycles between concurrent transactions. It tracks read and write dependencies and aborts transactions that would violate serializability. The performance cost: more transaction aborts (your application MUST have retry logic), additional memory for tracking dependencies (`max_pred_locks_per_transaction`), and reduced throughput under high concurrency.

**How MVCC snapshots implement this:**

A snapshot is essentially a list: "these transaction IDs were in-progress when I started." A tuple is visible if its `xmin` is committed and not in my snapshot's in-progress list, and its `xmax` is either 0 or from a transaction that's still in-progress or was aborted.

- Read Committed: new snapshot per statement
- Repeatable Read / Serializable: single snapshot for the entire transaction

**When to use Serializable:**

Financial calculations where write skew would cause incorrect balances, inventory systems where overselling is unacceptable, or any case where correctness depends on multiple transactions not interleaving in specific ways. Always pair with retry logic:

```typescript
async function withSerializableRetry<T>(
  pool: Pool,
  fn: (client: PoolClient) => Promise<T>,
  maxRetries = 3
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err: any) {
      await client.query('ROLLBACK');
      if (err.code === '40001' && attempt < maxRetries - 1) continue; // serialization failure
      throw err;
    } finally {
      client.release();
    }
  }
  throw new Error('Max retries exceeded');
}
```

</details>

<details>
<summary>7. How does PostgreSQL's locking system work and why does it offer multiple lock granularities -- what are the differences between row-level locks, table-level locks, and advisory locks, when do you use SELECT FOR UPDATE vs SELECT FOR SHARE, how does PostgreSQL detect and resolve deadlocks, and when are advisory locks the right tool vs row-level locks?</summary>

**Row-level locks:**

Acquired implicitly by UPDATE, DELETE, and explicitly by SELECT ... FOR UPDATE/SHARE. They only block other transactions trying to lock the same rows — they never block readers (MVCC handles read visibility). Row locks are stored in the tuple header (`xmax`), not in a separate lock table, so they're lightweight even for millions of locked rows.

- **SELECT FOR UPDATE** — Locks selected rows exclusively. Other transactions attempting FOR UPDATE on the same rows will block. Use when you need to read-then-write atomically: "fetch the account balance, then deduct."
- **SELECT FOR SHARE** — Shared lock. Multiple transactions can hold FOR SHARE on the same rows, but FOR UPDATE will block. Use when you need to ensure a referenced row doesn't change (e.g., ensure a parent row exists and won't be deleted while you insert a child).
- **SKIP LOCKED** — Non-blocking variant: skip rows that are already locked. Perfect for job queues:

```sql
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

**Table-level locks:**

Acquired implicitly by DDL operations. Most DML takes a low-level lock (RowExclusiveLock) that's compatible with other DML. But `ALTER TABLE ADD COLUMN`, `DROP TABLE`, and similar DDL take AccessExclusiveLock, which blocks everything including reads. This is why naive DDL on production tables causes outages.

Key compatibility: multiple transactions can hold RowExclusiveLock simultaneously (that's how concurrent INSERTs work), but AccessExclusiveLock is incompatible with everything.

**Advisory locks:**

Application-level locks managed by PostgreSQL but with no connection to actual data. You choose an arbitrary integer (or pair of integers) as the lock key.

```sql
SELECT pg_advisory_lock(12345);      -- blocks until acquired
SELECT pg_try_advisory_lock(12345);  -- returns true/false immediately
SELECT pg_advisory_unlock(12345);
```

Use cases: preventing concurrent execution of a cron job, rate-limiting per tenant, distributed mutex across application instances sharing a database. Unlike row locks, advisory locks have no implicit deadlock detection between them and row locks — be careful mixing them.

**When advisory locks beat row locks:** When the thing you're synchronizing isn't a specific row — it's a logical operation. "Only one process should run this data import at a time" doesn't correspond to any database row.

**Deadlock detection:**

PostgreSQL runs a deadlock detector every `deadlock_timeout` (default 1 second). It builds a wait-for graph of transactions and looks for cycles. When a cycle is found, it picks one transaction (typically the one that would be cheapest to roll back) and aborts it with a deadlock error. The other transaction(s) proceed.

Prevention strategies: always lock rows in a consistent order (e.g., by primary key), keep transactions short, and avoid acquiring locks you don't need.

**Diagnosing locks:**

```sql
SELECT blocked.pid AS blocked_pid,
       blocked.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks lock ON lock.locktype = bl.locktype
  AND lock.relation = bl.relation
  AND lock.pid != bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = lock.pid
WHERE NOT bl.granted;
```

</details>

<details>
<summary>8. Why does PostgreSQL's process-per-connection architecture create a hard ceiling on concurrency -- what resources does each backend process consume, why is max_connections typically set much lower than the number of concurrent application threads, what makes idle-in-transaction sessions particularly dangerous (holding locks, preventing VACUUM, consuming connection slots), and how should you size application-level connection pools independently of whether you use PgBouncer?</summary>

**Process-per-connection architecture:**

Every client connection spawns a dedicated OS process (not a thread). Each backend process consumes:
- **Memory**: `work_mem` (for sorts/hashes, allocated per operation per query), `temp_buffers`, plus the process's own stack and heap — typically 5-10 MB idle, potentially hundreds of MB under heavy queries
- **Shared memory**: a slot in the proc array, lock table entries, buffer pin tracking
- **OS resources**: a PID, file descriptors, context-switching overhead

At 500 connections, you might consume 5 GB of RAM just for process overhead before any queries run. Context switching between 500 processes degrades performance non-linearly — throughput often drops after ~100-200 active connections on typical hardware.

**Why max_connections should be low:**

The optimal number of *actively querying* connections is roughly `(CPU cores * 2) + effective_spindle_count` (a formula from the PostgreSQL wiki). For an 8-core server with SSDs, that's ~20 active connections for maximum throughput. Setting `max_connections = 500` doesn't help — it just allows 480 connections to contend for resources, making everyone slower.

**Idle-in-transaction danger:**

A session in `idle in transaction` state (started a transaction but hasn't committed or rolled back) is uniquely harmful:
1. **Holds locks** — any row locks from SELECT FOR UPDATE or data modifications remain held, blocking other transactions
2. **Prevents VACUUM** — VACUUM can't clean dead tuples that might still be visible to this transaction's snapshot, causing table bloat across ALL tables
3. **Consumes a connection slot** — doing nothing useful but preventing other clients from connecting
4. **Holds a snapshot** — prevents transaction ID wraparound protection on the entire database

Set `idle_in_transaction_session_timeout` (e.g., `30s`) to automatically kill these sessions.

**Application-level pool sizing:**

Size the pool based on actual concurrency needs, not the number of application instances:

```
pool_size = connections_per_instance * num_instances <= max_connections - reserved
```

Practical sizing:
- Start with `pool_size = CPU_cores * 2` per instance (matching PostgreSQL's optimal active connection count)
- If you have 4 instances, each gets `max_connections / 4` minus some headroom for admin connections
- A common setup: `max_connections = 100`, 4 app instances with `pool_size = 20` each, leaving 20 for monitoring/admin

**With PgBouncer** (transaction-mode pooling), your application pools can be larger because PgBouncer multiplexes many app connections onto fewer PostgreSQL connections. But you still need application-level pooling to avoid overwhelming PgBouncer itself. PgBouncer handles connection reuse; your app pool handles connection lifecycle and backpressure.

</details>

<details>
<summary>9. Why do data type choices matter in PostgreSQL schema design — when should you use timestamptz vs timestamp, text vs varchar(n), identity columns vs serial, why is the money type a trap (and what to use instead for currency), and what are the UUID generation options (gen_random_uuid, uuid-ossp, pgcrypto) and their tradeoffs?</summary>

**timestamptz vs timestamp:**

Always use `timestamptz` (timestamp with time zone). Despite the name, it doesn't store the time zone — it converts the input to UTC for storage and converts back to the session's time zone on retrieval. `timestamp` (without time zone) stores the literal value with no conversion — if your server moves time zones, or different clients have different `timezone` settings, the same value means different things. This is a source of subtle, hard-to-debug errors.

```sql
-- timestamptz: same moment, different displays based on session timezone
SET timezone = 'America/New_York';
SELECT '2024-01-15 14:00:00+00'::timestamptz;  -- 2024-01-15 09:00:00-05

SET timezone = 'UTC';
SELECT '2024-01-15 14:00:00+00'::timestamptz;  -- 2024-01-15 14:00:00+00
```

The only valid use for `timestamp` is when the value isn't a point in time — like "this store opens at 09:00" regardless of time zone.

**text vs varchar(n):**

Use `text`. PostgreSQL stores `text`, `varchar`, and `varchar(n)` identically — there's zero performance difference. The only thing `varchar(n)` adds is a length check on every insert/update. If you need length validation, do it in your application where you can give meaningful error messages. The real cost of `varchar(n)`: when you need to increase the limit, it requires an `ALTER TABLE` that (before PG 11) rewrites the entire table.

**identity vs serial:**

Use `GENERATED ALWAYS AS IDENTITY` (SQL standard) instead of `serial` (PostgreSQL extension).

```sql
-- Old way (serial) - creates a sequence with implicit ownership, confusing on dump/restore
CREATE TABLE orders (id serial PRIMARY KEY);

-- New way (identity) - SQL standard, cleaner ownership, prevents accidental manual values
CREATE TABLE orders (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY);
```

`serial` creates a standalone sequence and sets a default — the sequence ownership is implicit and can get out of sync during dump/restore. `IDENTITY` is part of the column definition, cleaner and standard-compliant.

**Money type trap:**

The `money` type formats output based on the `lc_monetary` locale setting. Change the locale, and your data displays differently. It also has limited precision and doesn't support multiple currencies. Instead:

```sql
-- Store as integer cents (no floating point issues)
price_cents integer NOT NULL  -- $19.99 stored as 1999

-- Or use numeric for sub-cent precision
price numeric(12, 4) NOT NULL  -- supports $99,999,999.9999
```

Integer cents is simplest and most common. `numeric` when you need sub-cent precision (financial calculations, exchange rates).

**UUID generation:**

- **`gen_random_uuid()`** — Built into PostgreSQL 13+ (no extension needed). Generates UUIDv4 (random). Best default choice.
- **`uuid-ossp` extension** — Provides `uuid_generate_v4()` (same as above) plus v1 (timestamp-based) and v5 (namespace-based). Only needed if you want non-v4 UUIDs.
- **UUIDv7** — Not natively supported yet but increasingly used via application-level generation. Time-ordered UUIDs that are B-tree friendly (sequential, so no random page splits). If using UUID primary keys at scale, generating UUIDv7 in your application and passing it to PostgreSQL gives better index performance than random UUIDv4.

**Performance note on UUID PKs:** Random UUIDs cause random I/O for B-tree index insertions (each new value goes to a random leaf page). At tens of millions of rows, this becomes measurable. UUIDv7 or `bigint` identity columns avoid this.

</details>

<details>
<summary>10. Why would you partition a PostgreSQL table and what are the tradeoffs — how does declarative partitioning work (range, list, hash), how does partition pruning speed up queries, and what maintenance and query-planning challenges do partitioned tables introduce?</summary>

**When to partition:**

Partitioning splits a large table into smaller physical tables (partitions) that look like one logical table. Consider it when:
- A table is large enough that maintenance operations (VACUUM, reindex) take too long on the whole table
- Queries consistently filter on a specific column (date range, tenant ID) and you want to skip irrelevant data entirely
- You need to efficiently drop old data (detach and drop a partition instead of DELETE + VACUUM)

**Declarative partitioning (PostgreSQL 10+):**

```sql
-- Range partitioning (most common — time-series data)
CREATE TABLE events (
  id bigint GENERATED ALWAYS AS IDENTITY,
  created_at timestamptz NOT NULL,
  payload jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_q1 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
  FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- List partitioning (multi-tenant, status-based)
CREATE TABLE orders (
  id bigint, region text NOT NULL, total numeric
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('us-east', 'us-west');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('eu-west', 'eu-central');

-- Hash partitioning (even distribution when no natural range/list key)
CREATE TABLE sessions (
  id uuid, user_id bigint NOT NULL, data jsonb
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... etc
```

**Partition pruning:**

When a query includes a WHERE clause on the partition key, PostgreSQL eliminates partitions that can't contain matching rows at plan time. A query for `WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01'` only scans the Q1 partition. Without partitioning, it scans the entire table. This is visible in EXPLAIN as `Partitions removed: N`.

**Challenges and tradeoffs:**

- **Query planning overhead** — The planner must consider all partitions. With hundreds of partitions, planning time itself becomes a bottleneck. Keep partition counts reasonable (dozens to low hundreds).
- **Unique constraints must include the partition key** — `PRIMARY KEY (id)` alone won't work; it must be `PRIMARY KEY (id, created_at)`. This is a fundamental PostgreSQL limitation.
- **Index maintenance** — Each partition has its own indexes. Creating a new partition requires creating indexes on it. Automated partition creation (via pg_partman or cron) must handle this.
- **Cross-partition queries** — Queries that don't filter on the partition key scan all partitions. If most of your queries don't include the partition key, partitioning hurts rather than helps.
- **Partition management** — Creating future partitions, detaching/dropping old ones, handling the default partition (catches rows that don't match any partition — monitor it, it should stay empty). pg_partman automates this.
- **No global indexes** — Each partition has local indexes. A query that needs to sort across partitions may merge results from many index scans, which can be slower than a single index scan on an unpartitioned table.

**Rule of thumb:** Don't partition until a table is at least tens of millions of rows AND you have queries that naturally filter on the partition key. Premature partitioning adds complexity with no benefit.

</details>

## Practical — SQL & Query Patterns

<details>
<summary>11. Query JSONB data in PostgreSQL — show containment operators (@>, ?), path operators (->>, #>>), indexing with GIN (jsonb_path_ops vs default operator class), and demonstrate a query that benefits from a GIN index. When should you store data as JSONB vs in normalized relational columns, and what queries become painful with JSONB?</summary>

**JSONB operators:**

```sql
-- Path operators: extract values
SELECT metadata->>'name' FROM products;              -- text value at key
SELECT metadata->'address'->>'city' FROM products;   -- nested path
SELECT metadata#>>'{address,city}' FROM products;    -- same via path array

-- Containment: does the JSONB contain this structure?
SELECT * FROM products WHERE metadata @> '{"color": "red"}';
SELECT * FROM products WHERE metadata @> '{"tags": ["sale"]}';

-- Existence: does a top-level key exist?
SELECT * FROM products WHERE metadata ? 'color';          -- has key 'color'
SELECT * FROM products WHERE metadata ?| ARRAY['color', 'size'];  -- has any of these keys
SELECT * FROM products WHERE metadata ?& ARRAY['color', 'size'];  -- has all of these keys
```

**GIN indexing — two operator classes:**

```sql
-- Default operator class: supports @>, ?, ?|, ?&
CREATE INDEX idx_metadata ON products USING gin (metadata);

-- jsonb_path_ops: supports only @> but is smaller and faster for containment queries
CREATE INDEX idx_metadata ON products USING gin (metadata jsonb_path_ops);
```

Use `jsonb_path_ops` when your queries are exclusively containment (`@>`), which is the most common pattern. It's ~2-3x smaller and faster than the default class. Use the default class when you also need key-existence queries (`?`, `?|`, `?&`).

**Query that benefits from GIN:**

```sql
-- With jsonb_path_ops GIN index, this uses the index for the @> operator
EXPLAIN ANALYZE
SELECT id, metadata->>'name'
FROM products
WHERE metadata @> '{"category": "electronics", "in_stock": true}';

-- Result: Bitmap Heap Scan → Bitmap Index Scan on idx_metadata
```

**Important:** GIN indexes do NOT help with `->>`  extraction in WHERE clauses:

```sql
-- This does NOT use the GIN index — it extracts text then compares
SELECT * FROM products WHERE metadata->>'color' = 'red';

-- For this pattern, use an expression index instead:
CREATE INDEX idx_color ON products ((metadata->>'color'));
```

**JSONB vs relational columns — when to use each:**

Use JSONB when:
- Schema varies per row (e.g., product attributes differ by category)
- You're storing third-party payloads or webhook data with unpredictable structure
- The data is read as a whole blob more often than queried by individual fields
- You need flexibility during rapid iteration and the schema isn't stable yet

Use relational columns when:
- You query, filter, or join on specific fields frequently
- You need constraints (NOT NULL, UNIQUE, foreign keys) on individual fields
- The structure is stable and well-known
- You need aggregations (SUM, AVG) on nested values — extracting from JSONB for every row is slow

**What becomes painful with JSONB:**
- Sorting by a nested field — requires extraction on every row, can't use a standard index efficiently
- JOINs on JSONB values — you end up extracting keys to join, which is slower than a proper FK column
- Enforcing data integrity — no constraints on JSONB internals; validation lives entirely in your application
- Aggregations across large datasets on nested values — sequential extraction is expensive

</details>

<details>
<summary>12. Given a slow query — walk through EXPLAIN ANALYZE output step by step: identify the scan type (Seq Scan, Index Scan, Index Only Scan, Bitmap Scan), understand join strategies (Nested Loop, Hash Join, Merge Join), read actual vs estimated rows to spot planner misestimates, and identify the single biggest bottleneck in the plan</summary>

**Reading EXPLAIN ANALYZE — a real example:**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, u.email
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending'
  AND o.created_at > '2024-01-01';
```

```
Hash Join  (cost=15.20..4521.33 rows=120 actual rows=8500 time=0.5..85.2ms)
  Hash Cond: (o.user_id = u.id)
  -> Seq Scan on orders o  (cost=0.00..4500.00 rows=120 actual rows=8500 time=0.1..80.3ms)
       Filter: (status = 'pending' AND created_at > '2024-01-01')
       Rows Removed by Filter: 991500
       Buffers: shared hit=12000 read=3500
  -> Hash  (cost=10.20..10.20 rows=500 actual rows=500 time=0.3..0.3ms)
       -> Seq Scan on users u  (cost=0.00..10.20 rows=500 actual rows=500)
```

**Step 1 — Find the biggest time consumer.** Read from inside out (leaf nodes first). The orders Seq Scan takes 80.3ms of the total 85.2ms. That's the bottleneck.

**Step 2 — Identify scan types:**

- **Seq Scan** — Reads every row in the table. Fine for small tables or when you're selecting a large fraction. Problematic here: scanning 1M rows to find 8,500.
- **Index Scan** — Traverses a B-tree to find matching rows, then fetches each row from the heap. Best for selective queries returning few rows.
- **Index Only Scan** — Same as Index Scan but all needed columns are in the index (covering index). Skips the heap visit entirely. Fastest scan type.
- **Bitmap Index Scan / Bitmap Heap Scan** — Two-phase: first builds a bitmap of matching pages from the index, then fetches those pages. Good for medium selectivity where Index Scan would be too many random I/Os but Seq Scan reads too much.

**Step 3 — Check estimated vs actual rows.** The planner estimated 120 rows but got 8,500. This 70x misestimate means the planner chose a Seq Scan thinking it wasn't worth using an index. The fix: `ANALYZE orders;` to update statistics, or check if the column has high correlation that the planner can't model.

**Step 4 — Understand join strategies:**

- **Nested Loop** — For each row in the outer table, scan the inner table. Best when the outer set is small and the inner has an index. O(n * m) without index.
- **Hash Join** — Builds a hash table from the smaller relation, then probes it for each row of the larger one. Best for equi-joins with no useful index. Needs memory for the hash table (`work_mem`).
- **Merge Join** — Both inputs must be sorted on the join key. Walks through both in order. Best when both sides are large and already sorted (e.g., from an index).

**Step 5 — Apply the fix:**

```sql
-- The bottleneck is the Seq Scan with two filter conditions
CREATE INDEX idx_orders_status_created ON orders (status, created_at)
  WHERE status = 'pending';  -- partial index since we only query pending

-- Re-run EXPLAIN ANALYZE to verify:
-- Should now show: Index Scan using idx_orders_status_created (actual rows=8500, time=2.1ms)
```

**Key things to look for in any plan:**
- `Rows Removed by Filter` — high numbers mean you're scanning too much
- `Buffers: read` — high values mean data isn't in cache, lots of disk I/O
- `Sort Method: external merge` — sort spilled to disk because `work_mem` is too low
- `actual loops` — multiply the node time by loops to get true cost

</details>

<details>
<summary>13. Write a recursive CTE that traverses a hierarchical structure (e.g., organization chart or category tree with parent_id) — show the SQL, explain how the recursive and non-recursive terms work together, and demonstrate when CTEs are optimization fences vs when they get inlined (PostgreSQL 12+ behavior with the MATERIALIZED/NOT MATERIALIZED hints)</summary>

**Recursive CTE — organization chart traversal:**

```sql
-- Table: employees (id, name, manager_id, department)
-- Find all reports (direct and indirect) under a given manager

WITH RECURSIVE org_tree AS (
  -- Non-recursive term (anchor): start with the root manager
  SELECT id, name, manager_id, 1 AS depth
  FROM employees
  WHERE id = 42  -- starting manager

  UNION ALL

  -- Recursive term: join each level's children
  SELECT e.id, e.name, e.manager_id, t.depth + 1
  FROM employees e
  JOIN org_tree t ON e.manager_id = t.id
)
SELECT * FROM org_tree ORDER BY depth, name;
```

**How it works:**

1. The non-recursive term runs once, producing the seed rows (the root manager)
2. The recursive term runs repeatedly, joining the working table (previous iteration's output) with the base table to find the next level
3. Each iteration adds newly discovered rows to the result set
4. Iteration stops when the recursive term produces zero new rows

**Practical additions:**

```sql
-- Build the full path and prevent infinite loops with a cycle guard
WITH RECURSIVE org_tree AS (
  SELECT id, name, manager_id, ARRAY[id] AS path, false AS is_cycle
  FROM employees WHERE id = 42

  UNION ALL

  SELECT e.id, e.name, e.manager_id,
         t.path || e.id,
         e.id = ANY(t.path)  -- cycle detection
  FROM employees e
  JOIN org_tree t ON e.manager_id = t.id
  WHERE NOT t.is_cycle  -- stop if we detect a cycle
)
SELECT * FROM org_tree WHERE NOT is_cycle;
```

**CTE optimization fences vs inlining (PostgreSQL 12+):**

Before PostgreSQL 12, all CTEs were optimization fences -- they materialized the CTE result first, then the outer query filtered it. The planner couldn't push WHERE conditions into the CTE.

```sql
-- Pre-PG12: this materializes ALL employees, then filters
WITH all_emps AS (
  SELECT * FROM employees
)
SELECT * FROM all_emps WHERE department = 'engineering';

-- Post-PG12: non-recursive CTEs referenced once are inlined by default
-- The planner pushes the WHERE into the CTE, equivalent to:
SELECT * FROM employees WHERE department = 'engineering';
```

**Controlling the behavior with hints:**

```sql
-- Force materialization (useful when the CTE is expensive and referenced multiple times)
WITH expensive_calc AS MATERIALIZED (
  SELECT user_id, sum(amount) AS total FROM transactions GROUP BY user_id
)
SELECT * FROM expensive_calc WHERE total > 1000
UNION ALL
SELECT * FROM expensive_calc WHERE total < 10;

-- Force inlining (useful when you know the outer filter dramatically reduces rows)
WITH all_orders AS NOT MATERIALIZED (
  SELECT * FROM orders  -- 100M rows
)
SELECT * FROM all_orders WHERE id = 12345;  -- planner pushes the id filter down
```

**Rules of thumb:**
- Recursive CTEs are always materialized (no choice)
- Non-recursive CTEs referenced once: inlined by default (PG 12+)
- Non-recursive CTEs referenced multiple times: materialized by default
- Use `MATERIALIZED` when the CTE is expensive and reused; use `NOT MATERIALIZED` when outer filters are highly selective

</details>

<details>
<summary>14. Use window functions for row numbering and ranking -- show ROW_NUMBER for deduplication or pagination, RANK vs DENSE_RANK for ranking with ties, and explain the PARTITION BY and ORDER BY interaction that controls how windows are defined. Demonstrate with SQL on a realistic table</summary>

**Table setup:** `sales (id, salesperson_id, region, amount, sale_date)`

**ROW_NUMBER for deduplication — get each salesperson's most recent sale:**

```sql
-- Common pattern: use ROW_NUMBER in a subquery, filter to rn = 1
SELECT * FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY salesperson_id ORDER BY sale_date DESC) AS rn
  FROM sales
) ranked
WHERE rn = 1;
```

ROW_NUMBER assigns a unique sequential number within each partition. Even if two sales have the same date, they get different numbers (arbitrary tiebreak). This makes it perfect for "pick exactly one" per group.

**ROW_NUMBER for row-number pagination:**

```sql
-- Page through results without OFFSET (which scans and discards)
-- Note: this still scans and numbers all rows — for true keyset pagination, see below
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (ORDER BY sale_date DESC, id DESC) AS rn
  FROM sales
  WHERE region = 'us-east'
) numbered
WHERE rn BETWEEN 21 AND 40;  -- page 2, 20 per page

-- True keyset pagination: uses the last-seen values to skip directly to the next page
-- Much faster at deep pages because it doesn't scan preceding rows
SELECT * FROM sales
WHERE region = 'us-east'
  AND (sale_date, id) < ($last_date, $last_id)
ORDER BY sale_date DESC, id DESC
LIMIT 20;
```

**RANK vs DENSE_RANK — handling ties:**

```sql
SELECT salesperson_id, amount,
  RANK()       OVER (ORDER BY amount DESC) AS rank,
  DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rank
FROM sales;
```

| salesperson_id | amount | rank | dense_rank |
|---|---|---|---|
| Alice | 5000 | 1 | 1 |
| Bob   | 5000 | 1 | 1 |
| Carol | 4000 | 3 | 2 |
| Dave  | 3000 | 4 | 3 |

- **RANK** skips numbers after ties: 1, 1, 3, 4 (Carol is 3rd, not 2nd)
- **DENSE_RANK** doesn't skip: 1, 1, 2, 3 (Carol is 2nd)

Use RANK for competitive rankings ("3 people finished ahead of you"). Use DENSE_RANK for level/tier assignments ("this is the 2nd highest amount").

**PARTITION BY and ORDER BY interaction:**

```sql
-- Running total per region, ordered by date
SELECT region, sale_date, amount,
  SUM(amount) OVER (
    PARTITION BY region           -- reset the window for each region
    ORDER BY sale_date            -- running sum in date order
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- default frame
  ) AS running_total
FROM sales;
```

- `PARTITION BY` divides rows into independent groups (like GROUP BY but without collapsing rows)
- `ORDER BY` determines the sequence within each partition
- Together: "within each region, in date order, compute a running total"

Without `ORDER BY`, the window function operates over the entire partition at once (no running behavior). Without `PARTITION BY`, the entire result set is one partition.

**LAG/LEAD — comparing with adjacent rows:**

```sql
-- Show each sale alongside the previous sale's amount for the same salesperson
SELECT salesperson_id, sale_date, amount,
  LAG(amount, 1) OVER (PARTITION BY salesperson_id ORDER BY sale_date) AS prev_amount,
  amount - LAG(amount, 1) OVER (PARTITION BY salesperson_id ORDER BY sale_date) AS change
FROM sales;
```

</details>

<details>
<summary>15. Create specialized indexes for real scenarios — show a partial index (only active users), an expression index (lower(email) for case-insensitive lookup), and a covering index with INCLUDE (to enable index-only scans). For each, demonstrate the query it optimizes and show EXPLAIN proving it's being used</summary>

Note: The index type definitions were covered in question 4. This answer focuses on practical creation, the queries they serve, and EXPLAIN verification.

**Partial index — only active users:**

```sql
-- Scenario: users table has 10M rows, only 500K are active.
-- Login queries always filter WHERE active = true.
CREATE INDEX idx_users_active_email ON users (email) WHERE active = true;
```

```sql
-- Query it optimizes:
EXPLAIN ANALYZE
SELECT id, email FROM users WHERE email = 'john@example.com' AND active = true;

-- Plan:
-- Index Scan using idx_users_active_email on users
--   Index Cond: (email = 'john@example.com')
--   Rows Removed by Filter: 0
--   Planning Time: 0.1ms
--   Execution Time: 0.03ms
```

The index is ~5% the size of a full B-tree on email because it only contains active rows. Without the partial index, PostgreSQL would either use a full email index and then filter on `active`, or use a composite index `(email, active)` that includes 9.5M inactive rows unnecessarily.

**Expression index — case-insensitive email lookup:**

```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));
```

```sql
-- Query it optimizes:
EXPLAIN ANALYZE
SELECT * FROM users WHERE lower(email) = lower('John@Example.COM');

-- Plan:
-- Index Scan using idx_users_lower_email on users
--   Index Cond: (lower(email) = 'john@example.com'::text)
--   Execution Time: 0.04ms
```

**Critical gotcha:** The WHERE clause must use the exact same expression as the index. `WHERE email = 'john@example.com'` (without `lower()`) will NOT use this index — it uses the raw column, not the expression. Also, `WHERE LOWER(email)` works because PostgreSQL normalizes function names, but `WHERE email ILIKE 'john@example.com'` does NOT use this index (ILIKE is a different operator).

**Covering index with INCLUDE — index-only scan:**

```sql
-- Scenario: dashboard query fetches order status and total, filtered by user_id
CREATE INDEX idx_orders_user ON orders (user_id) INCLUDE (status, total, created_at);
```

```sql
-- Query it optimizes:
EXPLAIN ANALYZE
SELECT status, total, created_at FROM orders WHERE user_id = 12345;

-- Plan:
-- Index Only Scan using idx_orders_user on orders
--   Index Cond: (user_id = 12345)
--   Heap Fetches: 0          <-- no heap visit needed
--   Execution Time: 0.05ms
```

Without INCLUDE, this would be an Index Scan (finds user_id in the index, then visits the heap for status/total/created_at). With INCLUDE, all needed columns are in the index leaf pages.

**Why INCLUDE instead of a composite index `(user_id, status, total, created_at)`?** INCLUDE columns are not part of the B-tree key — they're only stored in leaf pages. This means:
- The tree is smaller and faster to traverse (only user_id in the branch nodes)
- INCLUDE columns can be types that B-tree doesn't support for ordering (like jsonb)
- You can't use INCLUDE columns in WHERE or ORDER BY via this index — they're payload, not keys

**Heap Fetches caveat:** Index-only scans still check the visibility map. If a page isn't all-visible (recent updates, VACUUM hasn't run), PostgreSQL must fetch the heap page to check tuple visibility. Run VACUUM regularly to keep `Heap Fetches: 0`.

</details>

<details>
<summary>16. What are the most common SQL antipatterns in PostgreSQL and how do you fix them — show why NOT IN fails silently when the subquery contains NULLs and how NOT EXISTS avoids this, demonstrate why BETWEEN with timestamps double-counts rows at boundaries and the correct >= / < pattern, and explain any other antipatterns you've encountered (implicit casting defeating indexes, SELECT * in production queries)?</summary>

**Antipattern 1: NOT IN with NULLs**

```sql
-- Find users who haven't placed any orders
-- BROKEN: returns zero rows if ANY order.user_id is NULL
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM orders);
```

Why: SQL's three-valued logic. `NOT IN` checks `id != val1 AND id != val2 AND ...`. If any value is NULL, `id != NULL` evaluates to `UNKNOWN`, and `AND UNKNOWN` makes the entire expression `UNKNOWN`, filtering out the row. One NULL in the subquery silently returns zero results.

```sql
-- FIX: use NOT EXISTS (handles NULLs correctly)
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- Alternative: filter NULLs explicitly (but NOT EXISTS is cleaner)
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL);
```

**Antipattern 2: BETWEEN with timestamps**

```sql
-- Count orders per day — BROKEN: rows at exactly midnight get counted in both days
SELECT date_trunc('day', created_at) AS day, count(*)
FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY 1;
```

`BETWEEN` is inclusive on both ends. A row with `created_at = '2024-01-31 00:00:00'` matches this range AND the next month's range if you use `BETWEEN '2024-01-31' AND '2024-02-28'`. This causes double-counting at boundaries.

```sql
-- FIX: use half-open interval (>= start, < end)
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
```

This is the standard pattern for time ranges. Every timestamp falls into exactly one bucket.

**Antipattern 3: Implicit casting defeating indexes**

```sql
-- orders.id is bigint, but the parameter is passed as text
SELECT * FROM orders WHERE id = '12345';
-- PostgreSQL casts '12345' to bigint → index works (lucky)

-- But the reverse is dangerous:
-- users.phone is text, queried with an integer
SELECT * FROM users WHERE phone = 5551234;
-- PostgreSQL casts the COLUMN to numeric for comparison → full table scan
-- The index on phone (text) can't be used because the comparison is on numeric
```

Fix: always ensure parameter types match column types. In TypeScript, use parameterized queries and let the driver handle type mapping. Watch for ORM-generated queries that might cast incorrectly.

**Antipattern 4: SELECT * in production queries**

```sql
-- AVOID in production code:
SELECT * FROM users WHERE id = 123;
```

Problems:
- Returns columns you don't need, wasting bandwidth and memory
- Prevents index-only scans (the query fetches all columns, so it must visit the heap)
- Schema changes silently add columns to your result set, potentially breaking serialization or leaking sensitive data
- Makes it impossible to know which columns your code actually depends on

```sql
-- FIX: always specify columns
SELECT id, email, name FROM users WHERE id = 123;
```

**Antipattern 5: N+1 queries from ORMs**

```sql
-- ORM loads 100 orders, then fires 100 individual queries for users
SELECT * FROM orders WHERE status = 'pending'; -- returns 100 rows
SELECT * FROM users WHERE id = 1;   -- for order 1
SELECT * FROM users WHERE id = 2;   -- for order 2
-- ... 98 more queries
```

Fix: use eager loading (`JOIN` or `IN` query) or configure your ORM's relation loading:

```sql
-- Single query with JOIN
SELECT o.*, u.email FROM orders o JOIN users u ON u.id = o.user_id WHERE o.status = 'pending';

-- Or batch the lookup
SELECT * FROM users WHERE id IN (1, 2, 3, ...);
```

</details>

<details>
<summary>17. Write SQL that demonstrates the practical difference between Read Committed and Repeatable Read isolation — show a scenario where Read Committed sees a phantom row that Repeatable Read doesn't, and a scenario where Repeatable Read throws a serialization error that requires application-level retry logic</summary>

**Scenario 1: Read Committed sees phantom rows, Repeatable Read doesn't**

The isolation level concepts were covered in question 6. Here's the concrete SQL demonstrating the difference.

```sql
-- Setup
CREATE TABLE accounts (id int PRIMARY KEY, balance numeric);
INSERT INTO accounts VALUES (1, 1000), (2, 2000);

-- === READ COMMITTED ===

-- Transaction A (Read Committed - default)         -- Transaction B
BEGIN;                                               --
SELECT sum(balance) FROM accounts; -- 3000           --
                                                     -- BEGIN;
                                                     -- INSERT INTO accounts VALUES (3, 500);
                                                     -- COMMIT;
SELECT sum(balance) FROM accounts; -- 3500 (!!!)     --
-- Phantom: a new row appeared between two reads
-- within the same transaction
COMMIT;

-- === REPEATABLE READ ===

-- Transaction A (Repeatable Read)                   -- Transaction B
BEGIN ISOLATION LEVEL REPEATABLE READ;               --
SELECT sum(balance) FROM accounts; -- 3500           --
                                                     -- BEGIN;
                                                     -- INSERT INTO accounts VALUES (4, 700);
                                                     -- COMMIT;
SELECT sum(balance) FROM accounts; -- 3500 (same!)   --
-- No phantom: the snapshot was taken at BEGIN,
-- so Transaction B's insert is invisible
COMMIT;
```

**Scenario 2: Repeatable Read throws a serialization error on concurrent update**

```sql
-- Both transactions read the same row and try to update it

-- Transaction A                                      -- Transaction B
BEGIN ISOLATION LEVEL REPEATABLE READ;                --
SELECT balance FROM accounts WHERE id = 1; -- 1000   --
                                                      -- BEGIN ISOLATION LEVEL REPEATABLE READ;
                                                      -- SELECT balance FROM accounts WHERE id = 1; -- 1000
UPDATE accounts SET balance = 900 WHERE id = 1;       --
COMMIT;                                               --
                                                      -- UPDATE accounts SET balance = 800 WHERE id = 1;
                                                      -- ERROR: could not serialize access due to
                                                      -- concurrent update
                                                      -- Transaction B must ROLLBACK and retry
```

In Read Committed, Transaction B would succeed — it re-evaluates the WHERE clause against the updated row and applies its update on top. In Repeatable Read, PostgreSQL detects that the row was modified after B's snapshot and aborts B.

**Application-level retry (building on the retry pattern from question 6):**

```typescript
async function transferFunds(
  pool: Pool,
  fromId: number,
  toId: number,
  amount: number
): Promise<void> {
  const maxRetries = 3;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL REPEATABLE READ');

      const { rows } = await client.query(
        'SELECT balance FROM accounts WHERE id = $1',
        [fromId]
      );
      if (rows[0].balance < amount) throw new Error('Insufficient funds');

      await client.query(
        'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
        [amount, fromId]
      );
      await client.query(
        'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
        [amount, toId]
      );

      await client.query('COMMIT');
      return; // success
    } catch (err: any) {
      await client.query('ROLLBACK');
      if (err.code === '40001' && attempt < maxRetries - 1) {
        // Serialization failure — retry the entire transaction
        continue;
      }
      throw err;
    } finally {
      client.release();
    }
  }
}
```

**Key takeaway:** Repeatable Read gives you consistency within a transaction but demands that your application handles serialization failures. Read Committed never throws serialization errors but can show inconsistent data across statements.

</details>

## Practical — Schema & Migrations

<details>
<summary>18. Walk through zero-downtime migration patterns for a production table with 100M rows — show how to add a column with a default without locking (PostgreSQL 11+ fast default vs older backfill pattern), add a NOT NULL constraint via CHECK constraint first, and create an index with CREATE INDEX CONCURRENTLY. Explain what each approach avoids and what breaks if you use naive DDL</summary>

**Pattern 1: Adding a column with a default value**

Naive approach (breaks production):
```sql
-- Pre-PostgreSQL 11: this rewrites the ENTIRE table, holding AccessExclusiveLock
-- On 100M rows, this locks reads AND writes for minutes to hours
ALTER TABLE orders ADD COLUMN priority int DEFAULT 0;
```

**PostgreSQL 11+ fast default** — stores the default in the catalog, not in each row. New reads return the default without a table rewrite. Existing rows are backfilled lazily:

```sql
-- PG 11+: instant operation, no table rewrite
ALTER TABLE orders ADD COLUMN priority int DEFAULT 0;
-- This is now safe! Takes milliseconds regardless of table size.
```

**Pre-PG 11 backfill pattern** (or when you need a computed default):

```sql
-- Step 1: Add nullable column (instant, no lock)
ALTER TABLE orders ADD COLUMN priority int;

-- Step 2: Backfill in batches to avoid long-running transactions
-- Do NOT do: UPDATE orders SET priority = 0; (locks 100M rows, generates 100M dead tuples)
-- Use a PROCEDURE (PG 11+) which supports COMMIT inside loops:
CREATE OR REPLACE PROCEDURE backfill_priority(batch_size int DEFAULT 10000)
LANGUAGE plpgsql AS $$
BEGIN
  LOOP
    UPDATE orders SET priority = 0
    WHERE id IN (
      SELECT id FROM orders WHERE priority IS NULL LIMIT batch_size
    );
    IF NOT FOUND THEN EXIT; END IF;
    COMMIT;  -- release locks between batches (only valid in PROCEDURE, not DO blocks)
  END LOOP;
END $$;

CALL backfill_priority(10000);

-- Step 3: Add the default for new rows
ALTER TABLE orders ALTER COLUMN priority SET DEFAULT 0;
```

**Pattern 2: Adding NOT NULL constraint safely**

Naive approach:
```sql
-- Scans the ENTIRE table while holding AccessExclusiveLock
ALTER TABLE orders ALTER COLUMN priority SET NOT NULL;
```

Safe approach — use a CHECK constraint first:
```sql
-- Step 1: Add CHECK constraint as NOT VALID (instant, no scan)
ALTER TABLE orders ADD CONSTRAINT orders_priority_not_null
  CHECK (priority IS NOT NULL) NOT VALID;

-- Step 2: Validate the constraint (scans table but only holds ShareUpdateExclusiveLock,
-- which allows reads AND writes to continue)
ALTER TABLE orders VALIDATE CONSTRAINT orders_priority_not_null;

-- Step 3 (PG 12+): Now the NOT NULL can use the existing constraint as proof
ALTER TABLE orders ALTER COLUMN priority SET NOT NULL;
-- PG 12+ recognizes the validated CHECK and skips the full table scan

-- Step 4: Drop the now-redundant CHECK constraint
ALTER TABLE orders DROP CONSTRAINT orders_priority_not_null;
```

**Pattern 3: Creating indexes concurrently**

Naive approach:
```sql
-- Holds ShareLock on the table — blocks all writes until the index is built
-- On 100M rows, writes are blocked for minutes
CREATE INDEX idx_orders_priority ON orders (priority);
```

Safe approach:
```sql
-- CONCURRENTLY: two passes over the table, no write lock
-- First pass: builds the index while allowing writes
-- Second pass: catches up with changes made during the first pass
CREATE INDEX CONCURRENTLY idx_orders_priority ON orders (priority);
```

**Caveats of CONCURRENTLY:**
- Cannot run inside a transaction block (no `BEGIN`/`COMMIT`)
- Takes ~2-3x longer than a regular index build
- If it fails midway, it leaves an INVALID index that you must drop and retry: `DROP INDEX CONCURRENTLY idx_orders_priority;`
- Check for invalid indexes: `SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;`

**Summary of what each pattern avoids:**

| Operation | Naive DDL | Zero-downtime approach | What you avoid |
|---|---|---|---|
| Add column + default | Table rewrite (pre-PG11) | Fast default (PG11+) or batch backfill | Minutes of full table lock |
| NOT NULL constraint | Full table scan + AccessExclusiveLock | CHECK NOT VALID → VALIDATE → SET NOT NULL | Write downtime during validation |
| Create index | ShareLock (blocks writes) | CREATE INDEX CONCURRENTLY | Write downtime during build |

</details>

<details>
<summary>19. Design a schema using proper PostgreSQL data types — show when to use timestamptz (and why timestamp without time zone is almost always wrong), why text is preferred over varchar(n) for most cases, how identity columns replace serial, and the best approach for UUID primary keys with performance considerations</summary>

The data type rationale was covered in depth in question 9. This answer focuses on a practical schema demonstrating all the correct choices together.

```sql
-- A well-designed schema applying modern PostgreSQL type conventions

CREATE TABLE users (
  -- Identity column: SQL standard, replaces serial
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

  -- text over varchar(n): no performance difference, avoids migration pain
  email text NOT NULL,
  name text NOT NULL,

  -- boolean with a default
  active boolean NOT NULL DEFAULT true,

  -- timestamptz: always, unless the value isn't a point in time
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE products (
  -- UUID primary key: for public-facing IDs or distributed systems
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),  -- PG 13+, no extension needed

  name text NOT NULL,
  description text,  -- nullable text, not varchar

  -- Currency: integer cents, not money type, not float
  price_cents integer NOT NULL CHECK (price_cents >= 0),
  currency text NOT NULL DEFAULT 'USD' CHECK (char_length(currency) = 3),

  -- JSONB for variable attributes (category-specific fields)
  attributes jsonb NOT NULL DEFAULT '{}',

  -- Enum for a fixed, small set of values
  status text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
  -- Note: text + CHECK is often preferred over CREATE TYPE ... AS ENUM
  -- because adding enum values requires ALTER TYPE which can be awkward in migrations

  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id bigint NOT NULL REFERENCES users(id),
  product_id uuid NOT NULL REFERENCES products(id),

  quantity integer NOT NULL CHECK (quantity > 0),
  unit_price_cents integer NOT NULL,
  total_cents integer NOT NULL GENERATED ALWAYS AS (quantity * unit_price_cents) STORED,

  -- Date ranges: use tstzrange for booking/scheduling
  -- delivery_window tstzrange,

  ordered_at timestamptz NOT NULL DEFAULT now()
);
```

**UUID primary key performance considerations:**

Random UUIDv4 (from `gen_random_uuid()`) causes random B-tree leaf page insertions. At scale (tens of millions of rows), this means:
- Higher write amplification (random I/O vs sequential)
- More buffer cache pressure (pages aren't reused sequentially)
- Index fragmentation

Mitigations:
1. **Use UUIDv7** (time-ordered) generated in your application — sequential inserts like bigint but globally unique. Best of both worlds.
2. **Keep bigint as PK internally, expose UUID externally** — internal JOINs use bigint (fast), API consumers see UUIDs (no enumeration).

```typescript
// Generate UUIDv7 in Node.js (using the uuid package)
import { v7 as uuidv7 } from 'uuid';
const id = uuidv7(); // time-ordered, B-tree friendly
```

3. For tables under ~10M rows, the UUIDv4 random insertion penalty is negligible. Don't optimize prematurely.

</details>

<details>
<summary>20. Compare migration tooling: write the same migration (add table, add index, backfill data) using node-pg-migrate and Prisma migrate — show both, explain the tradeoffs (raw SQL control vs schema diff approach), demonstrate a rollback strategy for each, and explain how you handle data migrations that can't be in a DDL transaction</summary>

**The migration: add a `notifications` table, create an index, backfill from existing data.**

**node-pg-migrate approach (raw SQL control):**

```typescript
// migrations/1710000000000_add-notifications.ts
import { MigrationBuilder } from 'node-pg-migrate';

export async function up(pgm: MigrationBuilder): Promise<void> {
  // DDL: create table
  pgm.createTable('notifications', {
    id: { type: 'bigint', primaryKey: true, generated: { precedence: 'ALWAYS' } },
    user_id: { type: 'bigint', notNull: true, references: 'users' },
    message: { type: 'text', notNull: true },
    read: { type: 'boolean', notNull: true, default: false },
    created_at: { type: 'timestamptz', notNull: true, default: pgm.func('now()') },
  });

  // Index: CONCURRENTLY not possible inside a transaction
  // node-pg-migrate wraps each migration in a transaction by default
  pgm.createIndex('notifications', 'user_id');
  pgm.createIndex('notifications', ['user_id', 'read'], {
    where: 'read = false',
    name: 'idx_notifications_unread',
  });
}

export async function down(pgm: MigrationBuilder): Promise<void> {
  pgm.dropTable('notifications');
}
```

For the concurrent index + data backfill (which can't be in a transaction):

```typescript
// migrations/1710000001000_backfill-notifications.ts
// Run with: node-pg-migrate up --no-transaction
export async function up(pgm: MigrationBuilder): Promise<void> {
  // Backfill from existing events table in batches
  pgm.sql(`
    INSERT INTO notifications (user_id, message, created_at)
    SELECT user_id, 'Welcome!', created_at
    FROM users
    WHERE NOT EXISTS (
      SELECT 1 FROM notifications n WHERE n.user_id = users.id
    )
  `);
}

export async function down(pgm: MigrationBuilder): Promise<void> {
  pgm.sql(`DELETE FROM notifications WHERE message = 'Welcome!'`);
}
```

**Prisma migrate approach (schema diff):**

```prisma
// schema.prisma — add the model
model Notification {
  id        BigInt   @id @default(autoincrement())
  userId    BigInt   @map("user_id")
  message   String
  read      Boolean  @default(false)
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  user User @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([userId, read])
  @@map("notifications")
}
```

```bash
# Generate the migration SQL (doesn't run it yet)
npx prisma migrate dev --name add-notifications --create-only

# Review the generated SQL in prisma/migrations/..._add_notifications/migration.sql
# Then apply:
npx prisma migrate dev
```

Prisma generates the SQL diff automatically. You can edit the generated SQL before applying (important for production-safe patterns like CONCURRENTLY).

**Tradeoffs:**

| Aspect | node-pg-migrate | Prisma migrate |
|---|---|---|
| SQL control | Full — you write raw SQL/JS | Generated from schema diff, editable |
| Rollback | Explicit `down()` function you write | No auto-rollback; you create a new migration to undo |
| Concurrent indexes | `--no-transaction` flag | Edit generated SQL manually |
| Data migrations | First-class (JS code in migration) | Must edit generated SQL or use separate scripts |
| Schema drift detection | None — you manage state | Detects drift between schema and DB |
| Team workflow | Predictable — SQL is what runs | Magic — diff generation can surprise you |

**Rollback strategies:**

- **node-pg-migrate**: Built-in `down()` functions. Run `node-pg-migrate down` to reverse the last migration. You write and test the rollback explicitly.
- **Prisma**: No `down` migrations. To roll back, you create a new "undo" migration. In practice, many teams keep a rollback SQL script alongside each Prisma migration for emergencies.

**Handling data migrations outside transactions:**

Data backfills on large tables shouldn't be in a DDL transaction because:
1. Long-running transactions hold locks and prevent VACUUM (as covered in question 8)
2. `CREATE INDEX CONCURRENTLY` cannot run inside a transaction
3. Batch backfills need to commit between batches to release locks

Approach: separate DDL migrations (transactional) from data migrations (non-transactional). Run data migrations as standalone scripts or with the `--no-transaction` flag. In node-pg-migrate, this is a flag. In Prisma, you'd write a separate Node.js script that runs after the structural migration.

</details>

## Practical — Production Configuration

<details>
<summary>21. Tune autovacuum for a high-write table that's generating dead tuples faster than the default settings can clean up — show the per-table autovacuum settings (autovacuum_vacuum_scale_factor, autovacuum_vacuum_cost_delay), explain how to monitor whether autovacuum is keeping up, and what emergency steps to take when a table is approaching transaction ID wraparound</summary>

The autovacuum fundamentals were covered in question 5. This answer focuses on the practical tuning workflow.

**Step 1: Identify the problem table**

```sql
-- Tables with the most dead tuples
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 1) AS dead_pct,
       last_autovacuum, last_autoanalyze,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

If you see a table with dead_pct > 20% and `last_autovacuum` was recent, autovacuum is running but can't keep up with the write rate.

**Step 2: Apply aggressive per-table settings**

```sql
ALTER TABLE high_write_table SET (
  -- Trigger vacuum at 1% dead tuples instead of default 20%
  autovacuum_vacuum_scale_factor = 0.01,

  -- Lower absolute threshold (default 50)
  autovacuum_vacuum_threshold = 500,

  -- Remove I/O throttling for this table (default 2ms delay)
  autovacuum_vacuum_cost_delay = 0,

  -- Higher cost limit per round (default 200, shared across all workers)
  autovacuum_vacuum_cost_limit = 2000,

  -- Also tune ANALYZE to keep statistics fresh
  autovacuum_analyze_scale_factor = 0.01
);
```

**Step 3: Monitor whether autovacuum is keeping up**

```sql
-- Check if autovacuum is actively running on this table
SELECT pid, query, state, wait_event_type, wait_event,
       now() - xact_start AS duration
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%';

-- Track dead tuple trend over time (run periodically)
SELECT relname, n_dead_tup, n_live_tup,
       now() - last_autovacuum AS time_since_last_vacuum
FROM pg_stat_user_tables
WHERE relname = 'high_write_table';
```

Key indicators that autovacuum is keeping up:
- dead_pct stays below 5-10%
- `last_autovacuum` is within the last few minutes for high-write tables
- Table size isn't growing despite stable row count

**Global settings to consider:**

```sql
-- Increase max workers if multiple tables need concurrent vacuum (default 3)
ALTER SYSTEM SET autovacuum_max_workers = 6;

-- Increase memory for vacuum to track more dead tuples per pass
ALTER SYSTEM SET autovacuum_work_mem = '512MB';  -- default uses maintenance_work_mem

-- Reduce naptime between launcher checks (default 1 min)
ALTER SYSTEM SET autovacuum_naptime = '15s';

SELECT pg_reload_conf();  -- apply without restart
```

**Emergency: approaching transaction ID wraparound**

```sql
-- Check how close you are
SELECT datname, age(datfrozenxid) AS xid_age,
       2147483647 - age(datfrozenxid) AS xids_remaining,
       round(age(datfrozenxid)::numeric / 2147483647 * 100, 1) AS pct_consumed
FROM pg_database ORDER BY xid_age DESC;

-- Per-table check
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC LIMIT 10;
```

If `xids_remaining` is below ~50 million, take immediate action:

1. **Kill any long-running transactions** that hold back the visibility horizon:
```sql
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction' OR xact_start < now() - interval '1 hour';

-- Terminate if needed
SELECT pg_terminate_backend(pid);
```

2. **Run manual VACUUM FREEZE on the worst tables:**
```sql
VACUUM (FREEZE, VERBOSE) high_write_table;
```

3. **If autovacuum is running but too slow**, temporarily increase its aggressiveness globally, or run manual vacuum in parallel on multiple tables from separate sessions.

4. **If PostgreSQL has already entered read-only protection mode** (refuses new transactions), you must run `vacuumdb --all --freeze` from the command line as the superuser. This is a last resort — it will be slow and I/O intensive.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>22. Your API's p99 latency jumped from 50ms to 3 seconds — walk through diagnosing the PostgreSQL side: identify the slow query using pg_stat_statements, run EXPLAIN ANALYZE to find the bottleneck (missing index, stale statistics, seq scan on a large table), check if the query plan changed due to a statistics update or table growth, and apply the fix</summary>

**Step 1: Identify the slow query with pg_stat_statements**

```sql
-- Top queries by total time (p99 spike means something got slower)
SELECT query,
       calls,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(max_exec_time::numeric, 2) AS max_ms,
       round(total_exec_time::numeric / 1000, 1) AS total_sec,
       rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Or find queries whose mean time is suspiciously high relative to calls
SELECT query, calls, round(mean_exec_time::numeric, 2) AS avg_ms
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- > 1 second average
ORDER BY total_exec_time DESC;
```

Suppose we find: `SELECT * FROM orders WHERE customer_id = $1 AND status = $2 ORDER BY created_at DESC LIMIT 20` averaging 2800ms when it used to be 30ms.

**Step 2: Run EXPLAIN ANALYZE on the slow query**

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders
WHERE customer_id = 12345 AND status = 'completed'
ORDER BY created_at DESC LIMIT 20;
```

```
Limit (actual rows=20, time=2847ms)
  -> Sort (actual rows=20, time=2847ms)
        Sort Key: created_at DESC
        Sort Method: top-N heapsort  Memory: 28kB
        -> Seq Scan on orders (actual rows=45000, time=2840ms)
              Filter: (customer_id = 12345 AND status = 'completed')
              Rows Removed by Filter: 9955000
              Buffers: shared hit=50000 read=35000
```

**Diagnosis:** Seq Scan on 10M rows, filtering down to 45K. The planner chose not to use an index. Check why.

**Step 3: Check if the query plan changed**

```sql
-- Were statistics recently updated?
SELECT relname, last_analyze, last_autoanalyze, n_live_tup
FROM pg_stat_user_tables WHERE relname = 'orders';

-- Check if the table grew significantly
SELECT pg_size_pretty(pg_total_relation_size('orders'));

-- Check index existence
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';
```

Common root causes:
1. **Missing index** — no index on `(customer_id, status, created_at)` exists
2. **Stale statistics** — autoanalyze hasn't run after a large data load, so the planner thinks the table has 100K rows when it has 10M
3. **Plan flip** — statistics update changed the selectivity estimate, causing the planner to switch from index scan to seq scan
4. **Table bloat** — dead tuples make the table much larger than expected, making seq scans slower

**Step 4: Apply the fix**

If missing index:
```sql
-- Composite index matching the query's WHERE + ORDER BY
CREATE INDEX CONCURRENTLY idx_orders_customer_status_created
  ON orders (customer_id, status, created_at DESC);

-- Verify the new plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE customer_id = 12345 AND status = 'completed'
ORDER BY created_at DESC LIMIT 20;

-- Expected: Index Scan Backward using idx_orders_customer_status_created
-- Execution Time: 0.5ms
```

If stale statistics:
```sql
-- Update statistics for this table
ANALYZE orders;

-- For specific columns with unusual distributions, increase the statistics target
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;  -- default is 100
ANALYZE orders;
```

If plan flip (query was fast before, index exists but planner ignores it):
```sql
-- Check the correlation — how physically ordered is the data?
SELECT attname, correlation FROM pg_stats
WHERE tablename = 'orders' AND attname IN ('customer_id', 'status', 'created_at');

-- If the planner's row estimate is way off, statistics are stale
-- Compare estimated vs actual in EXPLAIN ANALYZE output
```

**Step 5: Verify without impacting production**

- Run `EXPLAIN (ANALYZE)` on a read replica first, not the primary
- Create the index with `CONCURRENTLY` to avoid blocking writes
- After applying, check pg_stat_statements again to confirm the avg_ms dropped
- Monitor the p99 in your APM tool over the next hour

</details>

<details>
<summary>23. Several transactions are blocked and your application is timing out — walk through diagnosing lock contention: use pg_locks joined with pg_stat_activity to find the blocking query, determine whether it's a row lock, advisory lock, or DDL lock, decide whether to wait or terminate the blocking process, and explain how to prevent this from recurring</summary>

**Step 1: Find what's blocking what**

```sql
-- The essential blocking query — shows blocker → blocked relationships
SELECT
  blocked_activity.pid AS blocked_pid,
  blocked_activity.query AS blocked_query,
  blocked_activity.state AS blocked_state,
  now() - blocked_activity.state_change AS blocked_duration,
  blocking_activity.pid AS blocking_pid,
  blocking_activity.query AS blocking_query,
  blocking_activity.state AS blocking_state,
  now() - blocking_activity.xact_start AS blocking_txn_duration,
  blocked_locks.locktype,
  blocked_locks.mode AS blocked_mode
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

**Step 2: Determine the lock type**

Look at the `locktype` and `blocked_mode` columns:

- **`relation` + `AccessExclusiveLock`** — DDL lock. Someone is running `ALTER TABLE`, `DROP TABLE`, or `TRUNCATE`. Even a pending DDL statement will queue behind existing transactions and block all subsequent queries on that table.
- **`relation` + `RowExclusiveLock` blocked by `AccessExclusiveLock`** — Normal DML blocked by DDL. This is the classic "migration blocking production" scenario.
- **`tuple` or `transactionid`** — Row-level lock contention. Two transactions are trying to UPDATE/DELETE the same row(s). Usually short-lived unless one transaction is idle-in-transaction.
- **`advisory`** — Advisory lock. Application-level lock held too long (covered in question 7).

**Step 3: Decide whether to wait or terminate**

Decision framework:
- **If the blocker is `idle in transaction`** — it's likely a bug (forgotten commit, connection leak). Terminate it.
- **If the blocker is actively running a DDL migration** — check with the team. If it's expected and almost done, wait. If it's been running for minutes on a large table, it was probably not done safely (should have used CONCURRENTLY).
- **If the blocker is a long-running analytical query** — it might be running on the primary instead of a replica. Terminate and redirect to a read replica.
- **If it's row-level contention between short transactions** — usually resolves itself. If one side is stuck, terminate the stale one.

```sql
-- Graceful: cancel the query (sends SIGINT, query stops but connection stays)
SELECT pg_cancel_backend(blocking_pid);

-- Forceful: terminate the connection entirely (sends SIGTERM)
SELECT pg_terminate_backend(blocking_pid);
```

Use `pg_cancel_backend` first. Only escalate to `pg_terminate_backend` if cancel doesn't work (e.g., the process is stuck in I/O).

**Step 4: Prevent recurrence**

For DDL lock issues:
- Always use `lock_timeout` before DDL: `SET lock_timeout = '3s';` — if the DDL can't acquire the lock in 3 seconds, it fails instead of queuing and blocking everything behind it
- Use zero-downtime migration patterns (as covered in question 18)

For idle-in-transaction:
- Set `idle_in_transaction_session_timeout = '30s'` (as covered in question 8)

For row-level contention:
- Keep transactions short — do all non-DB work outside the transaction
- Use `SKIP LOCKED` for queue-like patterns (as covered in question 7)
- Lock rows in consistent order to prevent deadlocks

For monitoring:
```sql
-- Alert when any query has been waiting on a lock for more than 30 seconds
SELECT pid, query, now() - state_change AS wait_duration
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
  AND state_change < now() - interval '30 seconds';
```

Add this as a monitoring check that pages on-call when triggered.

</details>

<details>
<summary>24. Your application suddenly can't connect to PostgreSQL and you see "too many connections" errors — walk through the diagnosis: check pg_stat_activity for idle connections, idle-in-transaction sessions, and connection leaks. Explain how PgBouncer misconfiguration, missing application-level pool limits, or long-running transactions cause this, and the immediate vs long-term fixes</summary>

**Step 1: Assess the situation**

```sql
-- How many connections exist and what are they doing?
SELECT state, count(*) FROM pg_stat_activity GROUP BY state ORDER BY count DESC;

-- What's the limit?
SHOW max_connections;  -- e.g., 100
```

Typical output revealing the problem:
```
state                | count
---------------------+------
idle                 | 45
idle in transaction  | 38
active               | 12
                     | 5    (background workers)
```

100 connections, only 12 active, 83 idle or idle-in-transaction. The database is full but barely working.

**Step 2: Dig into the culprits**

```sql
-- Idle-in-transaction: who and for how long?
SELECT pid, usename, client_addr, application_name,
       now() - xact_start AS txn_duration,
       now() - state_change AS idle_duration,
       query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;

-- Connection distribution by application/host
SELECT application_name, client_addr, count(*)
FROM pg_stat_activity
GROUP BY application_name, client_addr
ORDER BY count DESC;
```

This reveals which application instance is hoarding connections and whether it's one misbehaving instance or a systemic issue.

**Step 3: Identify root causes**

**PgBouncer misconfiguration:**
- `default_pool_size` set too high — PgBouncer opens more backend connections than `max_connections` allows
- `max_client_conn` too high without corresponding backend limits — accepts thousands of app connections but PostgreSQL can't handle them
- Session-mode pooling instead of transaction-mode — each client holds a backend connection for the entire session, defeating the purpose of pooling

**Missing application-level pool limits:**
- No `max` setting on the connection pool (e.g., `pg` Pool in Node.js defaults to 10, but if you have 20 instances that's 200 connections)
- Multiple connection pools in the same application (one for reads, one for writes, one for admin) that collectively exceed limits

**Connection leaks:**
- Code paths that acquire a connection but don't release it on error:
```typescript
// LEAK: if query throws, client is never released
const client = await pool.connect();
const result = await client.query('SELECT ...');
client.release();

// FIX: always use try/finally
const client = await pool.connect();
try {
  const result = await client.query('SELECT ...');
  return result;
} finally {
  client.release();
}

// BETTER: use pool.query() which handles acquire/release automatically
const result = await pool.query('SELECT ...');
```

**Long-running transactions:**
- Background jobs that open a transaction, do external API calls, then come back to commit — holding a connection (and snapshot) for minutes

**Step 4: Immediate fixes**

```sql
-- Kill idle-in-transaction sessions older than 5 minutes
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < now() - interval '5 minutes';

-- Kill idle connections from a specific misbehaving application
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE application_name = 'leaky-service'
  AND state = 'idle';
```

**Step 5: Long-term fixes**

```sql
-- Auto-kill idle-in-transaction sessions
ALTER SYSTEM SET idle_in_transaction_session_timeout = '30s';

-- Auto-kill completely idle sessions (PG 14+)
ALTER SYSTEM SET idle_session_timeout = '300s';

SELECT pg_reload_conf();
```

Application-side:
- Set explicit `max` on every connection pool: `new Pool({ max: 10 })`
- Calculate: `pool_max_per_instance * num_instances < max_connections - reserved_connections`
- Use `pool.query()` instead of manual `connect()`/`release()` where possible
- Add connection pool monitoring (pool size, waiting requests, idle connections) to your dashboards
- If using PgBouncer, set transaction-mode pooling and size `default_pool_size` to match PostgreSQL's optimal active connections (as discussed in question 8)

</details>

<details>
<summary>25. A table that was 2GB is now 20GB even though the row count hasn't changed significantly — walk through diagnosing table bloat: check pg_stat_user_tables for dead tuples, determine why autovacuum isn't keeping up (long-running transactions holding back the visibility horizon, insufficient worker settings), and show how to reclaim space (VACUUM FULL as last resort and its locking implications, pg_repack as an alternative)</summary>

**Step 1: Confirm bloat exists**

```sql
-- Basic check: dead tuples vs live tuples
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 1) AS dead_pct,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       pg_size_pretty(pg_relation_size(relid)) AS table_size,
       last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'bloated_table';
```

If `n_dead_tup` is high, autovacuum hasn't cleaned them. But even if `n_dead_tup` is low, the table can still be bloated -- VACUUM marks space as reusable but doesn't return it to the OS. The table file stays 20GB even after dead tuples are cleaned, with lots of empty space inside.

```sql
-- Estimate actual bloat using pgstattuple extension
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('bloated_table');
-- Look at: dead_tuple_percent and free_space_percent
-- If free_space > 50%, the table is heavily bloated
```

**Step 2: Determine why autovacuum isn't keeping up**

**Check 1: Long-running transactions holding back the visibility horizon**

```sql
-- Any old transactions preventing VACUUM from cleaning tuples?
SELECT pid, now() - xact_start AS duration, state, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 5;
```

A transaction from 3 hours ago means VACUUM can't remove any dead tuples created in the last 3 hours — across ALL tables. This is the most common cause of runaway bloat and was discussed in question 8 (idle-in-transaction dangers).

**Check 2: Autovacuum settings too conservative**

```sql
-- Is autovacuum running frequently enough?
SELECT relname, last_autovacuum, last_vacuum,
       n_dead_tup,
       -- When will autovacuum trigger?
       (SELECT setting::int FROM pg_settings WHERE name = 'autovacuum_vacuum_threshold')
       + n_live_tup * (SELECT setting::float FROM pg_settings WHERE name = 'autovacuum_vacuum_scale_factor')
       AS vacuum_trigger_threshold
FROM pg_stat_user_tables
WHERE relname = 'bloated_table';
```

If the trigger threshold is 200K dead tuples but the table generates 500K dead tuples per hour, autovacuum runs once but can't finish before the next batch arrives. Apply per-table tuning as shown in question 21.

**Check 3: Autovacuum worker starvation**

```sql
-- Are all autovacuum workers busy with other tables?
SELECT query, now() - xact_start AS duration
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%';

SHOW autovacuum_max_workers;  -- default 3
```

If all 3 workers are busy vacuuming other large tables, your bloated table waits in the queue.

**Step 3: Reclaim space**

**Regular VACUUM** (run first — no downtime):
```sql
VACUUM VERBOSE bloated_table;
```
This removes dead tuples and marks space as reusable, but does NOT shrink the file. New inserts will reuse the space, so the table won't grow further. Often this is sufficient — the 20GB file contains 2GB of data and 18GB of reusable space, and future inserts fill the gaps.

**VACUUM FULL** (last resort — causes downtime):
```sql
-- WARNING: acquires AccessExclusiveLock — blocks ALL reads and writes
-- Rewrites the entire table to a new file, reclaiming all space
-- On a 20GB table, this could take 10+ minutes with full downtime
VACUUM FULL bloated_table;
```

Only use VACUUM FULL when:
- The table is so bloated that sequential scans are unacceptably slow (scanning 20GB to read 2GB of data)
- Disk space is critically low and you need the space back NOW
- You can tolerate the downtime

**pg_repack** (online alternative — no downtime):
```sql
-- Install the extension
CREATE EXTENSION pg_repack;
```

```bash
# Run from command line — rewrites the table without locking reads/writes
pg_repack -t bloated_table -d mydb
```

pg_repack works by:
1. Creating a new copy of the table
2. Replaying changes made during the copy (using a trigger)
3. Swapping the old and new tables atomically

It needs roughly 2x the table's disk space temporarily, but doesn't block reads or writes. This is the production-safe way to reclaim space.

**Prevention:**
- Monitor dead tuple ratios and table size vs row count
- Set `idle_in_transaction_session_timeout` to prevent long-running transactions from blocking VACUUM
- Tune autovacuum aggressively for high-write tables (question 21)
- Consider partitioning time-series tables so old partitions can be dropped entirely instead of vacuumed (question 10)

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>26. Tell me about a time you had to optimize a slow PostgreSQL query in production — what tools did you use, what was the root cause, and how did you verify the fix without impacting production traffic?</summary>

**What the interviewer is looking for:**
- Systematic debugging approach (not guessing)
- Familiarity with PostgreSQL diagnostic tools (pg_stat_statements, EXPLAIN ANALYZE, pg_stat_user_tables)
- Awareness of production safety (testing on replicas, concurrent index creation, monitoring after deploy)
- Understanding of root causes beyond "add an index"

**Suggested structure (STAR format):**

1. **Situation**: What was the system, what was the symptom? (e.g., "API p99 latency spiked from 50ms to 2s during peak traffic")
2. **Task**: What was your responsibility? (e.g., "On-call, needed to diagnose and fix within the SLA")
3. **Action**: Walk through your debugging process step by step:
   - How you identified the slow query (pg_stat_statements, APM tool, slow query log)
   - How you analyzed it (EXPLAIN ANALYZE on a replica)
   - What the root cause was (missing index, stale statistics, plan regression, table bloat, lock contention)
   - How you applied the fix safely (CREATE INDEX CONCURRENTLY, ANALYZE, config change)
   - How you verified it worked (compared EXPLAIN plans before/after, monitored p99 in dashboard)
4. **Result**: Quantify the improvement (e.g., "p99 dropped from 2s to 40ms, resolved within 30 minutes")

**Key points to hit:**
- Mention specific tools: pg_stat_statements, EXPLAIN (ANALYZE, BUFFERS), pg_stat_activity
- Show you tested on a read replica before touching production
- Explain why you chose that fix over alternatives
- If possible, mention what you did to prevent recurrence (added monitoring, adjusted autovacuum, added the query pattern to CI test suite)

**Example outline to personalize:**
"Our order search API started timing out during a flash sale. pg_stat_statements showed the search query went from 20ms average to 3s. EXPLAIN ANALYZE on a replica revealed a Seq Scan on the orders table — the planner had switched plans after an ANALYZE updated statistics with the new data distribution. I created a composite index CONCURRENTLY on (customer_id, status, created_at) and ran ANALYZE. The query dropped to 5ms. I added a monitoring alert for queries exceeding 500ms and tuned the statistics target on the status column to prevent future plan regressions."

</details>

<details>
<summary>27. Describe a PostgreSQL migration that was particularly risky or complex — what made it challenging (table size, zero-downtime requirement, data transformation), how did you execute it safely, and what would you do differently?</summary>

**What the interviewer is looking for:**
- Understanding of PostgreSQL locking behavior and why naive DDL is dangerous
- Knowledge of safe migration patterns (covered in question 18)
- Risk assessment and mitigation planning
- Ability to learn from experience and improve processes

**Suggested structure:**

1. **Context**: What migration was needed and why? (schema change, data model evolution, performance improvement)
2. **What made it risky**: Be specific — table size (e.g., 200M rows), zero-downtime requirement, data transformation that couldn't be rolled back, multi-step coordination across services
3. **How you planned it**: Testing strategy (staging environment with production-sized data), runbook, rollback plan, communication with the team
4. **How you executed it**: Step-by-step — what ran in what order, what tools you used, how you monitored during the migration
5. **What went well / what you'd do differently**: Honest reflection

**Key points to hit:**
- Name the specific safe patterns you used: CREATE INDEX CONCURRENTLY, CHECK NOT VALID + VALIDATE, batch backfills, dual-write period
- Mention how you handled the rollback plan (could you roll back at each step?)
- If something went wrong, own it and explain what you learned
- Show awareness of the locking implications of each DDL statement

**Example outline to personalize:**
"We needed to change our events table's primary key from a serial integer to a UUID to support a multi-region architecture. The table had 150M rows with zero-downtime requirement. The challenge: you can't ALTER a primary key without an AccessExclusiveLock.

Our approach:
1. Added a new `uuid_id` column with a default (PG 11+ fast default)
2. Backfilled UUIDs in batches of 50K rows with commits between batches
3. Created a unique index CONCURRENTLY on `uuid_id`
4. Updated the application to write both `id` and `uuid_id` (dual-write)
5. Switched reads to use `uuid_id`
6. In a brief maintenance window (< 1 second), swapped the primary key constraint

What I'd do differently: I'd use pg_repack to do the primary key swap online instead of the brief maintenance window, and I'd add better progress monitoring for the backfill step."

</details>

<details>
<summary>28. Tell me about a time you dealt with a PostgreSQL operational issue (replication lag, disk filling up, connection exhaustion, or bloat) — what were the symptoms, how did you diagnose and resolve it, and what monitoring or configuration changes did you add to prevent recurrence?</summary>

**What the interviewer is looking for:**
- Incident response skills — calm, systematic approach under pressure
- Deep understanding of PostgreSQL operational characteristics
- Ability to move from symptom to root cause to fix to prevention
- Monitoring and observability mindset

**Suggested structure:**

1. **Symptoms**: How did you discover the issue? (alert, user report, dashboard anomaly)
2. **Diagnosis**: What diagnostic steps did you take? What tools did you use? How did you narrow down the root cause?
3. **Resolution**: What was the immediate fix? What was the longer-term fix?
4. **Prevention**: What monitoring, configuration, or process changes did you add to catch this earlier or prevent it entirely?

**Key points to hit by scenario type:**

*Connection exhaustion* (most common for backend engineers):
- Symptoms: `FATAL: too many connections`, application errors
- Diagnosis: pg_stat_activity showing idle-in-transaction sessions, connection distribution by app
- Fix: terminate leaked connections, fix connection pool config
- Prevention: idle_in_transaction_session_timeout, pool monitoring, connection leak detection in CI
- (Concepts from questions 8 and 24)

*Table bloat / disk filling up:*
- Symptoms: disk usage alerts, slow sequential scans, table size growing without row count increase
- Diagnosis: pg_stat_user_tables dead tuples, pgstattuple, long-running transactions blocking VACUUM
- Fix: kill long transactions, run VACUUM, use pg_repack for severe cases
- Prevention: autovacuum tuning, idle-in-transaction timeout, disk usage monitoring
- (Concepts from questions 5, 21, and 25)

*Replication lag:*
- Symptoms: read replicas returning stale data, monitoring showing increasing lag
- Diagnosis: pg_stat_replication on primary, check for long-running queries on replica blocking WAL replay, check network/disk I/O on replica
- Fix: cancel blocking queries on replica, increase replica resources, enable hot_standby_feedback
- Prevention: max_standby_streaming_delay tuning, monitoring replication lag with alerts at thresholds

**Example outline to personalize:**
"We got paged at 2 AM for disk space alerts on our primary PostgreSQL server — 90% full, growing fast. Our events table had gone from 50GB to 180GB in a week despite stable write volume. pg_stat_user_tables showed 800M dead tuples. A background analytics job was running 4-hour transactions on the primary, holding back the VACUUM horizon. I terminated the long-running transaction, ran VACUUM on the events table (which took 45 minutes), and the dead tuple count dropped to near zero. The table was still 180GB (VACUUM doesn't shrink the file), but new inserts reused the space.

For prevention: I moved the analytics job to a read replica, set idle_in_transaction_session_timeout to 60s, added alerts on dead_tuple_ratio > 10% and xid_age > 500M, and tuned autovacuum for the events table with a 1% scale factor."

</details>
