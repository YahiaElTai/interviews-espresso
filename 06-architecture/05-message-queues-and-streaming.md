# Message Queues & Streaming

> **25 questions** — 13 theory, 9 practical, 3 experience

- Message queues vs event streams: fundamental differences and when to use each
- Delivery guarantees: at-most-once, at-least-once, exactly-once tradeoffs and why exactly-once is hard
- When to use messaging vs alternatives: HTTP calls, database-backed jobs, cron, task queues (Bull/BullMQ) as a middle ground — signs a queue adds value vs signs it's overkill
- Queue vs topic/pub-sub semantics: competing consumers vs fan-out, SNS+SQS pattern, RabbitMQ exchanges, Kafka topics, Pub/Sub subscriptions
- Kafka architecture: append-only log, partitioning, partition key strategy (per-entity, per-tenant, hot partitions), ordering guarantees, rebalancing
- Ordering vs parallelism: why you can't have both easily, partition-level ordering, and when to relax ordering
- Consumer groups: Kafka partition assignment, Pub/Sub subscriptions, SQS visibility timeout, failure modes
- Offset management and replay: committed offsets, offset reset policies (earliest/latest), rewinding consumers, compacted topics for latest-state lookups
- Idempotent message processing: deduplication IDs, idempotent writes, transactional outbox pattern
- Backpressure and flow control: consumer lag, prefetch limits, pull vs push consumers, visibility timeout tuning
- Dead letter queues: routing failed messages, triage strategies, replay and reprocessing
- Message serialization: JSON vs Avro vs Protobuf, schema evolution, schema registries

---

## Foundational

<details>
<summary>1. What is the fundamental difference between message queues and event streams — how do queues (SQS, RabbitMQ) differ from streams (Kafka, Pub/Sub) in message retention, replay capability, and consumer model, and when would you choose each one?</summary>

**Message queues** deliver a message to exactly one consumer, then delete it. The broker is a middleman — once a message is consumed and acknowledged, it's gone. Think of it as a work distribution mechanism.

**Event streams** are an append-only log. Messages (events) are retained for a configurable period (or indefinitely with compaction). Multiple consumers can read the same stream independently, each tracking their own position (offset). Think of it as a shared, ordered history.

| Dimension | Queue (SQS, RabbitMQ) | Stream (Kafka, Pub/Sub) |
|---|---|---|
| **Retention** | Deleted after consumption/ack | Retained for configured period (hours to forever) |
| **Replay** | No — once consumed, it's gone | Yes — consumers can rewind to any offset |
| **Consumer model** | Competing consumers — one message goes to one consumer | Consumer groups — each group sees all messages, consumers within a group split partitions |
| **Ordering** | Best-effort (FIFO queues exist but with throughput limits) | Partition-level ordering guaranteed |
| **Scaling** | Add more consumers, broker distributes | Add partitions, consumers assigned to partitions |

**When to choose a queue:**
- Task distribution (send email, process payment) — the message represents work to be done once
- You don't need replay or audit trails
- Simpler operational model is more valuable than advanced features

**When to choose a stream:**
- Multiple services need to react to the same event independently
- You need replay capability (reprocess after a bug fix, build new read models)
- Event sourcing, audit logs, or change data capture
- High throughput with ordering guarantees per entity

</details>

<details>
<summary>2. When should you use a message queue vs the alternatives (direct HTTP calls, database-backed job queues, cron jobs, task queue libraries like Bull/BullMQ) — what signals tell you a full message broker adds real value (decoupling, load leveling, retry isolation), when is a task queue library the right middle ground, and what signals tell you messaging is adding unnecessary complexity?</summary>

**The alternatives and when they're enough:**

- **Direct HTTP calls** — Fine when the consumer must respond synchronously (user waits for result), you trust the downstream service's availability, and the call is fast. Breaks down when the downstream is slow, unreliable, or you need retry logic.
- **Database-backed job queues** — A table with a `status` column, polled by workers. Works for low-throughput background work where you already have a database. Breaks down with high throughput (polling overhead, row locking) and doesn't scale horizontally well.
- **Cron jobs** — Good for periodic batch work (nightly reports, cleanup). Not suitable for event-driven or real-time processing.
- **Task queue libraries (Bull/BullMQ)** — The right middle ground for many Node.js services. Uses Redis as the backend, gives you retries, delayed jobs, rate limiting, priorities, and a dashboard. No separate broker to operate.

**Signals a full message broker adds value:**
- Multiple services need to react to the same event (fan-out)
- You need decoupling — the producer shouldn't know or care who consumes
- Traffic is bursty and you need load leveling (queue absorbs spikes)
- You need retry isolation — a failing consumer shouldn't affect the producer
- You need replay, audit trails, or event sourcing
- Cross-team or cross-language boundaries where a shared protocol matters

**Signals messaging is overkill:**
- Single producer, single consumer, low throughput — a task queue library (BullMQ) handles this with far less operational cost
- The "queue" is just deferring work within the same service — a simple in-process job scheduler works
- You're adding a queue just for retries — HTTP retries with exponential backoff might be enough
- Your team doesn't have the operational capacity to run/monitor a broker
- The system has fewer than 3 services — the decoupling benefit is minimal

**The BullMQ sweet spot:** When you need reliable background job processing with retries, delayed jobs, and rate limiting within a Node.js service that already uses Redis. It's the pragmatic choice before graduating to Kafka/SQS.

</details>

## Conceptual Depth

<details>
<summary>3. What are the delivery guarantee levels (at-most-once, at-least-once, exactly-once) — what does each actually mean in practice, why is exactly-once so hard to achieve (two generals problem, network partitions), and why do most production systems settle for at-least-once with idempotent consumers?</summary>

**At-most-once:** The message is delivered zero or one time. The producer sends and forgets (fire-and-forget), or the consumer acknowledges before processing. If anything fails, the message is lost. Fast but lossy — suitable for metrics, logs, or telemetry where losing some data is acceptable.

**At-least-once:** The message is delivered one or more times. The producer retries on failure, and the consumer acknowledges only after successful processing. If the ack is lost (network blip after processing), the broker redelivers — causing duplicates. This is the default for most systems (SQS, Kafka with `acks=all`, Pub/Sub).

**Exactly-once:** The message is delivered and processed exactly one time. This is what everyone wants but almost no one truly gets.

**Why exactly-once is so hard:**

The fundamental problem is the two generals problem — in a distributed system, you cannot guarantee that both sides (broker and consumer) agree on the state of a message in the presence of network failures. Consider:

1. Consumer processes message and writes to database
2. Consumer sends ack to broker
3. Network fails — ack is lost
4. Broker assumes message wasn't processed, redelivers
5. Consumer processes it again — duplicate

There's no atomic operation that spans "process the message" and "acknowledge to the broker" across network boundaries. Even Kafka's exactly-once semantics (idempotent producers + transactions) only guarantee exactly-once *within Kafka* — the moment you write to an external system (database, API), you're back to at-least-once.

**Why at-least-once + idempotency wins:**

- It's achievable with standard infrastructure
- Duplicates are a bounded, handleable problem (deduplication IDs, idempotent writes, upserts)
- Losing messages (at-most-once) is usually worse than processing them twice
- The consumer owns the idempotency logic, which is where the domain knowledge lives anyway

The practical rule: **design every consumer to be idempotent, assume every message might arrive more than once, and stop chasing exactly-once across system boundaries.**

</details>

<details>
<summary>4. Why do messaging systems distinguish between competing consumers (work queues) and fan-out (pub-sub) — what problem does each pattern solve, when do you need both in the same system, and what are the tradeoffs of implementing fan-out at the broker level vs in application code?</summary>

**Competing consumers (work queue):** Multiple consumers share the workload — each message goes to exactly one consumer. Solves the problem of parallelizing work processing. Example: 5 workers processing payment jobs from a queue, each payment handled by one worker.

**Fan-out (pub-sub):** Every subscriber gets a copy of every message. Solves the problem of notifying multiple independent systems about the same event. Example: an "order placed" event goes to the shipping service, analytics service, and notification service simultaneously.

**When you need both:** Most real systems do. An "order placed" event needs to fan out to multiple services (shipping, analytics, notifications), but within each service you want competing consumers to share the load. This is exactly what Kafka consumer groups provide — each group sees all messages (fan-out), but within a group, partitions are split across consumers (competing).

**Broker-level fan-out vs application-level fan-out:**

| Approach | Pros | Cons |
|---|---|---|
| **Broker-level** (Kafka consumer groups, SNS+SQS, RabbitMQ exchanges) | Decoupled — producer doesn't know about consumers; adding a new subscriber requires no producer changes; broker handles delivery guarantees | Need broker support for the pattern; more infrastructure to configure |
| **Application-level** (service reads message, then calls other services) | Simple to start; no special broker features needed | Tight coupling — producer must know all consumers; failure in one fan-out target can block others; retry logic becomes complex; adding a new consumer requires code changes |

**The clear winner is broker-level fan-out** for anything beyond trivial cases. Application-level fan-out creates a single point of failure and couples the producing service to all consumers. The whole point of messaging is decoupling — doing fan-out in application code undermines that.

</details>

<details>
<summary>5. How do different messaging systems implement competing consumers and fan-out — compare the SNS+SQS fan-out pattern, RabbitMQ exchange types, Kafka consumer groups with multiple topics, and GCP Pub/Sub subscriptions, and what tradeoffs does each approach make?</summary>

**AWS SNS + SQS:**
- SNS topic handles fan-out — each SQS queue subscribes to the topic and gets a copy of every message
- Each SQS queue has competing consumers (multiple Lambda functions or workers polling)
- Fan-out + competing consumers is a first-class, managed pattern
- **Tradeoffs:** No message ordering by default (FIFO SQS exists but caps at 300 msg/s per group), no replay capability, messages deleted after consumption, max message size 256KB

**RabbitMQ exchanges:**
- Fan-out via exchange types: `fanout` (all bound queues get every message), `direct` (routing key match), `topic` (pattern-based routing), `headers`
- Each queue has competing consumers
- Very flexible routing — you can route subsets of events to specific queues based on routing keys
- **Tradeoffs:** No built-in replay, messages deleted after ack, complex exchange/binding topology can become hard to reason about, single-node or clustered but not designed for extreme throughput

**Kafka consumer groups:**
- A single topic holds all messages. Each consumer group gets all messages independently (fan-out). Within a group, partitions are assigned to individual consumers (competing)
- To add a new subscriber, just create a new consumer group — no broker configuration changes
- **Tradeoffs:** Replay is built in (rewind offsets). Ordering guaranteed per partition. But you can't have more consumers than partitions in a group, and rebalancing causes pauses. No per-message routing — all consumers in a group see all messages from assigned partitions (filter in application code)

**GCP Pub/Sub:**
- A topic has multiple subscriptions (fan-out). Each subscription delivers to its own set of subscribers (competing consumers)
- Similar mental model to SNS+SQS but as a single managed service
- Supports both push (HTTP endpoint) and pull delivery
- **Tradeoffs:** At-least-once delivery, no native ordering guarantee (ordering keys available but with throughput limits), 7-day retention, replay by seeking to a timestamp. Simpler operational model than Kafka but less control

**Summary of key distinctions:**

| System | Fan-out mechanism | Competing consumer mechanism | Replay | Ordering |
|---|---|---|---|---|
| SNS + SQS | SNS subscriptions | SQS consumers | No | FIFO queues (limited) |
| RabbitMQ | Exchange bindings | Queue consumers | No | Per-queue FIFO |
| Kafka | Consumer groups | Partition assignment | Yes (offsets) | Per-partition |
| GCP Pub/Sub | Subscriptions | Subscription subscribers | Yes (seek) | Ordering keys (limited) |

</details>

<details>
<summary>6. Why did Kafka choose an append-only log as its core abstraction instead of a traditional message queue — how do partitions enable horizontal scaling, why is partition key strategy critical for ordering guarantees, and what happens to message ordering when you have more consumers than partitions?</summary>

**Why an append-only log:**

Kafka was designed at LinkedIn for high-throughput event streaming, not task distribution. An append-only log gives you:

- **Sequential disk writes** — appending is the fastest I/O operation. Kafka achieves millions of writes/second by leveraging OS page cache and sequential access patterns.
- **Retention and replay** — messages aren't deleted after consumption. Consumers track their position (offset) independently, enabling replay, multiple consumer groups, and rebuilding state.
- **Simplicity** — no need for complex per-message ack tracking or redelivery queues. The log is the source of truth; consumers just move forward through it.

**How partitions enable scaling:**

A topic is split into partitions, each an independent, ordered log. Partitions are distributed across brokers, so:
- **Write scaling:** Producers write to different partitions in parallel across different brokers
- **Read scaling:** Each consumer in a group is assigned a subset of partitions, reading in parallel
- **Storage scaling:** Data is spread across multiple disks/machines

**Why partition key strategy is critical:**

Messages with the same partition key always go to the same partition (via hash(key) % num_partitions). This is the ONLY ordering guarantee Kafka provides — **ordering within a partition**.

- **Order ID as key:** All events for order #123 go to the same partition, processed in order. Good for per-entity ordering.
- **Customer ID as key:** All events for a customer go to the same partition. Good for per-customer ordering.
- **Tenant ID as key:** Dangerous — one high-volume tenant creates a "hot partition" that one consumer must handle alone while others sit idle.
- **No key (null):** Round-robin distribution. Maximum parallelism, zero ordering.

**More consumers than partitions:**

If you have 6 consumers in a group but only 4 partitions, 2 consumers sit completely idle — they receive no messages. A partition can only be assigned to one consumer within a group. This is why you should provision enough partitions upfront (partitions can be increased but not decreased, and increasing changes key-to-partition mapping).

</details>

<details>
<summary>7. How do Kafka consumer groups work and why does rebalancing cause processing pauses — what triggers a rebalance, how does the cooperative sticky assignor reduce disruption compared to the eager protocol, and what are the tradeoffs of static group membership?</summary>

**How consumer groups work:**

A consumer group is a set of consumers that cooperatively consume a topic. The group coordinator (a Kafka broker) assigns partitions to consumers so that each partition is handled by exactly one consumer in the group. Consumers send heartbeats to the coordinator to signal liveness and periodically poll for messages (failing to poll triggers a rebalance).

**What triggers a rebalance:**
- A consumer joins the group (new instance spun up)
- A consumer leaves the group (shutdown, crash, missed heartbeat, exceeded `max.poll.interval.ms`)
- Topic partition count changes
- Consumer subscribes to a new topic matching a pattern

**Why rebalancing causes pauses (eager protocol):**

With the default eager rebalance protocol, ALL consumers in the group stop processing, revoke ALL their partition assignments, rejoin the group, and get new assignments. During this window (seconds to minutes depending on group size), zero messages are processed. This is the "stop-the-world" rebalance.

**Cooperative sticky assignor:**

Instead of revoking all assignments, the cooperative protocol only revokes partitions that actually need to move. It works in two phases:
1. First rebalance: consumers report their current assignments. The coordinator computes new assignments and only revokes partitions that need to move.
2. Second rebalance: revoked partitions are assigned to their new owners.

Consumers that keep their partitions never stop processing. Only the partitions being moved experience a brief pause. This dramatically reduces disruption.

**Static group membership:**

Each consumer is assigned a persistent `group.instance.id`. When a consumer with static membership disconnects (restart, brief network issue), the broker waits for `session.timeout.ms` before triggering a rebalance instead of rebalancing immediately. When the consumer comes back with the same instance ID, it gets its previous partitions back without any rebalance.

**Tradeoffs of static membership:**
- **Pros:** Eliminates rebalances during rolling deployments, brief network blips, or container restarts. Much smoother operations.
- **Cons:** If a consumer truly dies, detection is delayed by the session timeout (you set it longer to avoid false rebalances). Requires managing unique instance IDs across deployments. Partitions assigned to a dead consumer are unprocessed until the timeout expires.

</details>

<details>
<summary>8. Why is there a fundamental tension between ordering and parallelism — why can't you have both easily, how does partition-level ordering work as a compromise, and when should you relax ordering requirements entirely?</summary>

**The fundamental tension:**

Total ordering means processing messages in exactly the order they were produced — sequentially, one at a time. Parallelism means processing multiple messages simultaneously across multiple consumers. These are inherently contradictory: the moment you process messages in parallel, you lose the guarantee that message B completes after message A, even if A was produced first.

A single ordered stream processed by a single consumer gives you perfect ordering but zero parallelism. Splitting across N consumers gives you N-way parallelism but no global ordering.

**Partition-level ordering as the compromise:**

Kafka's answer is partitions. Within a single partition, messages are strictly ordered and processed by exactly one consumer. Across partitions, there's no ordering guarantee. By choosing a meaningful partition key, you get:

- **Per-entity ordering:** Partition by order ID — all events for order #123 are ordered. Events for different orders process in parallel across partitions.
- **Per-customer ordering:** Partition by customer ID — all events for a customer are ordered. Different customers process in parallel.

This works because most ordering requirements are scoped to an entity, not global. You rarely need "all events in the entire system must be in order" — you need "all events for this order must be in order."

**When to relax ordering entirely:**

- **Aggregation/analytics:** Counting page views, computing averages — the final result is the same regardless of processing order.
- **Independent operations:** Sending notification emails — the order doesn't matter as long as each email is sent.
- **Idempotent operations:** If processing is idempotent and the final state is the same regardless of order (last-write-wins, upserts by ID), strict ordering adds cost for no benefit.
- **When ordering is "nice to have" but not a correctness requirement:** Logs arriving slightly out of order in a search index is usually fine; debiting an account before a credit is not.

**The practical rule:** Start by asking "what breaks if messages arrive out of order?" If the answer is "nothing" or "we can handle it with idempotency," skip ordering and take the parallelism. Only enforce ordering where correctness depends on it, and scope it as narrowly as possible (per-entity, not global).

</details>

<details>
<summary>9. How does Kafka's offset management work and why is it central to replay capability — what are committed offsets, how do offset reset policies (earliest vs latest) affect consumer behavior on restart, when would you rewind a consumer group to reprocess messages, and what are compacted topics and how do they provide latest-state lookups instead of full event history?</summary>

**Committed offsets:**

Every message in a partition has a sequential offset (0, 1, 2, ...). A consumer tracks which offset it has processed. When it commits an offset, it's telling Kafka "I've processed everything up to and including this offset." Committed offsets are stored in the internal `__consumer_offsets` topic. On restart, the consumer resumes from its last committed offset.

Two commit strategies:
- **Auto-commit** (`enable.auto.commit=true`): Offsets committed periodically (every 5s by default). Risk: if the consumer crashes after auto-commit but before finishing processing, messages are skipped (at-most-once).
- **Manual commit**: Consumer explicitly commits after processing. Risk: if the consumer crashes after processing but before committing, messages are redelivered (at-least-once). This is the preferred approach.

**Offset reset policies:**

When a consumer group has no committed offset (first time reading, or offsets expired), `auto.offset.reset` controls where to start:
- **`earliest`**: Start from the beginning of the partition. Used when you need to process all historical data (building a new read model, backfilling).
- **`latest`**: Start from the newest messages only. Used when historical data isn't needed (real-time monitoring, dashboards).

**When to rewind a consumer group:**

- **Bug fix replay:** Your consumer had a bug that wrote incorrect data. Fix the code, reset offsets to before the bug was deployed, reprocess.
- **New consumer/read model:** A new service needs to build its state from historical events.
- **Schema migration:** Reprocess events with an updated schema or transformation.

Rewind using: `kafka-consumer-groups.sh --reset-offsets --to-datetime <timestamp> --group <group> --topic <topic> --execute`

**Compacted topics:**

Instead of retaining all messages for a time period, a compacted topic retains only the latest message for each key. Kafka's log compaction periodically removes older records with the same key, keeping only the most recent.

Use cases:
- **Latest state lookups:** A topic keyed by user ID where each message is the user's current profile. Consumers read the compacted topic to rebuild a complete snapshot of all user states.
- **Changelog/CDC:** Database change events compacted to the latest state per row.
- **KTable in Kafka Streams:** Compacted topics back materialized views.

A message with a null value (tombstone) tells compaction to delete that key entirely. Compacted topics give you the semantics of a key-value store built on top of the log abstraction.

</details>

<details>
<summary>10. How do you achieve idempotent message processing — what are the strategies (deduplication IDs, idempotent database writes, transactional outbox), how do deduplication windows work, and why is idempotency required even when the broker claims exactly-once delivery?</summary>

**Why idempotency is always required:**

As covered in question 3, exactly-once delivery across system boundaries is impossible. Even Kafka's exactly-once semantics only guarantee exactly-once within Kafka itself. The moment your consumer writes to a database, calls an API, or sends an email, you're in at-least-once territory. Broker-level exactly-once doesn't help when the failure happens between "process" and "ack."

**Strategy 1: Deduplication IDs**

Each message carries a unique ID (UUID, or a natural key like `orderId + eventType + version`). The consumer checks a deduplication store before processing:

```typescript
async function processMessage(message: OrderEvent): Promise<void> {
  const dedupeKey = `${message.orderId}:${message.eventId}`;

  // Check + insert atomically in a transaction
  const alreadyProcessed = await db.query(
    'INSERT INTO processed_messages (dedup_key, processed_at) VALUES ($1, NOW()) ON CONFLICT (dedup_key) DO NOTHING RETURNING dedup_key',
    [dedupeKey]
  );

  if (!alreadyProcessed.rowCount) {
    return; // already processed, skip
  }

  await processOrder(message);
}
```

**Deduplication windows:** You can't keep dedup IDs forever. Set a TTL (e.g., 7 days) based on how long a duplicate might realistically arrive. SQS provides a 5-minute deduplication window for FIFO queues. After the window, a duplicate won't be caught — design your processing to be safe even without the dedup check (idempotent writes).

**Strategy 2: Idempotent database writes**

Design writes so that applying them multiple times produces the same result:

- **Upserts:** `INSERT ... ON CONFLICT DO UPDATE` — writing the same data twice results in the same state
- **Conditional updates:** `UPDATE orders SET status = 'shipped' WHERE id = $1 AND status = 'paid'` — the second execution is a no-op because the status already changed
- **Version checks:** `UPDATE orders SET ... WHERE id = $1 AND version = $2` — optimistic concurrency control

This is often simpler and more reliable than dedup IDs because it's embedded in the business logic itself.

**Strategy 3: Transactional outbox pattern**

This solves the dual-write problem on the **producer** side. Instead of writing to the database AND publishing a message (which can partially fail), the producer writes both the business data and the outbound message to the same database in a single transaction. A separate process (outbox relay) polls the outbox table and publishes to the broker.

```
1. BEGIN TRANSACTION
2. INSERT INTO orders (...)
3. INSERT INTO outbox (topic, key, payload)
4. COMMIT

-- Separate relay process:
5. SELECT * FROM outbox WHERE published = false
6. Publish to Kafka
7. UPDATE outbox SET published = true WHERE id = ...
```

This guarantees the message is published if and only if the business write succeeded. The relay might publish duplicates (crash between step 6 and 7), so the consumer still needs idempotency — but it eliminates the "wrote to DB but message never sent" failure mode.

**In practice, combine strategies:** Use the outbox pattern on the producer side to guarantee message delivery, and idempotent writes (upserts, conditional updates) on the consumer side to handle duplicates. Add dedup IDs as a safety net for operations that aren't naturally idempotent (sending emails, charging credit cards).

</details>

<details>
<summary>11. Why is backpressure critical in messaging systems — what is consumer lag, how do pull-based consumers (Kafka, SQS) handle backpressure differently from push-based consumers (Pub/Sub push subscriptions, RabbitMQ), and what are the tradeoffs of each model when a consumer falls behind?</summary>

**Why backpressure matters:**

Without backpressure, a fast producer and slow consumer leads to one of two outcomes: the broker runs out of storage (messages pile up), or the consumer runs out of memory (messages delivered faster than it can process them). Backpressure is the mechanism that prevents this by slowing down the flow to match the consumer's capacity.

**Consumer lag** is the difference between the latest message produced and the latest message consumed. In Kafka, it's measured as the offset gap: if the latest offset is 1,000,000 and the consumer's committed offset is 950,000, lag is 50,000 messages. Growing lag means the consumer can't keep up.

**Pull-based consumers (Kafka, SQS):**

The consumer controls the pace — it fetches messages when ready. Backpressure is built in naturally:
- Consumer polls for a batch, processes it, then polls again
- If processing is slow, polls are less frequent — the broker just holds messages
- Consumer never gets overwhelmed because it only takes what it can handle
- Lag grows on the broker side (which is designed for it — Kafka retains on disk efficiently)

**Tradeoffs:** Higher latency — there's a gap between "message available" and "consumer polls." Tuning `fetch.min.bytes` and `fetch.max.wait.ms` balances latency vs. efficiency.

**Push-based consumers (RabbitMQ default, Pub/Sub push subscriptions):**

The broker pushes messages to the consumer as they arrive. Backpressure must be explicitly configured:
- **RabbitMQ prefetch (`prefetch_count`):** Limits how many unacknowledged messages the broker delivers. Without it, RabbitMQ floods the consumer. Set prefetch to match your concurrency (e.g., 10 if you process 10 at a time).
- **Pub/Sub push:** Sends HTTP requests to your endpoint. If the endpoint returns errors or times out, Pub/Sub backs off with exponential retry. But if your endpoint is slow (accepts but takes forever to process), messages pile up in your application memory.

**Tradeoffs:** Lower latency — messages delivered immediately. But the consumer can be overwhelmed if prefetch/flow control isn't configured properly. The consumer must actively signal its capacity.

| Model | Backpressure | Latency | Overwhelm risk | Configuration |
|---|---|---|---|---|
| Pull (Kafka, SQS) | Natural — consumer controls pace | Higher (poll interval) | Low | `fetch.min.bytes`, batch size |
| Push (RabbitMQ, Pub/Sub push) | Must configure (prefetch, ack deadline) | Lower (immediate delivery) | High without limits | `prefetch_count`, max outstanding messages |

</details>

<details>
<summary>12. What are dead letter queues and how should you use them — how do messages get routed to a DLQ (max retry exceeded, deserialization failure, schema mismatch), what triage strategies help you analyze DLQ contents, and how do you safely replay messages from the DLQ back to the main queue?</summary>

A **dead letter queue (DLQ)** is a separate queue where messages that can't be processed successfully are sent instead of being discarded or retried forever. It's a safety net that prevents one bad message from blocking the entire pipeline (the "poison pill" problem).

**How messages get routed to a DLQ:**

- **Max retry exceeded:** The consumer failed to process the message N times (e.g., SQS `maxReceiveCount`, or application-level retry counter). After the final attempt, the message moves to the DLQ.
- **Deserialization failure:** The message payload can't be parsed (corrupted, wrong format, encoding issue). No amount of retrying will fix this.
- **Schema mismatch:** The message was produced with a schema version the consumer doesn't understand (missing required fields, unknown type).
- **Business logic rejection:** The message is valid structurally but violates a business rule that won't change (referencing a deleted entity, invalid state transition).

**Key distinction:** Transient failures (database timeout, network blip) should be retried with backoff, not sent to the DLQ immediately. Only route to DLQ after retries are exhausted or for non-retryable errors.

**Triage strategies:**

- **Enrich DLQ messages with context:** When routing to DLQ, attach metadata — original topic, partition, offset, consumer group, error message, stack trace, retry count, timestamp. Without this context, DLQ messages are nearly impossible to triage.
- **Categorize failures:** Group DLQ messages by error type. "90% are schema mismatches from producer v2.3" is actionable; "500 failed messages" is not.
- **Alert on DLQ depth:** Monitor DLQ message count. A spike often indicates a systemic issue (bad deployment, schema change) rather than individual message problems.
- **DLQ dashboard:** Build a simple UI or use existing tools (AWS Console for SQS DLQ, custom Kafka consumer reading the DLQ topic) to browse, search, and inspect failed messages.

**Safe replay:**

1. **Fix the root cause first** — deploy the code fix, schema update, or data fix before replaying
2. **Test with a sample** — replay a few messages to a staging environment to verify the fix works
3. **Replay in controlled batches** — don't dump the entire DLQ back at once; replay in small batches and monitor for errors
4. **Preserve ordering if needed** — if the original topic had ordering requirements, replay in the same order
5. **Ensure idempotency** — the original processing may have partially succeeded before failing; replay must handle duplicates (as covered in question 10)

</details>

<details>
<summary>13. What are the tradeoffs between message serialization formats (JSON, Avro, Protobuf) — how does each handle schema evolution (adding fields, removing fields, renaming), what is a schema registry and why does it matter for Avro/Protobuf, and when is JSON's simplicity worth the performance and schema safety tradeoff?</summary>

| Dimension | JSON | Avro | Protobuf |
|---|---|---|---|
| **Format** | Text, self-describing (keys in every message) | Binary, schema required for reading | Binary, schema required for reading |
| **Size** | Largest (field names repeated) | Smallest (no field names in payload) | Small (field tags instead of names) |
| **Speed** | Slowest (parse text) | Fast (binary, schema-driven) | Fast (binary, code-generated) |
| **Human readable** | Yes | No | No |
| **Schema required** | No (schemaless) | Yes (reader and writer schemas) | Yes (.proto files) |

**Schema evolution — how each handles changes:**

**JSON:**
- Adding fields: Works — old consumers ignore unknown fields (if coded defensively)
- Removing fields: Works — consumers handle missing fields with defaults
- Renaming: Breaking — field names are the contract
- No enforcement: Nothing prevents a producer from sending garbage. Schema validation is purely application-side (Zod, JSON Schema, etc.)

**Avro:**
- Adding fields: Safe if new field has a default value (backward-compatible)
- Removing fields: Safe if removed field had a default (forward-compatible)
- Renaming: Supported via aliases
- Schema resolution: Reader and writer schemas are compared at deserialization time. Avro resolves differences automatically (fills defaults, ignores unknown fields)

**Protobuf:**
- Adding fields: Safe — new fields get a new field number, old consumers skip unknown fields
- Removing fields: Safe if you don't reuse the field number (use `reserved`)
- Renaming: Field numbers are the contract, not names — renaming is safe
- **Key rule:** Never reuse or change field numbers

**Schema registry:**

A centralized service (Confluent Schema Registry is the standard) that stores and versions schemas. Producers register schemas before publishing; consumers fetch schemas to deserialize.

Why it matters:
- **Compatibility enforcement:** The registry rejects schema changes that would break consumers (e.g., removing a field without a default in Avro). You configure compatibility modes: BACKWARD (new schema can read old data), FORWARD (old schema can read new data), FULL (both).
- **Schema ID in messages:** Each message carries a schema ID (typically 4 bytes in the header). The consumer looks up the schema by ID. This decouples deployment — producer and consumer can deploy independently as long as schemas are compatible.
- **Single source of truth:** Without a registry, schema files are copy-pasted across repos, drift, and break in production.

**When JSON's simplicity wins:**
- Low message volume where serialization overhead doesn't matter
- Internal APIs where you control both sides and iterate quickly
- Debugging — you can read messages with `kafkacat` or CloudWatch without special tooling
- Early-stage systems where the schema is still changing rapidly
- When the team doesn't have the operational capacity to run a schema registry

**When to invest in Avro/Protobuf:**
- High throughput where payload size and serialization speed matter (Avro can be 5-10x smaller than JSON)
- Cross-team boundaries where schema contracts prevent breaking changes
- Long-lived event streams where backward/forward compatibility is critical
- You already have a schema registry in your infrastructure

</details>

## Practical — Queue Configuration & Patterns

<details>
<summary>14. Set up a Kafka topic with proper partitioning for an order processing system — choose the partition key strategy (order ID for per-entity ordering), configure the producer with appropriate acks and retries, set up a consumer group, and demonstrate what happens when you add a consumer to the group (rebalancing)</summary>

**Topic creation:**

```bash
# 12 partitions — enough for scaling to 12 consumers, not so many that it creates overhead
# Replication factor 3 — survives loss of 2 brokers
kafka-topics.sh --create \
  --topic order-events \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \  # 7 days
  --config min.insync.replicas=2     # at least 2 replicas must ack
```

**Partition key strategy:** Use `orderId` as the partition key. All events for the same order (created, paid, shipped, delivered) land on the same partition and are processed in order. This avoids the hot partition problem that would occur with `customerId` (one customer with millions of orders skews load).

**Producer configuration (TypeScript with KafkaJS):**

```typescript
import { Kafka, CompressionTypes } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092'],
});

const producer = kafka.producer({
  idempotent: true,           // prevents duplicate messages on producer retries
  maxInFlightRequests: 5,     // safe with idempotent=true (up to 5 allowed); must be 1 for transactional producers
});

await producer.connect();

async function publishOrderEvent(event: OrderEvent): Promise<void> {
  await producer.send({
    topic: 'order-events',
    compression: CompressionTypes.Snappy,
    acks: -1,                 // acks=all — wait for all in-sync replicas
    messages: [
      {
        key: event.orderId,   // partition key — per-order ordering
        value: JSON.stringify(event),
        headers: {
          eventType: event.type,
          correlationId: event.correlationId,
        },
      },
    ],
  });
}
```

Key producer settings:
- `acks: -1` (all) — the broker waits for all in-sync replicas to acknowledge. Combined with `min.insync.replicas=2`, this guarantees the message survives a single broker failure.
- `idempotent: true` — the producer assigns a sequence number to each message. If a retry sends the same message twice, the broker deduplicates it.
- Retries are infinite by default in KafkaJS when idempotent is enabled.

**Consumer group setup:**

```typescript
const consumer = kafka.consumer({
  groupId: 'order-processing-group',
  sessionTimeout: 30000,
  heartbeatInterval: 3000,
  maxWaitTimeInMs: 5000,
});

await consumer.connect();
await consumer.subscribe({ topic: 'order-events', fromBeginning: false });

await consumer.run({
  partitionsConsumedConcurrently: 3, // process 3 partitions in parallel
  eachMessage: async ({ topic, partition, message }) => {
    const event = JSON.parse(message.value!.toString()) as OrderEvent;
    console.log(`Partition ${partition}, Offset ${message.offset}: ${event.type}`);
    await processOrderEvent(event);
    // KafkaJS auto-commits after eachMessage returns successfully
  },
});
```

**What happens when you add a consumer:**

Starting state: 2 consumers, 12 partitions. Consumer A has partitions 0-5, Consumer B has partitions 6-11.

Add Consumer C:
1. C joins the group and sends a JoinGroup request to the coordinator
2. Coordinator triggers a rebalance
3. With **eager protocol**: All consumers stop, revoke all partitions, rejoin. New assignment: A gets 0-3, B gets 4-7, C gets 8-11. Processing paused during the entire rebalance.
4. With **cooperative sticky assignor**: A keeps 0-3, revokes 4-5. B keeps 6-9, revokes 10-11. Second rebalance assigns 4-5 and 10-11 to C. Only moved partitions pause — A and B continue processing their kept partitions throughout.

</details>

<details>
<summary>15. Implement an idempotent message consumer that handles at-least-once delivery — show the consumer code that uses a deduplication ID stored in the database, wraps processing in a transaction, and handles the case where the same message arrives twice. Compare this with using the transactional outbox pattern on the producer side</summary>

**Idempotent consumer with dedup ID in a transaction:**

The core idea: wrap the dedup check and business logic in a single database transaction. If the dedup ID already exists, skip. If the transaction fails partway, both the dedup record and the business write roll back together.

```typescript
import { Kafka } from 'kafkajs';
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

interface OrderEvent {
  eventId: string;      // unique per event — the dedup key
  orderId: string;
  type: 'ORDER_PAID';
  amount: number;
}

async function handleOrderPaid(event: OrderEvent): Promise<void> {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // Attempt to insert dedup record — ON CONFLICT means it's a duplicate
    const { rowCount } = await client.query(
      `INSERT INTO processed_events (event_id, processed_at)
       VALUES ($1, NOW())
       ON CONFLICT (event_id) DO NOTHING`,
      [event.eventId]
    );

    if (rowCount === 0) {
      // Already processed — skip
      await client.query('ROLLBACK');
      console.log(`Duplicate event ${event.eventId}, skipping`);
      return;
    }

    // Business logic — same transaction guarantees atomicity
    await client.query(
      `UPDATE orders SET status = 'paid', paid_amount = $1 WHERE id = $2`,
      [event.amount, event.orderId]
    );

    await client.query(
      `INSERT INTO payments (order_id, amount, event_id) VALUES ($1, $2, $3)`,
      [event.orderId, event.amount, event.eventId]
    );

    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err; // let the consumer retry
  } finally {
    client.release();
  }
}

// Wire it up to the Kafka consumer
await consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value!.toString()) as OrderEvent;
    await handleOrderPaid(event);
  },
});
```

**What happens with a duplicate message:**
1. First delivery: dedup insert succeeds (rowCount=1), business logic runs, transaction commits
2. Second delivery: dedup insert hits `ON CONFLICT DO NOTHING` (rowCount=0), function returns early, no business logic executes
3. If the first delivery crashed between dedup insert and commit: the transaction rolls back entirely (both dedup and business writes), so the retry processes it as a fresh message

**Dedup table maintenance:** Add a TTL cleanup job to prevent the table from growing forever:
```sql
-- Run daily via cron
DELETE FROM processed_events WHERE processed_at < NOW() - INTERVAL '7 days';
```

**Comparison with transactional outbox (producer side):**

The outbox pattern (covered in question 10) solves a different problem — it guarantees the producer publishes a message if and only if the business write succeeds. It doesn't make the consumer idempotent.

| Concern | Idempotent consumer (dedup) | Transactional outbox |
|---|---|---|
| **Solves** | Duplicate processing on the consumer side | Dual-write problem on the producer side |
| **Where** | Consumer | Producer |
| **Mechanism** | Dedup check + transaction | Business write + outbox insert in one transaction |
| **Still needs the other?** | Yes — without outbox, producer might fail to publish | Yes — outbox relay can publish duplicates |

**In practice, use both:** Outbox on the producer guarantees delivery. Idempotent consumer handles the duplicates that inevitably arise from retries on either side.

</details>

<details>
<summary>16. Configure competing consumers and fan-out for the same event stream — show how to set up a Kafka topic where order events are consumed by both a shipping service (competing consumers sharing the work) and an analytics service (independent consumer group seeing all events), and explain why consumer groups enable this pattern</summary>

**The pattern:** One topic, multiple consumer groups. Each group independently reads all messages (fan-out). Within each group, partitions are distributed across consumers (competing consumers).

```
                          ┌─ Shipping Consumer 1 (partitions 0-3)
order-events ──► shipping-group ─┤
                          └─ Shipping Consumer 2 (partitions 4-7)

                          ┌─ Analytics Consumer 1 (partitions 0-3)
order-events ──► analytics-group ┤
                          └─ Analytics Consumer 2 (partitions 4-7)
```

**Shipping service — competing consumers sharing the work:**

```typescript
// shipping-service/consumer.ts
const shippingConsumer = kafka.consumer({
  groupId: 'shipping-group',  // unique group ID for shipping
});

await shippingConsumer.connect();
await shippingConsumer.subscribe({ topic: 'order-events' });

await shippingConsumer.run({
  eachMessage: async ({ partition, message }) => {
    const event = JSON.parse(message.value!.toString()) as OrderEvent;

    // Only process relevant events — filter in application code
    if (event.type === 'ORDER_PAID') {
      await createShipment(event.orderId, event.shippingAddress);
    }
    // Other event types are consumed (offset advances) but ignored
  },
});
```

Deploy 2-4 instances of the shipping service. Kafka assigns partitions across them automatically — each instance handles a subset of orders.

**Analytics service — independent consumer group seeing all events:**

```typescript
// analytics-service/consumer.ts
const analyticsConsumer = kafka.consumer({
  groupId: 'analytics-group',  // different group ID — sees ALL messages independently
});

await analyticsConsumer.connect();
await analyticsConsumer.subscribe({ topic: 'order-events' });

await analyticsConsumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value!.toString()) as OrderEvent;
    // Process every event type for analytics
    await writeToAnalyticsStore(event);
  },
});
```

**Why consumer groups enable this:**

Each consumer group maintains its own set of committed offsets. `shipping-group` and `analytics-group` track their positions independently:
- Shipping can be behind (slow shipment creation) without affecting analytics
- Analytics can be reset to reprocess historical data without touching shipping
- Adding a third service (notifications) is just creating a new consumer group — no topic changes, no producer changes

This is Kafka's killer feature compared to traditional queues. With SQS, you'd need SNS fan-out to separate queues per service (as covered in question 5). With Kafka, one topic serves unlimited consumer groups, each with independent offset tracking and competing consumer support built in.

**Scaling each group independently:** Shipping might need 4 consumers (shipment creation is slow) while analytics needs only 2 (writes are fast). Each group scales based on its own throughput needs, up to the partition count.

</details>

<details>
<summary>17. Set up a dead letter queue with proper routing — show the configuration that routes messages to a DLQ after N failed attempts, implement a DLQ consumer that logs failed messages with context for triage, and demonstrate how to replay messages from the DLQ back to the original queue after fixing the issue</summary>

Kafka doesn't have built-in DLQ support like SQS does — you implement it in application code by publishing failed messages to a separate topic.

**DLQ topic creation:**

```bash
kafka-topics.sh --create \
  --topic order-events-dlq \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=2592000000  # 30 days — longer retention for investigation
```

**Consumer with retry + DLQ routing:**

```typescript
const MAX_RETRIES = 3;
const dlqProducer = kafka.producer();
await dlqProducer.connect();

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const retryCount = parseInt(message.headers?.retryCount?.toString() ?? '0');

    try {
      const event = JSON.parse(message.value!.toString()) as OrderEvent;
      await processOrderEvent(event);
    } catch (error) {
      if (retryCount >= MAX_RETRIES || isNonRetryable(error)) {
        // Route to DLQ with full context
        await dlqProducer.send({
          topic: `${topic}-dlq`,
          messages: [
            {
              key: message.key,
              value: message.value,
              headers: {
                ...message.headers,
                originalTopic: topic,
                originalPartition: String(partition),
                originalOffset: message.offset,
                errorMessage: (error as Error).message,
                errorStack: (error as Error).stack ?? '',
                failedAt: new Date().toISOString(),
                retryCount: String(retryCount),
              },
            },
          ],
        });
        console.error(`Message sent to DLQ after ${retryCount} retries`, {
          topic,
          partition,
          offset: message.offset,
          error: (error as Error).message,
        });
      } else {
        throw error; // let KafkaJS retry (message won't be committed)
      }
    }
  },
});

function isNonRetryable(error: unknown): boolean {
  return (
    error instanceof SyntaxError ||          // JSON parse failure
    error instanceof SchemaValidationError || // schema mismatch
    error instanceof BusinessRuleError        // invalid state transition
  );
}
```

**DLQ consumer for triage:**

```typescript
const dlqConsumer = kafka.consumer({ groupId: 'dlq-triage-group' });
await dlqConsumer.subscribe({ topic: 'order-events-dlq' });

await dlqConsumer.run({
  eachMessage: async ({ message }) => {
    const headers = message.headers ?? {};
    const failedMessage = {
      key: message.key?.toString(),
      payload: message.value?.toString(),
      originalTopic: headers.originalTopic?.toString(),
      originalPartition: headers.originalPartition?.toString(),
      originalOffset: headers.originalOffset?.toString(),
      error: headers.errorMessage?.toString(),
      failedAt: headers.failedAt?.toString(),
      retries: headers.retryCount?.toString(),
    };

    // Store in a queryable format for investigation
    await db.query(
      `INSERT INTO dlq_messages (key, payload, original_topic, error, failed_at)
       VALUES ($1, $2, $3, $4, $5)`,
      [failedMessage.key, failedMessage.payload, failedMessage.originalTopic,
       failedMessage.error, failedMessage.failedAt]
    );
  },
});
```

**Replay from DLQ after fixing the issue:**

```typescript
async function replayDlqMessages(
  sourceTopic: string,
  batchSize: number = 100,
): Promise<void> {
  const replayConsumer = kafka.consumer({ groupId: 'dlq-replay-group' });
  const replayProducer = kafka.producer();

  await replayConsumer.connect();
  await replayProducer.connect();
  await replayConsumer.subscribe({ topic: `${sourceTopic}-dlq` });

  let replayed = 0;

  await replayConsumer.run({
    eachMessage: async ({ message }) => {
      // Republish to original topic without DLQ headers
      await replayProducer.send({
        topic: sourceTopic,
        messages: [
          {
            key: message.key,
            value: message.value,
            headers: { replayedFromDlq: 'true' },
          },
        ],
      });

      replayed++;
      if (replayed % batchSize === 0) {
        console.log(`Replayed ${replayed} messages, pausing for monitoring...`);
        // Optional: pause briefly to check for errors before continuing
      }
    },
  });
}
```

**Gotcha:** Before replaying, verify the fix works by testing with a few DLQ messages in a staging environment. Replaying 10,000 broken messages back to the main topic just fills the DLQ again.

</details>

<details>
<summary>18. Implement message serialization with schema evolution — show a Protobuf or Avro message definition, demonstrate adding a new optional field (backward-compatible), show how a consumer running the old schema handles messages with the new field, and explain what happens with a breaking change (removing a required field)</summary>

Using Protobuf since it's more common in Node.js/TypeScript ecosystems than Avro.

**Initial schema (v1):**

```protobuf
// order_event.proto — v1
syntax = "proto3";

message OrderEvent {
  string order_id = 1;
  string customer_id = 2;
  double amount = 3;
  string currency = 4;
  string event_type = 5;       // ORDER_CREATED, ORDER_PAID, etc.
  int64 timestamp_ms = 6;
}
```

**Adding a new optional field (backward-compatible — v2):**

```protobuf
// order_event.proto — v2
syntax = "proto3";

message OrderEvent {
  string order_id = 1;
  string customer_id = 2;
  double amount = 3;
  string currency = 4;
  string event_type = 5;
  int64 timestamp_ms = 6;
  string shipping_address = 7;  // NEW — optional by default in proto3
  repeated string item_ids = 8; // NEW — list of item IDs
}
```

**How a consumer running old schema (v1) handles new messages:**

Protobuf's wire format is based on field numbers, not field names. When a v1 consumer reads a message with fields 7 and 8:
- Fields 1-6 are deserialized normally
- Fields 7 and 8 are unknown — Protobuf **silently ignores** unknown fields
- No error, no crash. The consumer processes the message with the fields it knows about.

```typescript
// Producer using v2 schema
const message = OrderEvent.encode({
  orderId: 'ord-123',
  customerId: 'cust-456',
  amount: 99.99,
  currency: 'EUR',
  eventType: 'ORDER_CREATED',
  timestampMs: Date.now(),
  shippingAddress: '123 Main St',  // new field
  itemIds: ['item-1', 'item-2'],   // new field
}).finish();

// Consumer using v1 schema — deserializes successfully
const event = OrderEventV1.decode(message);
// event.orderId = 'ord-123'
// event.shippingAddress → undefined (field doesn't exist in v1 schema)
// No error thrown
```

**Using protobuf with Kafka in TypeScript:**

```typescript
import { load } from 'protobufjs';

const root = await load('order_event.proto');
const OrderEvent = root.lookupType('OrderEvent');

// Producer — serialize
const payload = OrderEvent.encode(
  OrderEvent.create({ orderId: 'ord-123', amount: 99.99 })
).finish();

await producer.send({
  topic: 'order-events',
  messages: [{ key: 'ord-123', value: Buffer.from(payload) }],
});

// Consumer — deserialize
await consumer.run({
  eachMessage: async ({ message }) => {
    const event = OrderEvent.decode(message.value!) as any;
    console.log(event.orderId, event.amount);
  },
});
```

**What happens with a breaking change (removing a field or reusing a field number):**

```protobuf
// BREAKING — v3 (bad)
message OrderEvent {
  string order_id = 1;
  // Removed customer_id (field 2) and reused the number
  string tenant_id = 2;        // BREAKS old consumers — they read tenant_id as customer_id
  double amount = 3;
}
```

If you reuse field number 2 with a different type or meaning, old consumers deserialize `tenant_id` bytes as `customer_id`. This causes silent data corruption — the worst kind of bug.

**Safe removal:**

```protobuf
message OrderEvent {
  string order_id = 1;
  reserved 2;                  // prevents field number reuse
  reserved "customer_id";      // prevents field name reuse
  double amount = 3;
  string tenant_id = 7;       // new field gets a NEW number
}
```

**Schema evolution rules for Protobuf:**
1. Never change a field number
2. Never reuse a field number — use `reserved` for removed fields
3. New fields must use new field numbers
4. Renaming fields is safe (wire format uses numbers, not names)
5. Changing field types is usually breaking (int32 to string breaks deserialization)

</details>

<details>
<summary>19. Configure backpressure handling for a consumer that processes messages slower than the producer publishes — show consumer lag monitoring, configure prefetch/batch limits to prevent memory exhaustion, implement autoscaling consumers based on lag metrics, and explain when to apply backpressure to the producer instead</summary>

**Consumer lag monitoring:**

```bash
# Check lag for a consumer group
kafka-consumer-groups.sh --describe --group order-processing-group --bootstrap-server kafka:9092

# Output:
# TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-events   0          142850          145000          2150
# order-events   1          138200          145100          6900  ← falling behind
```

Expose lag as a Prometheus metric for alerting:

```typescript
import { register, Gauge } from 'prom-client';

const consumerLag = new Gauge({
  name: 'kafka_consumer_lag',
  help: 'Number of messages behind the latest offset',
  labelNames: ['topic', 'partition', 'group'],
});

// Update lag periodically using the admin client
const admin = kafka.admin();
await admin.connect();

async function updateLagMetrics(): Promise<void> {
  const offsets = await admin.fetchTopicOffsets('order-events');
  const groupOffsets = await admin.fetchOffsets({
    groupId: 'order-processing-group',
    topics: ['order-events'],
  });

  for (const topicOffset of offsets) {
    const groupOffset = groupOffsets.find(
      (g) => g.partitions.some((p) => p.partition === topicOffset.partition)
    );
    const committed = parseInt(groupOffset?.partitions
      .find((p) => p.partition === topicOffset.partition)?.offset ?? '0');
    const latest = parseInt(topicOffset.offset);

    consumerLag.set(
      { topic: 'order-events', partition: String(topicOffset.partition), group: 'order-processing-group' },
      latest - committed
    );
  }
}

setInterval(updateLagMetrics, 10_000);
```

**Configure batch limits to prevent memory exhaustion:**

```typescript
const consumer = kafka.consumer({
  groupId: 'order-processing-group',
  maxBytes: 1048576,           // 1MB max per partition per fetch
  maxWaitTimeInMs: 5000,       // wait up to 5s to fill a batch
  // Don't fetch more until current batch is processed
});

// Process one message at a time per partition (safe but slower)
await consumer.run({
  partitionsConsumedConcurrently: 2,  // limit concurrent partitions
  eachMessage: async ({ message }) => {
    await processOrderEvent(JSON.parse(message.value!.toString()));
  },
});

// OR: use eachBatch for more control
await consumer.run({
  eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
    for (const message of batch.messages) {
      await processOrderEvent(JSON.parse(message.value!.toString()));
      resolveOffset(message.offset);  // mark progress within batch
      await heartbeat();               // prevent session timeout during slow processing
    }
  },
});
```

**Autoscaling consumers based on lag (Kubernetes HPA):**

```yaml
# HPA based on consumer lag via Prometheus adapter
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-consumer-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-consumer
  minReplicas: 2
  maxReplicas: 12     # match partition count
  metrics:
    - type: External
      external:
        metric:
          name: kafka_consumer_lag
          selector:
            matchLabels:
              group: order-processing-group
              topic: order-events
        target:
          type: AverageValue
          averageValue: "1000"   # scale up when lag exceeds 1000 per consumer
```

Key constraint: `maxReplicas` should not exceed the partition count — extra consumers sit idle (as covered in question 6).

**When to apply backpressure to the producer instead:**

- **Bounded system:** The queue has a size limit (RabbitMQ with `x-max-length`, SQS retention window). If the consumer can't keep up, the broker drops messages. Better to slow the producer.
- **Producer controls the pace:** If the producer is a batch job or API gateway, you can rate-limit or throttle it. If the producer is a stream of user actions, you can't slow down users.
- **Cascading failure risk:** If consumer lag causes broker disk to fill up, the whole cluster is at risk. Producer-side rate limiting (quota in Kafka, or application-level) prevents this.
- **Cost control:** Managed services (Pub/Sub, SQS) charge per message. Runaway producers generate costs even if consumers are offline.

The general principle: scale consumers first (it's cheaper and simpler), apply producer backpressure only when consumer scaling hits its limit (partition count) or when the broker itself is at risk.

</details>

## Practical — Troubleshooting & Decision-Making

<details>
<summary>20. Consumer lag is growing steadily on a Kafka topic — walk through the diagnosis: check if it's a slow consumer (processing bottleneck), uneven partition distribution (hot partition from a skewed key), consumer rebalancing storms, or insufficient consumer instances. Show the commands and metrics for each check</summary>

Walk through this systematically, starting with the most common causes.

**Step 1: Confirm lag is growing and identify scope**

```bash
# Check current lag per partition
kafka-consumer-groups.sh --describe --group order-processing-group \
  --bootstrap-server kafka:9092

# TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG    CONSUMER-ID
# order-events   0          500000          502000          2000   consumer-1
# order-events   1          498000          510000          12000  consumer-2  ← hot partition?
# order-events   2          501000          503000          2000   consumer-3
# order-events   3          499000          501000          2000   -           ← no consumer!
```

Key observations: Is lag uniform across partitions, or concentrated? Are all partitions assigned to consumers?

**Step 2: Check for insufficient consumer instances**

If any partition shows `-` for CONSUMER-ID (partition 3 above), it's unassigned — you have fewer consumers than partitions, or a consumer died.

```bash
# Check group member count
kafka-consumer-groups.sh --describe --group order-processing-group \
  --members --bootstrap-server kafka:9092

# GROUP                    CONSUMER-ID          PARTITIONS
# order-processing-group   consumer-1           0
# order-processing-group   consumer-2           1
# order-processing-group   consumer-3           2
# Missing: partition 3 has no consumer
```

**Fix:** Scale up consumer instances. Check if a pod crashed (`kubectl get pods`). Ensure consumer count does not exceed partition count.

**Step 3: Check for hot partition (skewed partition key)**

If one partition has significantly more lag than others (partition 1 with 12,000 lag above), the partition key is skewed.

```bash
# Check message rate per partition
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list kafka:9092 --topic order-events --time -1  # latest offsets
# Compare with offsets from 5 minutes ago to see rate per partition

# Or use Prometheus:
# rate(kafka_server_brokertopicmetrics_messagesin_total{topic="order-events"}[5m]) BY (partition)
```

If one partition receives 10x the messages, your partition key has a hot key (e.g., one large tenant, one popular product).

**Fix:** Change the partition key strategy to something more evenly distributed, or add a suffix to break up hot keys: `${customerId}-${orderId % 4}` (trades ordering for distribution).

**Step 4: Check for slow consumer (processing bottleneck)**

If lag is growing evenly across all partitions, each consumer is individually too slow.

Metrics to check:
- **Message processing time:** How long does each message take? If your consumer does a database write that takes 500ms per message, you process 2 msg/s per consumer.
- **External dependency latency:** Is the database slow? Is an HTTP call timing out?
- **CPU/memory:** Is the consumer pod resource-constrained?

```bash
# Check consumer pod resource usage
kubectl top pods -l app=order-consumer

# Check if processing time has increased (application metric)
# histogram: order_event_processing_duration_seconds
# Alert: p99 > 1s
```

**Fix:** Optimize the processing logic (batch database writes, add connection pooling, cache lookups). If the processing is inherently slow, scale up consumers and/or partitions.

**Step 5: Check for consumer rebalancing storms**

If lag spikes correlate with rebalances (visible in consumer logs or metrics), consumers are spending more time rebalancing than processing.

```bash
# Check consumer group state
kafka-consumer-groups.sh --describe --group order-processing-group \
  --state --bootstrap-server kafka:9092

# GROUP                    STATE
# order-processing-group   PreparingRebalance  ← stuck in rebalance
```

Signs of rebalance storms:
- Consumer logs show frequent "Revoking partitions" / "Joining group" messages
- `max.poll.interval.ms` is being exceeded (processing takes too long, consumer is kicked)
- Session timeouts from GC pauses or network issues

**Fix:**
- Increase `max.poll.interval.ms` if processing is legitimately slow
- Reduce `max.poll.records` so each poll returns fewer messages, processed faster
- Switch to cooperative sticky assignor (as covered in question 7)
- Use static group membership to survive brief disconnects

**Diagnosis summary flowchart:**

```
Lag growing?
├── Uneven across partitions → Hot partition (skewed key)
├── Some partitions unassigned → Insufficient consumers / crashed consumer
├── Even across all partitions
│   ├── Processing time increased → Slow consumer / dependency bottleneck
│   └── Frequent state changes → Rebalancing storm
└── Check all of the above — often multiple causes compound
```

</details>

<details>
<summary>21. Messages are being processed multiple times causing duplicate records — walk through the diagnosis: determine if it's caused by visibility timeout too short (SQS), consumer rebalancing (Kafka), long processing time exceeding ack deadline (Pub/Sub), or a missing idempotency check. Show the fix for each scenario</summary>

Start by confirming duplicates are actually happening (not just suspected), then narrow down the cause by platform.

**Step 1: Confirm and quantify duplicates**

Check for duplicate records in the database — look for the same entity written multiple times with slightly different timestamps:

```sql
SELECT order_id, COUNT(*), MIN(created_at), MAX(created_at)
FROM payments
GROUP BY order_id
HAVING COUNT(*) > 1
ORDER BY MAX(created_at) DESC;
```

The time gap between duplicates tells you a lot: seconds apart suggests redelivery from the broker; minutes apart suggests a rebalance or restart.

**Step 2: Diagnose by platform**

**Scenario A: SQS visibility timeout too short**

The visibility timeout is how long SQS hides a message from other consumers after one consumer receives it. If processing takes longer than the timeout, SQS makes the message visible again and another consumer picks it up.

Diagnosis:
```bash
# Check the current visibility timeout
aws sqs get-queue-attributes --queue-url $QUEUE_URL \
  --attribute-names VisibilityTimeout
# If VisibilityTimeout=30 and processing takes 45s → duplicates

# Check ApproximateNumberOfMessagesNotVisible — messages currently being processed
# If this drops unexpectedly, messages are becoming visible again before processing completes
```

Fix:
```bash
# Increase visibility timeout to 2-3x your max processing time
aws sqs set-queue-attributes --queue-url $QUEUE_URL \
  --attributes VisibilityTimeout=120

# Or extend dynamically during processing (heartbeat pattern)
```

```typescript
// Extend visibility timeout during long processing (AWS SDK v3)
import { SQSClient, ChangeMessageVisibilityCommand, Message } from '@aws-sdk/client-sqs';

const sqs = new SQSClient({});

async function processWithHeartbeat(message: Message): Promise<void> {
  const heartbeat = setInterval(async () => {
    await sqs.send(new ChangeMessageVisibilityCommand({
      QueueUrl: QUEUE_URL,
      ReceiptHandle: message.ReceiptHandle!,
      VisibilityTimeout: 60, // extend by 60s
    }));
  }, 20_000); // extend every 20s

  try {
    await processOrder(message);
  } finally {
    clearInterval(heartbeat);
  }
}
```

**Scenario B: Kafka consumer rebalancing**

When a rebalance occurs, partitions are revoked from one consumer and assigned to another. If the first consumer was mid-processing and hadn't committed the offset, the new consumer reprocesses those messages.

Diagnosis:
- Check consumer logs for frequent "Revoking partitions" messages
- Check if `max.poll.interval.ms` is being exceeded (consumer is kicked from the group)
- Correlate duplicate timestamps with rebalance events

Fix:
```typescript
const consumer = kafka.consumer({
  groupId: 'order-processing-group',
  sessionTimeout: 30000,      // increase if consumers are slow to heartbeat
  heartbeatInterval: 3000,    // send heartbeats frequently to avoid being kicked
  // Note: Kafka's max.poll.interval.ms is not exposed in KafkaJS — session timeout
  // and heartbeat interval are the knobs for detecting slow consumers.
  // Cooperative sticky assignment is a Kafka protocol feature not available in KafkaJS;
  // KafkaJS uses round-robin by default. The Java client supports CooperativeStickyAssignor.
});

// Reduce batch size so each poll completes faster
await consumer.run({
  eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
    for (const message of batch.messages) {
      await processOrder(JSON.parse(message.value!.toString()));
      resolveOffset(message.offset); // commit progress within batch
      await heartbeat();              // prevent session timeout
    }
  },
});
```

**Scenario C: Pub/Sub ack deadline exceeded**

Pub/Sub redelivers if the subscriber doesn't acknowledge within the ack deadline (default 10s, max 600s). Slow processing causes redelivery.

Diagnosis:
- Check the `pubsub.googleapis.com/subscription/ack_message_count` vs `pubsub.googleapis.com/subscription/num_outstanding_messages` metrics
- Look for `expired_ack_deadlines` metric spikes

Fix:
```typescript
// Extend ack deadline using modifyAckDeadline
const subscription = pubsub.subscription('order-sub', {
  ackDeadline: 60,           // initial deadline in seconds
  flowControl: {
    maxMessages: 10,          // limit concurrent messages
  },
});

// Pub/Sub client libraries automatically extend deadlines (lease management)
// Ensure you're using the official client library, not raw pull
subscription.on('message', async (message) => {
  try {
    await processOrder(JSON.parse(message.data.toString()));
    message.ack();
  } catch (err) {
    message.nack();            // redeliver for retry
  }
});
```

**Scenario D: Missing idempotency check (the real root cause)**

All of the above are timing fixes. The fundamental fix is making the consumer idempotent so duplicates don't cause duplicate records regardless of the cause (as covered in questions 10 and 15).

```typescript
// The permanent fix: idempotent write
await db.query(
  `INSERT INTO payments (order_id, amount, event_id)
   VALUES ($1, $2, $3)
   ON CONFLICT (event_id) DO NOTHING`,  // dedup key prevents duplicate records
  [event.orderId, event.amount, event.eventId]
);
```

**The right approach:** Fix the timing issue (increase timeout/deadline) to reduce unnecessary reprocessing AND add idempotency as a safety net. Never rely solely on timing — networks are unreliable.

</details>

<details>
<summary>22. Your team is building an order processing system that needs strict per-customer ordering, replay capability for reprocessing failed batches, and must handle 10K events/second — evaluate Kafka, GCP Pub/Sub, and SQS/SNS against these requirements, show the architecture sketch for each option, and justify your final recommendation with specific tradeoffs (cost, operational burden, ordering guarantees, replay support)</summary>

**Requirements summary:**
1. Strict per-customer ordering (all events for a customer processed in sequence)
2. Replay capability (reprocess failed batches)
3. 10K events/second throughput
4. Order processing (presumably multiple downstream consumers — shipping, payments, analytics)

---

**Option A: Apache Kafka (self-managed or Confluent Cloud)**

```
Order Service ──► Kafka Topic: order-events (24 partitions, key=customerId)
                      │
                      ├──► shipping-group (3 consumers)
                      ├──► payment-group (3 consumers)
                      └──► analytics-group (2 consumers)
```

| Requirement | How Kafka handles it |
|---|---|
| Per-customer ordering | Partition by `customerId` — all events for a customer go to the same partition, processed in order |
| Replay | Rewind consumer group offsets to any point in time. Retention configurable (7 days, 30 days, forever) |
| 10K events/s | Easily handles this — Kafka is designed for millions of events/s. 24 partitions is plenty |
| Fan-out | Multiple consumer groups each see all messages independently |

**Tradeoffs:**
- **Ordering risk:** If one customer generates disproportionate traffic, their partition becomes a hot spot (as covered in question 6). Monitor partition distribution.
- **Operational burden:** Self-managed Kafka requires ZooKeeper (or KRaft), broker management, partition rebalancing, monitoring. Confluent Cloud reduces this but costs more.
- **Cost:** Self-managed is cheapest at scale; Confluent Cloud runs ~$1-3 per GB ingested.

---

**Option B: GCP Pub/Sub**

```
Order Service ──► Pub/Sub Topic: order-events (ordering key=customerId)
                      │
                      ├──► shipping-subscription (push to Cloud Run)
                      ├──► payment-subscription (pull, 3 consumers)
                      └──► analytics-subscription (push to BigQuery)
```

| Requirement | How Pub/Sub handles it |
|---|---|
| Per-customer ordering | Ordering keys — messages with the same key are delivered in order to the same subscriber. But: throughput is limited to ~1 MB/s per ordering key region |
| Replay | Seek to a timestamp on a subscription — replays all messages from that point. 7-day retention (configurable up to 31 days) |
| 10K events/s | Handles this easily — Pub/Sub scales automatically with no partition management |
| Fan-out | Multiple subscriptions per topic, each independent |

**Tradeoffs:**
- **Ordering limitation:** Ordering keys work but with throughput constraints. If one customer generates very high volume, you may hit the per-key throughput limit.
- **Operational burden:** Fully managed — zero infrastructure to operate. Biggest advantage.
- **Cost:** ~$40 per TiB of messages published + delivery. At 10K events/s with 1KB messages, ~$25-30/month. Very cost-effective.
- **Replay limitation:** Can only seek to a timestamp, not to a specific offset. Less precise than Kafka.

---

**Option C: AWS SNS + SQS**

```
Order Service ──► SNS Topic: order-events
                      │
                      ├──► SQS: shipping-queue (FIFO, group=customerId) → 3 consumers
                      ├──► SQS: payment-queue (FIFO, group=customerId) → 3 consumers
                      └──► SQS: analytics-queue (standard) → 2 consumers
```

| Requirement | How SNS+SQS handles it |
|---|---|
| Per-customer ordering | FIFO queues with `MessageGroupId=customerId`. Messages within a group are strictly ordered |
| Replay | No native replay. Once a message is consumed and deleted, it's gone. Would need a separate archive (S3, DynamoDB) |
| 10K events/s | FIFO queues: 300 messages/s per group without batching, 3,000/s with batching. Standard queues: unlimited. This is the blocker |
| Fan-out | SNS fans out to multiple SQS queues |

**Tradeoffs:**
- **Ordering + throughput conflict:** FIFO queues cap at 300 msg/s per message group (3,000 with batching). At 10K events/s across all customers, this works if traffic is well-distributed. But if a few customers are high-volume, their groups hit the ceiling.
- **No replay:** The most significant gap. You'd need to archive every message to S3 and build custom replay tooling.
- **Operational burden:** Fully managed, very simple to operate.
- **Cost:** $0.40 per million SQS requests. At 10K/s, ~$10-15/month. Cheapest option.

---

**Recommendation: Kafka (Confluent Cloud or Amazon MSK Serverless)**

| Criterion | Kafka | Pub/Sub | SNS+SQS |
|---|---|---|---|
| Per-customer ordering | Strong (partition key) | Good (ordering keys, throughput limits) | Limited (FIFO caps) |
| Replay | Best (offset rewind, any point) | Good (timestamp seek, 31-day max) | None natively |
| 10K events/s | Trivial | Trivial | Constrained with FIFO |
| Fan-out | Consumer groups | Subscriptions | SNS+SQS queues |
| Operational burden | Medium (use managed) | Low | Low |
| Cost | Medium | Low | Lowest |

**Why Kafka wins here:**

The combination of strict ordering + replay + high throughput is Kafka's sweet spot. The requirements explicitly call for replay capability for reprocessing failed batches, which eliminates SQS. Pub/Sub is viable but its ordering key throughput limits and coarser replay granularity make Kafka the safer choice for strict ordering at scale.

**Mitigate Kafka's operational cost** by using a managed offering (Confluent Cloud, Amazon MSK Serverless, or Aiven). This gets you Kafka's semantics without running brokers.

**If you're already on GCP:** Pub/Sub is a strong second choice. Its ordering keys handle per-customer ordering, and timestamp-based seek covers most replay use cases. The simpler operational model might outweigh Kafka's finer-grained control, depending on how strict "strict ordering" truly needs to be.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>23. Tell me about a time you introduced a message queue or streaming platform to a system — what problem were you solving, what technology did you choose and why, and what unexpected challenges did you encounter?</summary>

**What the interviewer is looking for:**
- You can identify when messaging is the right solution (not just "we needed a queue")
- You evaluated alternatives and made a deliberate technology choice with reasoning
- You encountered real-world complexity beyond the "happy path" tutorial setup
- You can articulate tradeoffs honestly, including things that didn't go smoothly

**Suggested structure (STAR with technical depth):**

1. **Situation:** What system existed, what pain point emerged
2. **Technology choice:** What you picked and why (include what you considered and rejected)
3. **Implementation:** Key design decisions (partition strategy, delivery guarantees, consumer patterns)
4. **Unexpected challenges:** What surprised you — this is the most important part
5. **Outcome:** What improved, what you'd do differently

**Key points to hit:**
- Show you considered alternatives first (as covered in question 2) — direct HTTP, database polling, task queues
- Explain WHY the chosen technology over others (not just "Kafka is popular")
- Be specific about unexpected challenges — ordering issues, duplicate handling, operational complexity, consumer lag during traffic spikes, schema evolution pain
- Show learning: what you'd do differently with hindsight

**Example outline to personalize:**

> "We had an order processing service that synchronously called 4 downstream services (shipping, inventory, notifications, analytics). When any one was slow or down, the entire order flow degraded. We needed to decouple these.
>
> We evaluated BullMQ (simpler, we already had Redis) vs GCP Pub/Sub (managed, team was on GCP). Chose Pub/Sub because we needed fan-out to multiple independent services, not just task distribution.
>
> The unexpected challenge was message ordering. We assumed Pub/Sub delivered in order, but it doesn't by default. We discovered this when inventory decrements arrived before the order creation event. Had to add ordering keys and redesign our consumer to handle out-of-order delivery gracefully.
>
> Second surprise was the operational monitoring gap — we had no visibility into consumer lag until messages started piling up during a Black Friday spike. We built custom lag dashboards and alerting after that incident.
>
> In hindsight, I'd invest in observability (lag metrics, DLQ monitoring) from day one, not as a reaction to an incident."

</details>

<details>
<summary>24. Describe a time you dealt with message ordering, duplication, or loss in production — what was the symptom, how did you diagnose it, and what changes did you make to prevent recurrence?</summary>

**What the interviewer is looking for:**
- You can diagnose production messaging issues methodically (not just guessing)
- You understand the underlying cause, not just the symptom
- Your fix addresses the root cause, not just the immediate symptom
- You added safeguards to prevent recurrence (monitoring, idempotency, tests)

**Suggested structure:**

1. **Symptom:** What was observed (duplicate records, missing data, wrong state)
2. **Investigation:** How you narrowed down the cause — logs, metrics, broker state
3. **Root cause:** The specific mechanism (rebalance, timeout, missing ack, partition skew)
4. **Fix:** Both immediate (hotfix) and long-term (architectural change)
5. **Prevention:** What guardrails you added (monitoring, idempotency, alerting)

**Key points to hit:**
- Be specific about the diagnostic process — "I checked consumer lag metrics and saw partition 3 had 50K lag while others were at 0" is much stronger than "we found a bug"
- Show you understand WHY the issue happened at the distributed systems level
- Demonstrate defense-in-depth thinking: the fix should include both the timing/config fix AND an idempotency/ordering safeguard
- Mention monitoring you added to catch this earlier next time

**Example outline to personalize:**

> "We noticed duplicate payment charges appearing in our billing reports — customers were being charged twice for the same order. The duplicates were 30-60 seconds apart.
>
> I started by querying the payments table for duplicates, which showed ~0.5% of orders affected. The timing gap pointed to message redelivery rather than an application bug.
>
> Checked consumer group state and found frequent rebalances correlating with the duplicate timestamps. Our Kafka consumer was doing a database write that sometimes took 35 seconds (slow query under load), exceeding `max.poll.interval.ms` of 30 seconds. The consumer was kicked from the group, a rebalance occurred, and the new consumer reprocessed uncommitted messages.
>
> Immediate fix: Increased `max.poll.interval.ms` to 120s and optimized the slow query (missing index). Long-term fix: Added an idempotency check using the event ID as a dedup key in the payments table (`ON CONFLICT DO NOTHING`), so even if redelivery happens, no duplicate charge occurs.
>
> Added alerting on consumer rebalance frequency and p99 processing time so we'd catch this pattern early."

</details>

<details>
<summary>25. Tell me about a time you had to scale a messaging system to handle significantly more traffic — what was the bottleneck, what approach did you take (more partitions, more consumers, different technology), and what tradeoffs did you accept?</summary>

**What the interviewer is looking for:**
- You can identify the actual bottleneck (not just "we needed more scale")
- You understand the scaling levers for your specific messaging system
- You made deliberate tradeoff decisions (ordering vs throughput, cost vs latency)
- You validated the scaling change with metrics, not just hope

**Suggested structure:**

1. **Context:** What traffic increase triggered the scaling need (organic growth, new feature, seasonal spike)
2. **Bottleneck identification:** How you found the actual constraint — was it consumer throughput, partition count, broker capacity, or network?
3. **Approach:** What scaling lever you pulled and why that one over alternatives
4. **Tradeoffs:** What you gave up or accepted as a consequence
5. **Validation:** How you confirmed the scaling worked (load test, gradual rollout, metrics)

**Key points to hit:**
- Show you diagnosed before acting — "we added partitions" is weaker than "consumer lag metrics showed all consumers at 100% CPU, so adding more consumers (and thus partitions) was the right lever"
- Demonstrate awareness of non-obvious consequences: adding Kafka partitions changes key-to-partition mapping (existing ordering assumptions break); adding consumers triggers rebalances; changing technology has migration cost
- Mention the tradeoff explicitly — interviewers want to hear you weigh costs, not just pick the "biggest" option

**Example outline to personalize:**

> "We onboarded a new high-volume tenant that tripled our event throughput from 3K to 10K events/second. Consumer lag started growing steadily — our 6 consumers couldn't keep up.
>
> Diagnosis: Each consumer was doing a synchronous database write per message (~10ms). At 6 consumers, max throughput was ~600 msg/s per consumer = 3,600 total. We needed 10K.
>
> We had two options: (1) optimize the consumer to batch writes (higher throughput per consumer), or (2) add more partitions and consumers (horizontal scaling). We chose both.
>
> First, we batched database writes — accumulate 100 messages, bulk insert. This got us to ~2,000 msg/s per consumer. Then we increased partitions from 6 to 16 and scaled consumers to 12.
>
> The tradeoff with batching: we accepted slightly higher latency (waiting to fill a batch) and more complex error handling (if one message in a batch fails, we had to handle partial failures). The tradeoff with more partitions: we had to rebalance all consumers, causing a brief processing pause, and partition key distribution shifted slightly.
>
> We validated with a load test at 15K events/s before the tenant went live. Post-launch, consumer lag stayed under 100 messages consistently."

</details>

