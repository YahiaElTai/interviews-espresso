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

</details><details>
<summary>8. What are the main database caching patterns (cache-aside, write-through, write-behind) -- how does each work, what consistency tradeoffs does each make, why is cache invalidation considered one of the hardest problems, and when is using a read replica a better alternative to adding a caching layer?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How does database replication work and when would you choose each topology — what are the tradeoffs between single-leader, multi-leader, and leaderless replication, how do synchronous vs asynchronous replication differ in durability and latency, and what challenges arise with multi-region topologies?</summary>

<!-- Answer will be added later -->

</details><details>
<summary>10. When should you shard a database and what are the tradeoffs of each sharding strategy — how do range-based, hash-based, and directory-based sharding work, what happens to cross-shard queries and transactions, why is sharding usually the last resort after other scaling approaches, and how does federation (functional partitioning — splitting by domain into separate databases) compare as a simpler scaling step before full sharding?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. What are the practical differences between consistency models (strong, eventual, read-your-writes, causal) -- how does each one affect what a user experiences in a distributed application, when is eventual consistency actually fine and when does it cause real bugs, and how do you choose the right consistency model for different parts of the same system?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Why are schema migrations dangerous on production databases — what locking behavior do DDL operations (ALTER TABLE, CREATE INDEX) trigger, how do you add columns and indexes to large tables without downtime, and what strategies prevent migrations from causing outages?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Query Writing & Optimization

<details>
<summary>13. Given a users table, orders table, and products table — design a composite index strategy that covers the 5 most common queries (user lookup, recent orders, orders by status, product search, user's order history). Show the CREATE INDEX statements, explain the column order choices, and demonstrate when a query would vs wouldn't use each index</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Given a slow query that joins three tables and filters on multiple conditions — walk through the EXPLAIN ANALYZE output line by line, identify the bottleneck (sequential scan, nested loop join, large sort), propose a fix (index, query rewrite, or materialized view), and verify the improvement with EXPLAIN ANALYZE again</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. You discover your Node.js API is executing 50 SQL queries for a single endpoint that returns a list of orders with their items — show the N+1 problem in ORM code (Prisma or TypeORM), demonstrate the fix using eager loading or joins, and explain what to watch for when the fix over-fetches data</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Implement cursor-based pagination for a table with 10 million rows sorted by created_at — show the API contract, the SQL query using keyset pagination (WHERE created_at < ?), how to encode/decode the cursor, and demonstrate the performance difference vs OFFSET with EXPLAIN ANALYZE</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. A financial application needs to transfer funds between two accounts — implement the transaction with the correct isolation level, show how you handle concurrent transfers to the same account, explain why you chose that isolation level over others, and what happens if you use a weaker one</summary>

<!-- Answer will be added later -->

</details>

## Practical — Schema & Data Access

<details>
<summary>18. Design a multi-tenant database schema for a SaaS application — compare schema-per-tenant vs shared-schema with tenant_id column vs row-level security, show the SQL for your chosen approach, and explain the tradeoffs in isolation, performance, and operational complexity as tenant count grows</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. You need to add a non-null column with a default value and a new index to a table with 50 million rows in production — walk through the migration steps that avoid locking the table, show the SQL commands, explain why a naive ALTER TABLE would cause downtime, and what tools or techniques help (CREATE INDEX CONCURRENTLY, backfill strategies)</summary>

<!-- Answer will be added later -->

</details>

## Practical — Production Operations<details>
<summary>20. Your API response times have degraded from 50ms to 2 seconds over the past week — walk through the systematic database diagnosis: checking pg_stat_activity for blocked queries, identifying the slow query with pg_stat_statements, running EXPLAIN ANALYZE, checking for missing indexes, bloated tables, or lock contention, and applying the fix</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. How does connection pooling work at the application level (pg pool, Prisma connection pool) vs with an external pooler like PgBouncer — what are the tradeoffs between session mode, transaction mode, and statement mode, how do you size the pool correctly, and what goes wrong when you get it wrong (connection exhaustion, idle timeouts, SSL/TLS overhead)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. What database metrics should you monitor in production and how do you act on them — walk through setting up alerts for slow queries (pg_stat_statements), connection pool saturation, replication lag, and table bloat, explain what each metric tells you, and what remediation steps you take when each threshold is breached?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>23. Tell me about a time you had to scale a database to handle significantly more traffic — what approach did you take (read replicas, caching, query optimization, sharding), what tradeoffs did you face, and what was the outcome?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>24. Describe a time you debugged a production database issue (slow queries, connection exhaustion, replication lag, or data corruption) — what were the symptoms, how did you diagnose the root cause, and what was the fix?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Tell me about a time you chose between SQL and NoSQL for a project — what were the requirements, what influenced your decision, and looking back, was it the right call?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>26. Describe a schema migration that went wrong or was particularly risky — what made it dangerous, how did you handle it, and what did you learn about safe migration practices?</summary>

<!-- Answer framework will be added later -->

</details>
