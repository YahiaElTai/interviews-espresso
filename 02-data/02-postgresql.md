# PostgreSQL

> **43 questions** — 13 theory, 30 practical

- PostgreSQL vs MySQL: architectural differences (MVCC, extensibility, type system), strengths, weaknesses, and when MySQL is the better choice
- MVCC internals: tuple versioning, dead tuples, and the VACUUM dependency
- WAL (Write-Ahead Log): crash recovery, replication, and point-in-time restore
- Index types: B-tree, GIN, GiST, BRIN, partial, expression, covering (INCLUDE)
- VACUUM and autovacuum: transaction ID wraparound, tuning, and warning signs
- Locking: row-level locks, SELECT FOR UPDATE, advisory locks, deadlock detection, and pg_locks diagnostics
- Transaction isolation levels: Read Committed, Repeatable Read, Serializable
- Table partitioning: declarative (range, list, hash), partition pruning, and maintenance
- Replication: streaming vs logical, synchronous vs async, replication slots and slot bloat
- Connection model: process-per-connection architecture, max_connections limits, application-level pool sizing, idle-in-transaction dangers
- PgBouncer: transaction vs session pooling, pool sizing, and prepared statement pitfalls
- Production tuning: shared_buffers, work_mem, effective_cache_size, checkpoint and WAL settings for OLTP workloads
- CTEs: recursive queries for hierarchical data, optimization fences, and inlining
- Window functions: ROW_NUMBER, RANK, LAG/LEAD, running totals, gap detection
- JSONB: querying, GIN indexing, containment operators, when to use vs relational columns
- EXPLAIN ANALYZE: reading query plans, scan types, join strategies, and bottleneck identification
- Materialized views: creation, concurrent refresh, indexing, and use cases
- Migrations: zero-downtime patterns (concurrent index creation, NOT NULL via constraints), tooling (node-pg-migrate, Prisma migrate), rollback strategies, data migrations
- Row-Level Security (RLS) for multi-tenant applications
- Data type choices: timestamptz vs timestamp, text vs varchar(n), identity vs serial, UUID generation, money type antipattern (use integer cents or numeric)
- Common SQL antipatterns: NOT IN with NULLs (use NOT EXISTS), BETWEEN with timestamps (double-counting at boundaries)
- Backup and recovery: pg_dump vs WAL-based backup (pgBackRest/WAL-G), PITR setup and verification
- Production monitoring and diagnostics: pg_stat_activity, pg_stat_statements, pg_stat_user_tables, pg_locks — diagnosing slow queries, connection exhaustion, lock contention, and bloat

---

## Foundational

<details>
<summary>1. How does PostgreSQL differ from MySQL architecturally — what makes PostgreSQL's MVCC implementation, extensibility model, and type system different, when does PostgreSQL genuinely win over MySQL, and when would you pick MySQL or another database instead?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How does PostgreSQL's MVCC (Multi-Version Concurrency Control) work under the hood — what is tuple versioning, how are dead tuples created by updates and deletes, why does this design enable readers to never block writers, and why does it create a dependency on VACUUM?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What is the Write-Ahead Log (WAL) and why is it fundamental to PostgreSQL — how does WAL provide crash recovery guarantees, how does replication build on WAL, and how does WAL archiving enable point-in-time recovery?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. What PostgreSQL index types exist beyond B-tree (GIN, GiST, BRIN, partial, expression, covering with INCLUDE) — what problem does each solve, when would you choose each over a plain B-tree, and when do specialized indexes hurt more than they help?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why is VACUUM critical for PostgreSQL's health and what happens without it — how does autovacuum work, what is transaction ID wraparound and why is it an existential threat, what are the warning signs that VACUUM is falling behind, and how do you tune autovacuum for high-write tables?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How do PostgreSQL's transaction isolation levels (Read Committed, Repeatable Read, Serializable) differ in practice — what anomalies does each prevent, why is Read Committed the default, when do you need Serializable and what's the performance cost, and how does PostgreSQL implement these via MVCC snapshots?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How does PostgreSQL's locking system work and why does it offer multiple lock granularities -- what are the differences between row-level locks, table-level locks, and advisory locks, when do you use SELECT FOR UPDATE vs SELECT FOR SHARE, how does PostgreSQL detect and resolve deadlocks, and when are advisory locks the right tool vs row-level locks?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why would you partition a PostgreSQL table and what are the tradeoffs — how does declarative partitioning work (range, list, hash), how does partition pruning speed up queries, and what maintenance and query-planning challenges do partitioned tables introduce?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How does PostgreSQL replication work and what are the tradeoffs of each mode — streaming vs logical replication (what each can and can't do), synchronous vs asynchronous (durability vs latency), what are replication slots and how can they cause slot bloat that fills your disk?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why does PgBouncer exist and what pooling mode should you use — what's the difference between transaction pooling and session pooling, how do you size the pool correctly relative to PostgreSQL's max_connections, and what breaks with prepared statements in transaction pooling mode?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why does PostgreSQL's process-per-connection architecture create a hard ceiling on concurrency -- what resources does each backend process consume, why is max_connections typically set much lower than the number of concurrent application threads, what makes idle-in-transaction sessions particularly dangerous (holding locks, preventing VACUUM, consuming connection slots), and how should you size application-level connection pools independently of whether you use PgBouncer?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What are the critical PostgreSQL configuration parameters for OLTP workloads — how do shared_buffers, work_mem, effective_cache_size, and checkpoint/WAL settings interact, what are reasonable starting values for different server sizes, and what misconfiguration symptoms tell you each parameter is wrong?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why do data type choices matter in PostgreSQL schema design — when should you use timestamptz vs timestamp, text vs varchar(n), identity columns vs serial, why is the money type a trap (and what to use instead for currency), and what are the UUID generation options (gen_random_uuid, uuid-ossp, pgcrypto) and their tradeoffs?</summary>

<!-- Answer will be added later -->

</details>

## Practical — SQL & Query Patterns

<details>
<summary>14. Write a recursive CTE that traverses a hierarchical structure (e.g., organization chart or category tree with parent_id) — show the SQL, explain how the recursive and non-recursive terms work together, and demonstrate when CTEs are optimization fences vs when they get inlined (PostgreSQL 12+ behavior with the MATERIALIZED/NOT MATERIALIZED hints)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Use window functions for row numbering and ranking -- show ROW_NUMBER for deduplication or pagination, RANK vs DENSE_RANK for ranking with ties, and explain the PARTITION BY and ORDER BY interaction that controls how windows are defined. Demonstrate with SQL on a realistic table</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Use window functions for row comparison and running calculations -- show LAG/LEAD for comparing rows to their neighbors and detecting gaps in sequences, and a running total using a frame clause (ROWS BETWEEN). Explain how frame specifications (ROWS vs RANGE, UNBOUNDED PRECEDING vs CURRENT ROW) change the result</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Query JSONB data in PostgreSQL — show containment operators (@>, ?), path operators (->>, #>>), indexing with GIN (jsonb_path_ops vs default operator class), and demonstrate a query that benefits from a GIN index. When should you store data as JSONB vs in normalized relational columns, and what queries become painful with JSONB?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Given a slow query — walk through EXPLAIN ANALYZE output step by step: identify the scan type (Seq Scan, Index Scan, Index Only Scan, Bitmap Scan), understand join strategies (Nested Loop, Hash Join, Merge Join), read actual vs estimated rows to spot planner misestimates, and identify the single biggest bottleneck in the plan</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Create a materialized view for a dashboard query that joins multiple tables and aggregates data — show the CREATE MATERIALIZED VIEW, add indexes on it, demonstrate REFRESH MATERIALIZED VIEW CONCURRENTLY (and why you need a unique index for concurrent refresh), and explain when materialized views are the right solution vs other caching approaches</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Create specialized indexes for real scenarios — show a partial index (only active users), an expression index (lower(email) for case-insensitive lookup), and a covering index with INCLUDE (to enable index-only scans). For each, demonstrate the query it optimizes and show EXPLAIN proving it's being used</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Write SQL that demonstrates the practical difference between Read Committed and Repeatable Read isolation — show a scenario where Read Committed sees a phantom row that Repeatable Read doesn't, and a scenario where Repeatable Read throws a serialization error that requires application-level retry logic</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Demonstrate two common SQL antipatterns that cause subtle bugs — show why NOT IN fails silently when the subquery contains NULLs (and the NOT EXISTS alternative), and why BETWEEN with timestamps double-counts rows at boundaries (and the >= / < alternative). For each, show the broken query, explain the failure mode, and show the correct version</summary>

<!-- Answer will be added later -->

</details>

## Practical — Schema & Migrations

<details>
<summary>23. Walk through zero-downtime migration patterns for a production table with 100M rows — show how to add a column with a default without locking (PostgreSQL 11+ fast default vs older backfill pattern), add a NOT NULL constraint via CHECK constraint first, and create an index with CREATE INDEX CONCURRENTLY. Explain what each approach avoids and what breaks if you use naive DDL</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Implement Row-Level Security for a multi-tenant SaaS application — show the policy creation for tenant_id-based isolation, how to set the current tenant via session variables (set_config/current_setting), demonstrate that queries automatically filter by tenant, and explain the pitfalls (forgetting to enable RLS, FORCE ROW SECURITY for table owners)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Set up declarative range partitioning on a large events table partitioned by month — show the CREATE TABLE with partitions, demonstrate partition pruning with EXPLAIN, show how to add and detach partitions for data lifecycle management, and explain what queries break or slow down with partitioned tables</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Design a schema using proper PostgreSQL data types — show when to use timestamptz (and why timestamp without time zone is almost always wrong), why text is preferred over varchar(n) for most cases, how identity columns replace serial, and the best approach for UUID primary keys with performance considerations</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Compare migration tooling: write the same migration (add table, add index, backfill data) using node-pg-migrate and Prisma migrate — show both, explain the tradeoffs (raw SQL control vs schema diff approach), demonstrate a rollback strategy for each, and explain how you handle data migrations that can't be in a DDL transaction</summary>

<!-- Answer will be added later -->

</details>

## Practical — Production Configuration

<details>
<summary>28. Tune PostgreSQL for a production OLTP workload on a server with 32GB RAM and 8 cores — show the postgresql.conf settings for shared_buffers, work_mem, effective_cache_size, maintenance_work_mem, checkpoint_completion_target, and max_wal_size. Explain why each value is what it is and what symptoms you'd see if each parameter is misconfigured</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Set up PgBouncer in transaction pooling mode between your Node.js application and PostgreSQL — show the pgbouncer.ini configuration, explain how to size the pool (max_client_conn vs default_pool_size vs PostgreSQL max_connections), what happens when the pool is exhausted, and how to handle the prepared statement limitation</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Set up PostgreSQL streaming replication with one synchronous and one asynchronous standby — show the configuration for primary and standbys, explain replication slots and when to use them, demonstrate promoting a standby to primary, and explain why synchronous replication adds latency but guarantees zero data loss</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. Set up WAL-based continuous backup using pgBackRest or WAL-G — show the configuration for WAL archiving and base backups, walk through restoring to a specific point in time after accidental data deletion, and compare this approach to pg_dump for different recovery scenarios</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Tune autovacuum for a high-write table that's generating dead tuples faster than the default settings can clean up — show the per-table autovacuum settings (autovacuum_vacuum_scale_factor, autovacuum_vacuum_cost_delay), explain how to monitor whether autovacuum is keeping up, and what emergency steps to take when a table is approaching transaction ID wraparound</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>33. Set up a production monitoring checklist using PostgreSQL system views — show the queries for pg_stat_activity (active connections, long-running queries), pg_stat_statements (top queries by total time and calls), pg_stat_user_tables (sequential scans, dead tuple ratio), and pg_locks (lock contention). Explain what healthy vs unhealthy numbers look like for each</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>34. Your API's p99 latency jumped from 50ms to 3 seconds — walk through diagnosing the PostgreSQL side: identify the slow query using pg_stat_statements, run EXPLAIN ANALYZE to find the bottleneck (missing index, stale statistics, seq scan on a large table), check if the query plan changed due to a statistics update or table growth, and apply the fix</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>35. Several transactions are blocked and your application is timing out — walk through diagnosing lock contention: use pg_locks joined with pg_stat_activity to find the blocking query, determine whether it's a row lock, advisory lock, or DDL lock, decide whether to wait or terminate the blocking process, and explain how to prevent this from recurring</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>36. Your application suddenly can't connect to PostgreSQL and you see "too many connections" errors — walk through the diagnosis: check pg_stat_activity for idle connections, idle-in-transaction sessions, and connection leaks. Explain how PgBouncer misconfiguration, missing application-level pool limits, or long-running transactions cause this, and the immediate vs long-term fixes</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>37. A table that was 2GB is now 20GB even though the row count hasn't changed significantly — walk through diagnosing table bloat: check pg_stat_user_tables for dead tuples, determine why autovacuum isn't keeping up (long-running transactions holding back the visibility horizon, insufficient worker settings), and show how to reclaim space (VACUUM FULL as last resort and its locking implications, pg_repack as an alternative)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>38. Your streaming replica has fallen hours behind and the replication slot is preventing WAL cleanup, filling the primary's disk — walk through the diagnosis using pg_stat_replication and pg_replication_slots, explain how to determine if the replica can catch up or needs to be rebuilt, how to safely drop or advance a stuck replication slot, and what prevents this in the future</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>39. PostgreSQL's WAL directory is growing rapidly and checkpoints are taking too long — walk through checking checkpoint frequency in the server logs, identify whether it's caused by too-low max_wal_size forcing frequent checkpoints, high write volume, or WAL archiving falling behind. Show the relevant log entries and configuration changes to fix each cause</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>40. Tell me about a time you had to optimize a slow PostgreSQL query in production — what tools did you use, what was the root cause, and how did you verify the fix without impacting production traffic?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>41. Describe a PostgreSQL migration that was particularly risky or complex — what made it challenging (table size, zero-downtime requirement, data transformation), how did you execute it safely, and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>42. Tell me about a time you dealt with a PostgreSQL operational issue (replication lag, disk filling up, connection exhaustion, or bloat) — what were the symptoms, how did you diagnose and resolve it, and what monitoring or configuration changes did you add to prevent recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>43. Describe a time you made a significant PostgreSQL architectural decision (partitioning, replication topology, managed vs self-hosted, or data modeling approach) — what drove the decision, what tradeoffs did you accept, and how did it work out?</summary>

<!-- Answer framework will be added later -->

</details>
