# Databases

> **37 questions** — 19 theory, 18 practical

- SQL vs NoSQL: relational (PostgreSQL, MySQL) vs document (MongoDB) vs key-value (DynamoDB) vs wide-column (Cassandra) — schema flexibility, consistency, query power, and when each model wins
- Specialized database types: time-series (TimescaleDB, InfluxDB), columnar (ClickHouse), graph (Neo4j) — when relational databases are the wrong fit
- ACID properties: atomicity, consistency, isolation, durability — what each guarantees and how different databases implement them
- Data modeling: normalization, denormalization, and schema design for multi-tenant systems
- Index internals: B-tree, hash, GIN/GiST, LSM trees, and write-amplification tradeoffs
- Query optimization: EXPLAIN ANALYZE, N+1 queries, composite index strategy
- Transactions, isolation levels, and MVCC (PostgreSQL vs MySQL)
- Locking: row-level locks, SELECT FOR UPDATE, advisory locks, deadlock detection and prevention
- Connection pooling: application-level (pg pool, Prisma) vs external (PgBouncer), SSL/TLS connections
- ORMs vs query builders vs raw SQL: Prisma, TypeORM, Knex — abstraction tradeoffs, performance pitfalls, migration tooling
- Caching patterns: cache-aside, write-through, write-behind, cache invalidation strategies, read replica as cache alternative
- Pagination: offset vs cursor-based, keyset pagination on large tables, performance implications
- Replication: single-leader, multi-leader, leaderless, sync vs async, multi-region topologies
- Sharding strategies: range, hash, directory-based — shard key selection, hotspot avoidance, cross-shard queries, resharding challenges; federation (functional partitioning) as a simpler alternative
- CAP theorem: what it actually means, why "pick two" is misleading, how PostgreSQL, MongoDB, DynamoDB, and CockroachDB make different tradeoffs during partitions
- Consistency models: strong, eventual, read-your-writes, causal — practical implications for application design and user experience
- Deployment options: self-hosted vs managed (RDS/Cloud SQL/AlloyDB) vs containerized
- High availability: automatic failover, quorum-based consensus, read replicas, health checks and promotion
- Backup and recovery: logical vs physical backups, point-in-time recovery, backup verification and restore testing
- Database monitoring: slow queries, connection pool saturation, replication lag, bloat tracking
- Safe schema migrations: locking behavior, adding columns/indexes on large tables, versioned migration tooling (Prisma, Flyway), rollback strategies, data vs schema migrations

---

## Foundational

<details>
<summary>1. Why would you choose a relational database (PostgreSQL, MySQL) vs a document store (MongoDB) vs a key-value store (DynamoDB) -- what are the real tradeoffs in schema flexibility, consistency guarantees, and query power? Beyond these common categories, when would you reach for a specialized database type like time-series (TimescaleDB), columnar (ClickHouse), or graph (Neo4j) instead of forcing a relational database to do the job?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How does data modeling work in relational databases -- what are the tradeoffs between normalization and denormalization, when does each approach win in practice, what are the signs that you've over-normalized or over-denormalized, and how do these choices affect query performance and maintenance as the schema evolves?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. How does the CAP theorem actually apply to real database choices — what do PostgreSQL, MongoDB, DynamoDB, and CockroachDB each sacrifice during a network partition, and why is the "pick two out of three" framing misleading?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. What do the ACID properties (atomicity, consistency, isolation, durability) actually guarantee, how do relational databases like PostgreSQL enforce each one, and how do NoSQL databases like MongoDB and DynamoDB make different ACID tradeoffs -- why would you choose a database that relaxes some of these guarantees?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>5. How do database indexes work internally — what are the structural differences between B-tree, hash, GIN/GiST, and LSM tree indexes, why does each exist for different workloads, and what is write amplification and when does it matter?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How do you approach query optimization systematically — how do you read an EXPLAIN ANALYZE output to identify bottlenecks, what causes N+1 queries and why are they insidious at scale, and how do you design a composite index strategy that covers your most important queries?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do database transactions and isolation levels work — what does each isolation level (Read Uncommitted through Serializable) protect against, how does MVCC implement isolation differently in PostgreSQL vs MySQL InnoDB, and why does choosing the wrong isolation level cause either phantom reads or unnecessary blocking?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How does database locking work and why does it matter for correctness — what are row-level locks, what does SELECT FOR UPDATE do and when do you need it, what are advisory locks used for, and how do deadlocks happen and how do databases detect and resolve them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why is connection pooling essential for production databases — what's the difference between application-level pooling (pg module pool, Prisma connection pool) and external poolers (PgBouncer), how do they differ in connection multiplexing, and what does SSL/TLS add in terms of overhead and connection management?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. What are the tradeoffs between ORMs (Prisma, TypeORM), query builders (Knex), and raw SQL — when does each abstraction level make sense, what performance pitfalls do ORMs introduce (lazy loading, N+1, inefficient generated SQL), and how does migration tooling differ between them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why do pagination approaches differ in performance — what are the tradeoffs between offset, cursor-based, and keyset pagination, why does offset pagination degrade on large tables, how does keyset pagination maintain consistent performance, and what breaks when the underlying data changes between page requests?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What are the main database caching patterns (cache-aside, write-through, write-behind) -- how does each work, what consistency tradeoffs does each make, why is cache invalidation considered one of the hardest problems, and when is using a read replica a better alternative to adding a caching layer?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. How does database replication work and when would you choose each topology — what are the tradeoffs between single-leader, multi-leader, and leaderless replication, how do synchronous vs asynchronous replication differ in durability and latency, and what challenges arise with multi-region topologies?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. What are the practical differences between consistency models (strong, eventual, read-your-writes, causal) -- how does each one affect what a user experiences in a distributed application, when is eventual consistency actually fine and when does it cause real bugs, and how do you choose the right consistency model for different parts of the same system?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. When should you shard a database and what are the tradeoffs of each sharding strategy — how do range-based, hash-based, and directory-based sharding work, what happens to cross-shard queries and transactions, why is sharding usually the last resort after other scaling approaches, and how does federation (functional partitioning — splitting by domain into separate databases) compare as a simpler scaling step before full sharding?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. How do database deployment options compare — what are the tradeoffs between self-hosted, managed services (RDS, Cloud SQL, AlloyDB), and containerized databases, when does the convenience of managed services outweigh the cost premium, and what operational responsibilities shift vs remain with your team?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How does database high availability work — what is automatic failover, how do quorum-based consensus mechanisms prevent split-brain, how do read replicas differ from HA replicas, and what role do health checks and promotion play in minimizing downtime?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. What are the tradeoffs between logical and physical database backups — how does point-in-time recovery (PITR) work, why do you need WAL archiving for PITR, and why is backup verification and restore testing critical even when you have automated backups running?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Why are schema migrations dangerous on production databases — what locking behavior do DDL operations (ALTER TABLE, CREATE INDEX) trigger, how do you add columns and indexes to large tables without downtime, and what strategies prevent migrations from causing outages?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Query Writing & Optimization

<details>
<summary>20. Given a users table, orders table, and products table — design a composite index strategy that covers the 5 most common queries (user lookup, recent orders, orders by status, product search, user's order history). Show the CREATE INDEX statements, explain the column order choices, and demonstrate when a query would vs wouldn't use each index</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Given a slow query that joins three tables and filters on multiple conditions — walk through the EXPLAIN ANALYZE output line by line, identify the bottleneck (sequential scan, nested loop join, large sort), propose a fix (index, query rewrite, or materialized view), and verify the improvement with EXPLAIN ANALYZE again</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. You discover your Node.js API is executing 50 SQL queries for a single endpoint that returns a list of orders with their items — show the N+1 problem in ORM code (Prisma or TypeORM), demonstrate the fix using eager loading or joins, and explain what to watch for when the fix over-fetches data</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Implement cursor-based pagination for a table with 10 million rows sorted by created_at — show the API contract, the SQL query using keyset pagination (WHERE created_at < ?), how to encode/decode the cursor, and demonstrate the performance difference vs OFFSET with EXPLAIN ANALYZE</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. A financial application needs to transfer funds between two accounts — implement the transaction with the correct isolation level, show how you handle concurrent transfers to the same account, explain why you chose that isolation level over others, and what happens if you use a weaker one</summary>

<!-- Answer will be added later -->

</details>

## Practical — Schema & Data Access

<details>
<summary>25. Design a multi-tenant database schema for a SaaS application — compare schema-per-tenant vs shared-schema with tenant_id column vs row-level security, show the SQL for your chosen approach, and explain the tradeoffs in isolation, performance, and operational complexity as tenant count grows</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. You need to add a non-null column with a default value and a new index to a table with 50 million rows in production — walk through the migration steps that avoid locking the table, show the SQL commands, explain why a naive ALTER TABLE would cause downtime, and what tools or techniques help (CREATE INDEX CONCURRENTLY, backfill strategies)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Set up connection pooling for a Node.js application with Prisma connecting to PostgreSQL through PgBouncer — show the configuration for both layers, explain why you need both an application-level pool and an external pooler, what happens when the pool is exhausted, and how to size the pools correctly</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Implement a "claim the next available job" pattern using SELECT FOR UPDATE SKIP LOCKED — show the SQL, explain why a naive SELECT followed by UPDATE has a race condition, how SKIP LOCKED prevents deadlocks by skipping already-locked rows, and what the alternative approaches are (advisory locks, optimistic locking)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Implement the same data access pattern (create user, find with filters, update, paginated list) using Prisma, a query builder (Knex), and raw SQL — compare the three for readability, type safety, and query efficiency, and show a case where the ORM generates a suboptimal query that the query builder or raw SQL handles better</summary>

<!-- Answer will be added later -->

</details>

## Practical — Production Operations

<details>
<summary>30. Set up read/write splitting for a Node.js application using a managed PostgreSQL service (RDS or Cloud SQL) with read replicas -- show the application-level routing logic that directs writes to the primary and reads to replicas, explain how you handle failover when the managed service promotes a replica (connection drops, in-flight transactions, DNS propagation delay), and what stale-read scenarios arise from replication lag</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. When using a managed database service (RDS, Cloud SQL, AlloyDB), walk through the actual procedure to recover from accidental data deletion using point-in-time recovery -- what steps do you take, what's the expected downtime, how do RTO and RPO targets affect your backup configuration choices, and why should you practice restore testing even when the provider handles automated backups?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Set up monitoring for a production PostgreSQL database — show what dashboards and alerts you'd configure for slow queries (pg_stat_statements), connection pool saturation, replication lag, and table/index bloat, and explain what each metric tells you and at what thresholds you'd alert</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>33. Your API response times have degraded from 50ms to 2 seconds over the past week — walk through the systematic database diagnosis: checking pg_stat_activity for blocked queries, identifying the slow query with pg_stat_statements, running EXPLAIN ANALYZE, checking for missing indexes, bloated tables, or lock contention, and applying the fix</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>34. Tell me about a time you had to scale a database to handle significantly more traffic — what approach did you take (read replicas, caching, query optimization, sharding), what tradeoffs did you face, and what was the outcome?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>35. Describe a time you debugged a production database issue (slow queries, connection exhaustion, replication lag, or data corruption) — what were the symptoms, how did you diagnose the root cause, and what was the fix?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Tell me about a time you chose between SQL and NoSQL for a project — what were the requirements, what influenced your decision, and looking back, was it the right call?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>37. Describe a schema migration that went wrong or was particularly risky — what made it dangerous, how did you handle it, and what did you learn about safe migration practices?</summary>

<!-- Answer framework will be added later -->

</details>
