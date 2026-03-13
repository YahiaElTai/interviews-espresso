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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. When should you use a message queue vs the alternatives (direct HTTP calls, database-backed job queues, cron jobs, task queue libraries like Bull/BullMQ) — what signals tell you a full message broker adds real value (decoupling, load leveling, retry isolation), when is a task queue library the right middle ground, and what signals tell you messaging is adding unnecessary complexity?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. What are the delivery guarantee levels (at-most-once, at-least-once, exactly-once) — what does each actually mean in practice, why is exactly-once so hard to achieve (two generals problem, network partitions), and why do most production systems settle for at-least-once with idempotent consumers?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. Why do messaging systems distinguish between competing consumers (work queues) and fan-out (pub-sub) — what problem does each pattern solve, when do you need both in the same system, and what are the tradeoffs of implementing fan-out at the broker level vs in application code?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How do different messaging systems implement competing consumers and fan-out — compare the SNS+SQS fan-out pattern, RabbitMQ exchange types, Kafka consumer groups with multiple topics, and GCP Pub/Sub subscriptions, and what tradeoffs does each approach make?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why did Kafka choose an append-only log as its core abstraction instead of a traditional message queue — how do partitions enable horizontal scaling, why is partition key strategy critical for ordering guarantees, and what happens to message ordering when you have more consumers than partitions?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do Kafka consumer groups work and why does rebalancing cause processing pauses — what triggers a rebalance, how does the cooperative sticky assignor reduce disruption compared to the eager protocol, and what are the tradeoffs of static group membership?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why is there a fundamental tension between ordering and parallelism — why can't you have both easily, how does partition-level ordering work as a compromise, and when should you relax ordering requirements entirely?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How does Kafka's offset management work and why is it central to replay capability — what are committed offsets, how do offset reset policies (earliest vs latest) affect consumer behavior on restart, when would you rewind a consumer group to reprocess messages, and what are compacted topics and how do they provide latest-state lookups instead of full event history?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How do you achieve idempotent message processing — what are the strategies (deduplication IDs, idempotent database writes, transactional outbox), how do deduplication windows work, and why is idempotency required even when the broker claims exactly-once delivery?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why is backpressure critical in messaging systems — what is consumer lag, how do pull-based consumers (Kafka, SQS) handle backpressure differently from push-based consumers (Pub/Sub push subscriptions, RabbitMQ), and what are the tradeoffs of each model when a consumer falls behind?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What are dead letter queues and how should you use them — how do messages get routed to a DLQ (max retry exceeded, deserialization failure, schema mismatch), what triage strategies help you analyze DLQ contents, and how do you safely replay messages from the DLQ back to the main queue?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What are the tradeoffs between message serialization formats (JSON, Avro, Protobuf) — how does each handle schema evolution (adding fields, removing fields, renaming), what is a schema registry and why does it matter for Avro/Protobuf, and when is JSON's simplicity worth the performance and schema safety tradeoff?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Queue Configuration & Patterns

<details>
<summary>14. Set up a Kafka topic with proper partitioning for an order processing system — choose the partition key strategy (order ID for per-entity ordering), configure the producer with appropriate acks and retries, set up a consumer group, and demonstrate what happens when you add a consumer to the group (rebalancing)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Implement an idempotent message consumer that handles at-least-once delivery — show the consumer code that uses a deduplication ID stored in the database, wraps processing in a transaction, and handles the case where the same message arrives twice. Compare this with using the transactional outbox pattern on the producer side</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Configure competing consumers and fan-out for the same event stream — show how to set up a Kafka topic where order events are consumed by both a shipping service (competing consumers sharing the work) and an analytics service (independent consumer group seeing all events), and explain why consumer groups enable this pattern</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Set up a dead letter queue with proper routing — show the configuration that routes messages to a DLQ after N failed attempts, implement a DLQ consumer that logs failed messages with context for triage, and demonstrate how to replay messages from the DLQ back to the original queue after fixing the issue</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Implement message serialization with schema evolution — show a Protobuf or Avro message definition, demonstrate adding a new optional field (backward-compatible), show how a consumer running the old schema handles messages with the new field, and explain what happens with a breaking change (removing a required field)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Configure backpressure handling for a consumer that processes messages slower than the producer publishes — show consumer lag monitoring, configure prefetch/batch limits to prevent memory exhaustion, implement autoscaling consumers based on lag metrics, and explain when to apply backpressure to the producer instead</summary>

<!-- Answer will be added later -->

</details>

## Practical — Troubleshooting & Decision-Making

<details>
<summary>20. Consumer lag is growing steadily on a Kafka topic — walk through the diagnosis: check if it's a slow consumer (processing bottleneck), uneven partition distribution (hot partition from a skewed key), consumer rebalancing storms, or insufficient consumer instances. Show the commands and metrics for each check</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Messages are being processed multiple times causing duplicate records — walk through the diagnosis: determine if it's caused by visibility timeout too short (SQS), consumer rebalancing (Kafka), long processing time exceeding ack deadline (Pub/Sub), or a missing idempotency check. Show the fix for each scenario</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Your team is building an order processing system that needs strict per-customer ordering, replay capability for reprocessing failed batches, and must handle 10K events/second — evaluate Kafka, GCP Pub/Sub, and SQS/SNS against these requirements, show the architecture sketch for each option, and justify your final recommendation with specific tradeoffs (cost, operational burden, ordering guarantees, replay support)</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>23. Tell me about a time you introduced a message queue or streaming platform to a system — what problem were you solving, what technology did you choose and why, and what unexpected challenges did you encounter?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>24. Describe a time you dealt with message ordering, duplication, or loss in production — what was the symptom, how did you diagnose it, and what changes did you make to prevent recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Tell me about a time you had to scale a messaging system to handle significantly more traffic — what was the bottleneck, what approach did you take (more partitions, more consumers, different technology), and what tradeoffs did you accept?</summary>

<!-- Answer framework will be added later -->

</details>

