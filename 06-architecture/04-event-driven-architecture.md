# Event-Driven Architecture

> **20 questions** — 11 theory, 5 practical, 4 experience

- Event-driven architecture vs synchronous request-response: decoupling benefits, eventual consistency tradeoffs, debugging complexity
- Core building blocks: events, commands, producers, consumers, brokers, topics/queues
- Eventual consistency: when to tolerate it vs when to require strong consistency
- Event ordering: partition-level ordering, handling out-of-order events, version-based conflict resolution, when ordering matters vs when it doesn't
- Event sourcing: storing events vs current state, event store vs message broker (different infrastructure for different purposes), event schema versioning and upcasting, snapshot optimization, replay complexity
- CQRS: separating reads from writes, read model projections, and when to avoid it
- Idempotent consumers: idempotency keys, deduplication windows, transactional processing, dead letter queues as a recovery mechanism
- Transactional outbox pattern for solving the dual-write problem
- Sagas: choreography vs orchestration, choreography's hidden coupling, compensating actions, and failure handling
- When to avoid event-driven architecture: signs of over-engineering, minimum complexity threshold, operational overhead vs value

---

## Foundational

<details>
<summary>1. Why would you choose event-driven architecture over synchronous request-response — what decoupling benefits does it provide, what are the eventual consistency tradeoffs you must accept, and why does debugging become significantly harder with async event flows?</summary>

**Decoupling benefits:**

- **Temporal decoupling**: The producer doesn't need the consumer to be running at the time of the event. An order service publishes `OrderCreated` and moves on — the notification service can process it whenever it comes back online.
- **Deployment decoupling**: Services evolve independently. Adding a new consumer (analytics, fraud detection) requires zero changes to the producer.
- **Load decoupling**: A broker absorbs traffic spikes. If the payment service gets 10x traffic, events queue up rather than cascading failures upstream.

**Eventual consistency tradeoffs:**

You give up the guarantee that all parts of the system reflect the same state at the same time. After an order is placed, the inventory view might still show the old count for a few seconds (or longer during failures). This means:

- Users might see stale data — you need UI patterns to handle this (optimistic updates, "processing" states).
- Business logic that spans services can't rely on reads being current — you need idempotent consumers and conflict resolution.
- Debugging state inconsistencies requires understanding which events have been processed and which haven't.

**Why debugging is harder:**

- **No single request trace**: A synchronous call chain has one request ID flowing through. Event-driven flows scatter across multiple consumers, each processing independently. A single user action might trigger a chain of events across 5 services.
- **Time becomes a variable**: Events can be delayed, reordered, or replayed. The bug might not manifest until minutes after the triggering action.
- **No stack trace**: When something fails in a consumer, the original producer context is gone. You need correlation IDs, distributed tracing, and structured logging to reconstruct the flow.
- **Invisible coupling**: The dependency between producer and consumer isn't in the code — it's in the event schema and the implicit contract of "who listens to what."

</details>

<details>
<summary>2. Why do event-driven systems need distinct building blocks (events vs commands, producers vs consumers, brokers, topics vs queues) — what role does each play, why is the distinction between events and commands important for system design, and how do topics and queues differ in how they route messages?</summary>

**Core building blocks:**

- **Event**: A fact about something that happened — immutable, past tense. `OrderPlaced`, `PaymentProcessed`. The producer doesn't know or care who consumes it.
- **Command**: A request for something to happen — imperative, directed at a specific consumer. `ProcessPayment`, `SendEmail`. The sender expects a specific service to act on it.
- **Producer**: Publishes events or sends commands. Owns the schema of what it emits.
- **Consumer**: Subscribes to events or handles commands. Responsible for its own error handling, idempotency, and retry logic.
- **Broker**: The intermediary (Kafka, RabbitMQ, GCP Pub/Sub) that stores and routes messages between producers and consumers. Handles delivery guarantees, retention, and consumer management.

**Why the event vs command distinction matters:**

Events and commands encode different architectural relationships:

- **Events are broadcast**: One producer, zero-to-many consumers. The order service doesn't know that shipping, analytics, and notifications all listen to `OrderPlaced`. This is true decoupling.
- **Commands are directed**: One sender, one receiver. The order service explicitly tells the payment service to charge the customer. This is a known dependency.

Conflating them causes problems. If you model "send notification" as an event, you lose the ability to reason about who's responsible for sending it. If you model "order placed" as a command sent to each downstream service, you've coupled the order service to every consumer.

**Topics vs queues:**

- **Topic (pub-sub)**: Every subscriber gets every message. Kafka topics, SNS topics, Pub/Sub topics. Good for fan-out — one event, many independent consumers. Each consumer group (Kafka) or subscription (Pub/Sub) maintains its own position.
- **Queue (point-to-point)**: Each message goes to exactly one consumer. SQS, RabbitMQ queues. Good for work distribution — multiple instances of the same service competing to process messages. Load is shared, not duplicated.

In practice, you often combine both: a topic for fan-out to different services, with each service running multiple instances as competing consumers on their subscription/consumer group.

</details>

<details>
<summary>3. When should you tolerate eventual consistency vs require strong consistency — what types of operations can safely be eventually consistent (notifications, analytics, search indexing), what must be strongly consistent (financial transactions, inventory), and how do you communicate consistency boundaries to the rest of the team?</summary>

**Safe for eventual consistency** — operations where a short delay between "truth" and "visible state" causes no real harm:

- **Notifications**: An email arriving 30 seconds late is fine.
- **Analytics and reporting**: Dashboards showing data from a few seconds ago is acceptable.
- **Search indexing**: Elasticsearch being a few seconds behind the source of truth is standard.
- **Recommendation engines**: Stale data has minimal impact on suggestions.
- **Caching layers**: By definition eventually consistent — TTL-based invalidation is a known tradeoff.

**Requires strong consistency** — operations where stale reads cause incorrect or harmful outcomes:

- **Financial transactions**: A bank balance read must reflect all completed transactions. Double-spending from stale reads is unacceptable.
- **Inventory with limited stock**: If 1 item remains and 2 users see "in stock" simultaneously, you get overselling. This needs atomic decrement or optimistic locking.
- **Authentication and authorization**: A revoked token or permission change must take effect immediately — not "eventually."
- **Booking/reservation systems**: Two people booking the same seat, room, or time slot.

**The gray area** — operations where you choose based on business tolerance:

- **User profile updates**: Usually eventual consistency is fine, but "I changed my name and the next page still shows the old one" (read-your-own-writes) frustrates users. Solve with session-sticky reads or write-through cache.
- **Order status**: The order service is the source of truth (strongly consistent), but the customer-facing status page can be eventually consistent via a read model.

**Communicating consistency boundaries:**

- **Document it explicitly**: Each service should declare what consistency model it provides. "Order reads are strongly consistent. Order search results may lag by up to 5 seconds."
- **Name the boundaries**: Use terms like "source of truth" vs "projection" or "read model" in architecture docs and code.
- **Make it visible in APIs**: If an endpoint returns eventually consistent data, document it. Some teams use response headers (`X-Consistency: eventual`) or API naming conventions.
- **ADRs (Architecture Decision Records)**: When you choose eventual consistency for something non-obvious, write down why and what the acceptable lag is.

</details>

## Conceptual Depth

<details>
<summary>4. Why would you store events instead of current state in an event-sourced system — what benefits does this provide (complete audit trail, replay, temporal queries), what makes schema evolution hard (versioned event types, upcasting), how do snapshots optimize replay performance as event volume grows, and what replay complexity issues remain even with snapshots?</summary>

**Why store events instead of current state:**

In a traditional system, you store the latest state: `order.status = 'shipped'`. In event sourcing, you store the sequence of events that produced that state: `OrderCreated -> PaymentReceived -> OrderShipped`. Current state is derived by replaying events.

**Benefits:**

- **Complete audit trail**: Every state change is recorded. You know not just that the order is shipped, but when payment was received, whether the address was changed, and in what order things happened. This is invaluable for compliance, debugging, and dispute resolution.
- **Replay**: You can rebuild any read model or projection from scratch by replaying events. Discovered a bug in your analytics projection? Fix the code, replay all events, and get corrected data.
- **Temporal queries**: "What was this customer's cart at 3:47 PM?" Replay events up to that timestamp. Traditional state stores can't answer this without separate audit logging.
- **Event-driven integration**: Events are already the source of truth, so publishing them to other services is natural — no dual-write problem between "update state" and "publish event."

**Why schema evolution is hard:**

Events are immutable — once stored, they can't be changed. But your domain model evolves:

- **Versioned event types**: `OrderCreatedV1` has `{ amount: number }`. Six months later, `OrderCreatedV2` has `{ amount: number, currency: string }`. Your event store now contains both versions.
- **Upcasting**: When replaying, you transform old events into the current schema. An upcaster converts V1 to V2 by adding `currency: 'USD'` as a default. This gets complex when you have many versions and the transformations chain.
- **Breaking changes**: Renaming a field or splitting an event into two requires careful migration. You can't just ALTER TABLE — those old events are permanent.

**Snapshot optimization:**

Replaying thousands of events to rebuild an entity is slow. Snapshots help:

- Periodically save the current state as a snapshot (e.g., every 100 events).
- To rebuild, load the latest snapshot and replay only the events after it.
- An order with 500 events might have a snapshot at event 400, so you only replay 100 events instead of 500.

**Replay complexity that remains even with snapshots:**

- **Snapshot schema also evolves**: Snapshots are state — when your domain model changes, old snapshots may be incompatible. You need snapshot migration or invalidation.
- **Side effects during replay**: If event handlers send emails or call external APIs, replaying them would duplicate those side effects. You need a "replay mode" flag to suppress side effects.
- **Ordering and concurrency**: Replaying events across multiple aggregates in the right order, especially when projections depend on events from multiple streams, is non-trivial.
- **Time and volume**: Even with snapshots, rebuilding all projections for millions of entities takes significant time and compute. You need tooling for parallel replay and progress tracking.

</details>

<details>
<summary>5. Why is an event store (like EventStoreDB or a custom append-only table) a different piece of infrastructure from a message broker (like Kafka or RabbitMQ) — what problem does each solve, when do you need both in an event-sourced system, and when is a message broker alone sufficient without a dedicated event store?</summary>

**Different problems, different infrastructure:**

| Aspect | Event Store | Message Broker |
|--------|------------|----------------|
| **Purpose** | Permanent record of all events for an entity (source of truth) | Transient delivery of messages between services |
| **Query model** | Read events for a specific aggregate by ID, in order | Consume events from a topic/queue, typically in arrival order |
| **Retention** | Forever — events are the data | Configurable — often days/weeks, then deleted |
| **Consistency** | Optimistic concurrency per stream (e.g., expected version checks) | No per-entity concurrency control |
| **Examples** | EventStoreDB, custom Postgres append-only table, DynamoDB single-table | Kafka, RabbitMQ, GCP Pub/Sub, SQS/SNS |

**An event store** answers: "Give me all events for Order #123 in order." It supports rebuilding an entity by replaying its event stream, enforcing business rules via optimistic concurrency (reject a write if the stream has been modified since you last read it).

**A message broker** answers: "Deliver this event to all interested consumers." It handles routing, delivery guarantees, consumer groups, and backpressure. It doesn't care about per-entity streams.

**When you need both:**

In a full event-sourced system, you typically need both:

1. The event store persists events as the source of truth for your domain.
2. The message broker distributes those events to other services for building read models, triggering workflows, and integration.

The write side appends to the event store, then publishes to the broker (often via the transactional outbox pattern to avoid dual writes).

**When a message broker alone is sufficient:**

You don't need a dedicated event store when:

- **You're doing event-driven architecture without event sourcing**: Services communicate via events, but each service stores its own current state in a normal database. The broker handles delivery, and you don't need per-entity event replay.
- **Kafka with long retention**: Kafka's append-only log with infinite retention can serve as a pseudo-event store for some use cases. However, it lacks per-entity stream queries — reading all events for one order means scanning the entire partition. Compacted topics give you latest-state lookups but not full history.
- **Simple pub-sub patterns**: Notifications, analytics pipelines, and async task processing don't need event replay. A broker alone is sufficient.

**Rule of thumb**: If you need to rebuild entity state from events (event sourcing), you need an event store. If you just need to react to events as they happen (event-driven), a broker is enough.

</details>

<details>
<summary>6. Why would you separate read and write models using CQRS — how do read model projections work, what consistency challenges arise between the write side and read side, and when is CQRS unnecessary complexity for your use case?</summary>

**Why separate reads from writes:**

In many systems, read and write workloads have fundamentally different shapes:

- **Writes** need validation, business rules, and consistency guarantees. They operate on normalized, aggregate-level data.
- **Reads** need speed, denormalization, and flexible query patterns. A product listing page might need data from products, inventory, pricing, and reviews — all pre-joined.

CQRS lets you optimize each side independently. The write model enforces invariants. The read model is a denormalized projection optimized for specific queries, potentially stored in a different database entirely (e.g., writes to Postgres, reads from Elasticsearch or Redis).

**How read model projections work:**

1. The write side processes a command and emits events: `ProductCreated`, `PriceChanged`, `StockUpdated`.
2. A projection handler subscribes to these events and updates a denormalized read model:

```typescript
async function handlePriceChanged(event: PriceChanged) {
  await readDb.query(
    `UPDATE product_listing
     SET price = $1, currency = $2, updated_at = $3
     WHERE product_id = $4`,
    [event.newPrice, event.currency, event.timestamp, event.productId]
  );
}
```

3. Query handlers read directly from this projection — no joins, no aggregation, just fast lookups.

**Consistency challenges:**

- **Stale reads**: After a write, the read model hasn't been updated yet. A user changes their address, refreshes the page, and sees the old address. Solutions: read-your-own-writes (query the write side for the current user after a write), optimistic UI updates, or polling until the projection catches up.
- **Projection lag monitoring**: You need to track how far behind each projection is. If the gap grows, reads become increasingly stale.
- **Projection rebuild**: If a projection has a bug, you need to replay all events to rebuild it. This can take hours for large datasets.
- **Ordering**: The projection must process events in order to produce correct state. Out-of-order processing can corrupt the read model.

**When CQRS is unnecessary complexity:**

- **Simple CRUD applications**: If your reads and writes are roughly the same shape and your query patterns are straightforward, CQRS adds two data models, a synchronization mechanism, and consistency headaches for no benefit.
- **Low read/write asymmetry**: If you don't need to scale reads and writes independently, a single model is simpler.
- **Small team**: CQRS increases the surface area of the system. If you have 2-3 developers, the operational overhead likely outweighs the benefits.
- **When you just need a cache**: Sometimes what looks like a need for CQRS is really just a need for a caching layer in front of your database.

</details>

<details>
<summary>7. Why must consumers in event-driven systems be idempotent — how do idempotency keys and deduplication windows work, what does transactional processing (consuming + writing in one transaction) guarantee, and how do dead letter queues serve as a recovery mechanism for messages that repeatedly fail?</summary>

**Why idempotency is mandatory:**

In any at-least-once delivery system (which is what most production systems use), the same message can be delivered multiple times. This happens due to:

- Consumer crashes after processing but before acknowledging.
- Network timeouts where the broker doesn't receive the ack.
- Consumer group rebalancing in Kafka.
- Visibility timeout expiring in SQS before processing completes.

If your consumer isn't idempotent, a redelivered `PaymentProcessed` event could charge a customer twice.

**Idempotency keys and deduplication windows:**

An idempotency key is a unique identifier for each event (usually the event ID or a combination of aggregate ID + sequence number). The consumer checks whether it's already processed this key before doing any work:

```typescript
async function processEvent(event: OrderCreatedEvent) {
  const alreadyProcessed = await db.query(
    'SELECT 1 FROM processed_events WHERE event_id = $1',
    [event.id]
  );
  if (alreadyProcessed.rows.length > 0) return; // skip duplicate

  // Process the event...
}
```

**Deduplication windows** limit how long you keep processed event IDs. You can't store every event ID forever, so you keep a window (e.g., 7 days) and assume events older than that won't be redelivered. This is a pragmatic tradeoff — unbounded deduplication tables grow forever.

**Transactional processing:**

Wrap the event processing and the idempotency record in a single database transaction:

```typescript
await db.transaction(async (tx) => {
  // Record that we've seen this event
  await tx.query(
    'INSERT INTO processed_events (event_id, processed_at) VALUES ($1, NOW()) ON CONFLICT DO NOTHING',
    [event.id]
  );

  // Do the actual work
  await tx.query(
    'INSERT INTO shipments (order_id, status) VALUES ($1, $2)',
    [event.orderId, 'pending']
  );
});
// Only ack the message AFTER the transaction commits
```

This guarantees atomicity: either both the work and the dedup record are written, or neither is. If the transaction fails, the message gets redelivered and reprocessed safely.

**Dead letter queues as recovery:**

When a message fails processing repeatedly (e.g., 3 retries), routing it to a DLQ prevents it from blocking the queue:

- **Without DLQ**: A poison message (malformed data, schema mismatch, bug in consumer) blocks the entire partition or queue. All subsequent messages wait.
- **With DLQ**: The failing message moves to a separate queue. Normal processing continues. Engineers can inspect DLQ messages, fix the root cause, and replay them.

DLQs are a recovery mechanism, not a solution. They buy you time. You still need to monitor DLQ depth, triage failures, and either fix and replay or discard messages.

</details>

<details>
<summary>8. What is the dual-write problem and how does the transactional outbox pattern solve it — why can't you reliably write to a database and publish an event in two separate operations, how does the outbox pattern make this atomic, and what are the approaches for reading from the outbox (polling, CDC)?</summary>

**The dual-write problem:**

When a service needs to update its database AND publish an event, doing both as separate operations is unreliable:

```typescript
// DANGEROUS: dual write
await db.query('UPDATE orders SET status = $1 WHERE id = $2', ['paid', orderId]);
await broker.publish('order.paid', { orderId }); // What if this fails?
```

If the database write succeeds but the event publish fails (network issue, broker down), the database says "paid" but no downstream service knows about it. If you reverse the order (publish first, then write), a failed DB write means downstream services process an event for a state change that never happened. There's no way to make two independent systems (database + broker) atomic without a coordination protocol.

**How the transactional outbox pattern solves it:**

Instead of publishing directly to the broker, write the event to an `outbox` table in the same database transaction as the domain change:

```typescript
await db.transaction(async (tx) => {
  // Domain change
  await tx.query('UPDATE orders SET status = $1 WHERE id = $2', ['paid', orderId]);

  // Outbox record — same transaction, guaranteed atomic
  await tx.query(
    `INSERT INTO outbox (id, aggregate_type, aggregate_id, event_type, payload, created_at)
     VALUES ($1, $2, $3, $4, $5, NOW())`,
    [uuid(), 'Order', orderId, 'OrderPaid', JSON.stringify({ orderId, amount: 99.99 })]
  );
});
```

Both writes are in one transaction — they either both commit or both roll back. No inconsistency possible.

**Reading from the outbox:**

A separate process reads the outbox table and publishes events to the broker:

**Polling approach:**

```typescript
// Runs on a schedule (e.g., every 100ms)
async function publishOutboxEvents() {
  const events = await db.query(
    'SELECT * FROM outbox WHERE published = false ORDER BY created_at LIMIT 100'
  );

  for (const event of events.rows) {
    await broker.publish(event.event_type, JSON.parse(event.payload));
    await db.query('UPDATE outbox SET published = true WHERE id = $1', [event.id]);
  }
}
```

- **Pros**: Simple to implement. No extra infrastructure.
- **Cons**: Polling interval introduces latency. Frequent polling wastes resources; infrequent polling delays events. Need to manage cleanup of old processed records.

**CDC (Change Data Capture) approach:**

Tools like Debezium read the database's transaction log (WAL in Postgres, binlog in MySQL) and publish changes to Kafka automatically.

- **Pros**: Near-real-time (milliseconds). No polling overhead. The outbox table doesn't need a `published` flag — CDC captures inserts as they happen.
- **Cons**: Requires additional infrastructure (Debezium, Kafka Connect). More complex to set up and operate. Tied to specific database change log formats.

**Recommendation**: Start with polling for simplicity. Move to CDC when latency requirements or volume make polling impractical.

</details>

<details>
<summary>9. How do sagas coordinate multi-step distributed workflows — what's the difference between choreography (each service reacts to events) and orchestration (central coordinator directs steps), why does choreography create hidden coupling as the workflow grows, how do compensating actions handle failures, and what makes saga failure handling harder than transaction rollback?</summary>

**Sagas** replace distributed transactions with a sequence of local transactions, each publishing events or commands that trigger the next step.

**Choreography vs orchestration:**

**Choreography** — each service listens to events and decides what to do:

```
OrderService emits OrderCreated
  → PaymentService hears it, charges customer, emits PaymentProcessed
    → InventoryService hears it, reserves stock, emits StockReserved
      → ShippingService hears it, creates shipment, emits ShipmentCreated
```

No central coordinator. Each service knows only about the events it consumes and produces.

**Orchestration** — a central saga orchestrator directs each step:

```
OrderSagaOrchestrator:
  1. Send ProcessPayment command → PaymentService
  2. On PaymentProcessed → Send ReserveStock command → InventoryService
  3. On StockReserved → Send CreateShipment command → ShippingService
  4. On ShipmentCreated → Mark saga complete
```

The workflow logic lives in one place. Individual services just handle commands.

**Why choreography creates hidden coupling:**

With 3-4 steps, choreography is elegant. At 8-10 steps, problems emerge:

- **No single view of the workflow**: To understand the order fulfillment flow, you have to trace event handlers across multiple services. No one file or service describes the complete workflow.
- **Implicit ordering**: The InventoryService must wait for PaymentProcessed — this dependency is implicit in "what events it subscribes to," not explicit in code.
- **Adding steps is risky**: Inserting a fraud check between payment and inventory means modifying multiple services and hoping the event chain still works.
- **Circular dependencies**: As workflows grow, services may start depending on each other's events in cycles.

**Compensating actions:**

When a step fails, you can't roll back previous steps (they're already committed in separate databases). Instead, you execute compensating actions — the business-logic inverse:

- Payment charged but inventory unavailable → issue a refund (not "undo payment" — a new forward action)
- Stock reserved but shipping failed → release the reservation

Compensating actions must be idempotent and may themselves fail, requiring their own retry logic.

**Why saga failure handling is harder than transaction rollback:**

- **Rollback is automatic and guaranteed**: In a database transaction, `ROLLBACK` undoes everything atomically. In a saga, you must explicitly code and trigger each compensation.
- **Partial failure states**: Between a failure and all compensations completing, the system is in an inconsistent state. The customer might see "payment charged" and "order failed" simultaneously.
- **Compensation can fail**: What if the refund API is down? Now you need retry logic for your error handling, and possibly a DLQ for failed compensations.
- **No isolation**: Other transactions can read intermediate states. In a database transaction, uncommitted changes are invisible to others.
- **Ordering matters**: Compensations must execute in reverse order of the original steps, and you need to track which steps completed to know what to compensate.

**Practical guidance**: Use choreography for simple, linear flows with 2-4 steps. Switch to orchestration when the workflow has branching, complex failure handling, or more than a handful of steps.

</details>

<details>
<summary>10. When should you NOT use event-driven architecture — what signals indicate a team is over-engineering with events when synchronous calls would suffice, what's the minimum scale or complexity that justifies the operational overhead, and what are the common failure modes of premature event-driven adoption?</summary>

**Signals you're over-engineering with events:**

- **Request-reply disguised as events**: If the producer publishes an event and then immediately polls or waits for a response event, you've built a synchronous call with extra steps and worse reliability. Just use HTTP or gRPC.
- **Single consumer**: If every event has exactly one consumer and you don't anticipate more, a direct service call is simpler and easier to debug.
- **Low latency requirement on the response**: Events introduce inherent latency. If the user needs an immediate, consistent response (payment confirmation, login), synchronous is the right model for the critical path.
- **Small team, few services**: 2-3 developers managing 2-3 services don't need Kafka. The operational overhead (broker management, consumer lag monitoring, DLQ triage, schema evolution) outweighs the decoupling benefits.
- **No independent scaling need**: If your services scale together and deploy together, you have a distributed monolith anyway — events don't help.

**Minimum complexity that justifies the overhead:**

There's no hard rule, but these conditions suggest event-driven adds real value:

- Multiple independent teams that need to react to the same business events without coordinating deployments.
- Significant read/write asymmetry where events feed multiple read models or projections.
- Traffic patterns where producers and consumers operate at different rates (burst writes, slow batch processing).
- Audit or replay requirements where events are the natural source of truth.
- Fan-out to 3+ independent consumers that genuinely evolve on different timelines.

**Common failure modes of premature adoption:**

- **"Event-driven" with a single Kafka topic acting as an RPC channel**: All the complexity, none of the decoupling. Services still tightly coupled, but now with eventual consistency and harder debugging.
- **No observability investment**: You adopt events but don't invest in distributed tracing, correlation IDs, or consumer lag dashboards. Debugging becomes guesswork.
- **Schema chaos**: No schema registry, no versioning strategy. Producers change event formats and consumers break silently.
- **Ignoring idempotency**: Assuming exactly-once delivery and building non-idempotent consumers. Works fine in testing, creates duplicate charges in production.
- **The "everything is an event" trap**: Internal method calls become events, simple database writes become event-sourced, CRUD endpoints become CQRS. The cognitive and operational cost explodes while the actual value stays flat.
- **No dead letter queue strategy**: Failed messages disappear or block the queue. Nobody notices until data inconsistencies surface days later.

**The test**: Can you clearly articulate what problem events solve that a simpler approach doesn't? If the answer is "it's more modern" or "we might need it later," synchronous calls are the right choice today.

</details>

<details>
<summary>11. How does event ordering work at the partition level, and when does ordering actually matter vs when can you safely ignore it — how do you handle out-of-order events when they arrive (version-based conflict resolution, last-write-wins, causal ordering), and what design decisions determine whether your system needs strict ordering or can tolerate reordering?</summary>

**Partition-level ordering:**

Kafka (and similar systems) guarantees ordering only within a single partition. Events with the same partition key land in the same partition and are consumed in order. Events across different partitions have no ordering guarantee.

If you use `orderId` as the partition key, all events for order #123 (`OrderCreated`, `PaymentProcessed`, `OrderShipped`) arrive in order. But events for order #123 and order #456 may interleave arbitrarily.

**When ordering matters:**

- **State machine transitions**: An order must go `created -> paid -> shipped`. Processing `shipped` before `paid` corrupts state.
- **Aggregate-level consistency**: Events that modify the same entity must be processed in sequence. Two `InventoryAdjusted` events for the same SKU applied out of order produce wrong stock counts.
- **Financial ledgers**: Debits and credits for an account must be ordered to maintain a correct running balance.

**When you can safely ignore ordering:**

- **Independent entities**: Events for different orders, different users, different products. No shared state, so ordering between them is irrelevant.
- **Additive/commutative operations**: Incrementing a counter, adding items to a set, or appending to a log. The result is the same regardless of order.
- **Notifications and analytics**: Sending an email 2 seconds before or after another email doesn't matter. Analytics events can be reordered without affecting aggregated results.

**Handling out-of-order events:**

**Version-based conflict resolution:**

Each event carries a version number. The consumer only applies events that are exactly version N+1:

```typescript
async function applyEvent(event: DomainEvent) {
  const current = await db.query(
    'SELECT version FROM aggregates WHERE id = $1', [event.aggregateId]
  );

  if (event.version !== current.rows[0].version + 1) {
    // Out of order — buffer or reject
    await bufferEvent(event);
    return;
  }

  await db.query(
    'UPDATE aggregates SET state = $1, version = $2 WHERE id = $3',
    [applyChange(current.rows[0].state, event), event.version, event.aggregateId]
  );
}
```

- Strict but complex. Requires buffering out-of-order events and re-processing when gaps are filled.

**Last-write-wins (LWW):**

Each event has a timestamp. The consumer always applies the event with the latest timestamp, ignoring older ones:

```typescript
if (event.timestamp > currentState.lastUpdated) {
  await updateState(event);
}
```

- Simple but lossy. Concurrent updates to different fields of the same entity can silently drop changes.

**Causal ordering:**

Use vector clocks or Lamport timestamps to establish happens-before relationships. Only enforce ordering between causally related events. Unrelated events can be processed in any order.

- Most flexible but most complex to implement. Rarely needed outside of distributed databases.

**Design decisions that determine your ordering needs:**

- **Partition key strategy**: Choosing the right key (entity ID, tenant ID, user ID) determines what's ordered relative to what. Wrong key = either too much ordering (killing parallelism) or too little (corrupting state).
- **Consumer design**: Can your consumer handle gaps? Does it buffer, reject, or apply out-of-order events with conflict resolution?
- **Business domain**: Does the domain have true sequential dependencies, or are the events commutative? If commutative, relax ordering and gain parallelism.
- **Number of partitions vs consumers**: More partitions means more parallelism but ordering is only within a partition. If you later need to re-key (change partition key), it requires a data migration.

</details>

## Practical — Event Patterns & Implementation

<details>
<summary>12. Implement an idempotent event consumer in Node.js — show the consumer that processes an order-created event, uses an idempotency key to prevent duplicate processing, wraps the work in a database transaction, and routes failures to a dead letter queue. Explain what happens when the same event is delivered twice</summary>

```typescript
import { Pool } from 'pg';
import { Consumer, Producer, EachMessagePayload } from 'kafkajs';

interface OrderCreatedEvent {
  eventId: string;
  orderId: string;
  customerId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  total: number;
  occurredAt: string;
}

const MAX_RETRIES = 3;

async function handleOrderCreated(
  pool: Pool,
  event: OrderCreatedEvent,
  dlqProducer: Producer
): Promise<void> {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // Idempotency check + record in one atomic INSERT
    const { rowCount } = await client.query(
      `INSERT INTO processed_events (event_id, event_type, processed_at)
       VALUES ($1, 'OrderCreated', NOW())
       ON CONFLICT (event_id) DO NOTHING`,
      [event.eventId]
    );

    if (rowCount === 0) {
      // Already processed — skip silently
      await client.query('ROLLBACK');
      return;
    }

    // Business logic: create shipment preparation record
    await client.query(
      `INSERT INTO shipment_preparations (order_id, customer_id, total, status, created_at)
       VALUES ($1, $2, $3, 'pending', NOW())`,
      [event.orderId, event.customerId, event.total]
    );

    // Insert line items
    for (const item of event.items) {
      await client.query(
        `INSERT INTO shipment_items (order_id, product_id, quantity)
         VALUES ($1, $2, $3)`,
        [event.orderId, item.productId, item.quantity]
      );
    }

    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error; // Let the outer handler decide retry vs DLQ
  } finally {
    client.release();
  }
}

// Track retry attempts in-process (resets if consumer restarts, which is acceptable
// because a restart effectively gives the message a fresh chance)
const retryCounts = new Map<string, number>();

// Consumer setup with retry + DLQ routing
async function startConsumer(
  consumer: Consumer,
  pool: Pool,
  dlqProducer: Producer
): Promise<void> {
  await consumer.subscribe({ topic: 'order.created', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }: EachMessagePayload) => {
      const event: OrderCreatedEvent = JSON.parse(message.value!.toString());
      const messageKey = `${topic}-${partition}-${message.offset}`;
      const retryCount = retryCounts.get(messageKey) ?? 0;

      try {
        await handleOrderCreated(pool, event, dlqProducer);
        retryCounts.delete(messageKey); // clean up on success
      } catch (error) {
        if (retryCount >= MAX_RETRIES) {
          // Route to DLQ after max retries exhausted
          await dlqProducer.send({
            topic: 'order.created.dlq',
            messages: [{
              key: message.key,
              value: message.value,
              headers: {
                ...message.headers,
                'original-topic': topic,
                'original-partition': String(partition),
                'failure-reason': (error as Error).message,
                'failed-at': new Date().toISOString(),
              },
            }],
          });
          retryCounts.delete(messageKey); // clean up after DLQ routing
        } else {
          retryCounts.set(messageKey, retryCount + 1);
          // Re-throw to trigger KafkaJS retry
          throw error;
        }
      }
    },
  });
}
```

**What happens when the same event is delivered twice:**

1. First delivery: The `INSERT INTO processed_events ... ON CONFLICT DO NOTHING` succeeds with `rowCount = 1`. The business logic executes, the transaction commits, and the message is acknowledged.
2. Second delivery: The same `INSERT` hits the `ON CONFLICT` clause, returning `rowCount = 0`. The function returns immediately without executing any business logic. The message is acknowledged and discarded.

The key insight is that the idempotency check and the business logic are in the same transaction. There's no window where the dedup record is written but the business logic isn't (or vice versa). If the transaction fails at any point, both the dedup record and the business writes roll back together, so the redelivered message can safely retry from scratch.

</details>

<details>
<summary>13. Implement the transactional outbox pattern — show the code that writes a domain change and an outbox record in a single database transaction, then a separate process that reads the outbox and publishes events to the broker. Compare the polling approach vs CDC (Change Data Capture) for reading the outbox</summary>

The conceptual explanation of the outbox pattern was covered in question 8. Here's the full implementation.

**Outbox table schema:**

```sql
CREATE TABLE outbox (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type VARCHAR(100) NOT NULL,
  aggregate_id VARCHAR(100) NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at TIMESTAMPTZ  -- NULL until published
);

CREATE INDEX idx_outbox_unpublished ON outbox (created_at) WHERE published_at IS NULL;
```

**Write side — atomic domain change + outbox record:**

```typescript
import { Pool } from 'pg';

interface CreateOrderParams {
  orderId: string;
  customerId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  total: number;
}

async function createOrder(pool: Pool, params: CreateOrderParams): Promise<void> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // Domain write
    await client.query(
      `INSERT INTO orders (id, customer_id, total, status, created_at)
       VALUES ($1, $2, $3, 'created', NOW())`,
      [params.orderId, params.customerId, params.total]
    );

    for (const item of params.items) {
      await client.query(
        `INSERT INTO order_items (order_id, product_id, quantity, price)
         VALUES ($1, $2, $3, $4)`,
        [params.orderId, item.productId, item.quantity, item.price]
      );
    }

    // Outbox write — same transaction
    await client.query(
      `INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
       VALUES ('Order', $1, 'OrderCreated', $2)`,
      [params.orderId, JSON.stringify({
        orderId: params.orderId,
        customerId: params.customerId,
        items: params.items,
        total: params.total,
      })]
    );

    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

**Polling relay — reads outbox and publishes to broker:**

```typescript
import { Producer } from 'kafkajs';

async function pollOutbox(pool: Pool, producer: Producer): Promise<void> {
  const client = await pool.connect();
  try {
    // Use SELECT FOR UPDATE SKIP LOCKED to allow multiple relay instances
    // without processing the same row twice
    await client.query('BEGIN');

    const { rows } = await client.query(
      `SELECT id, event_type, aggregate_id, payload
       FROM outbox
       WHERE published_at IS NULL
       ORDER BY created_at
       LIMIT 50
       FOR UPDATE SKIP LOCKED`
    );

    if (rows.length === 0) {
      await client.query('ROLLBACK');
      return;
    }

    for (const row of rows) {
      await producer.send({
        topic: row.event_type.replace(/([A-Z])/g, '.$1').slice(1).toLowerCase(),
        messages: [{
          key: row.aggregate_id,
          value: JSON.stringify(row.payload),
          headers: { 'outbox-id': row.id },
        }],
      });

      await client.query(
        'UPDATE outbox SET published_at = NOW() WHERE id = $1',
        [row.id]
      );
    }

    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// Run polling on interval
setInterval(() => pollOutbox(pool, producer), 200);
```

**Polling vs CDC comparison:**

| Aspect | Polling | CDC (Debezium) |
|--------|---------|----------------|
| **Latency** | 100-500ms (polling interval) | ~10-50ms (reads WAL in near-real-time) |
| **Infrastructure** | None beyond the app itself | Debezium + Kafka Connect + connector config |
| **Complexity** | Simple — a loop with a query | Significant — WAL configuration, connector management, schema mapping |
| **Scaling** | `FOR UPDATE SKIP LOCKED` allows multiple pollers | Debezium runs as a single connector per table (HA via Kafka Connect) |
| **Cleanup** | You manage it — delete old published rows on a schedule | Can delete rows immediately after insert since CDC captures from the WAL |
| **Failure handling** | If publish fails, row stays unpublished, next poll retries | If Debezium falls behind, it resumes from its last WAL position |

**When to use which:**

- **Polling**: Start here. Works well up to thousands of events per second. Simple to operate, debug, and reason about.
- **CDC**: When you need sub-100ms latency from write to event delivery, or when event volume makes polling expensive (tens of thousands per second). The infrastructure cost is justified at that scale.

</details>

<details>
<summary>14. Implement a saga using choreography for an order fulfillment workflow (order → payment → inventory → shipping) — show the events each service publishes and consumes, implement a compensating action (refund payment when inventory fails), and demonstrate the failure flow when a middle step fails</summary>

The conceptual explanation of choreography vs orchestration and compensating actions was covered in question 9. Here's the full implementation.

**Shared event types:**

```typescript
// events.ts — shared across services
interface OrderCreated {
  type: 'OrderCreated';
  orderId: string;
  customerId: string;
  items: Array<{ productId: string; quantity: number }>;
  total: number;
  correlationId: string;
}

interface PaymentProcessed {
  type: 'PaymentProcessed';
  orderId: string;
  paymentId: string;
  amount: number;
  correlationId: string;
}

interface PaymentFailed {
  type: 'PaymentFailed';
  orderId: string;
  reason: string;
  correlationId: string;
}

interface StockReserved {
  type: 'StockReserved';
  orderId: string;
  reservationId: string;
  correlationId: string;
}

interface StockReservationFailed {
  type: 'StockReservationFailed';
  orderId: string;
  reason: string;
  correlationId: string;
}

interface PaymentRefunded {
  type: 'PaymentRefunded';
  orderId: string;
  paymentId: string;
  correlationId: string;
}

interface ShipmentCreated {
  type: 'ShipmentCreated';
  orderId: string;
  trackingNumber: string;
  correlationId: string;
}
```

**Order Service — starts the saga:**

```typescript
// Listens to: nothing (initiator)
// Publishes: OrderCreated
// Also listens to: PaymentFailed, StockReservationFailed (to update order status)
async function createOrder(params: CreateOrderParams): Promise<void> {
  const correlationId = crypto.randomUUID();

  await db.transaction(async (tx) => {
    await tx.query(
      `INSERT INTO orders (id, customer_id, total, status) VALUES ($1, $2, $3, 'pending')`,
      [params.orderId, params.customerId, params.total]
    );

    await insertOutbox(tx, {
      type: 'OrderCreated',
      orderId: params.orderId,
      customerId: params.customerId,
      items: params.items,
      total: params.total,
      correlationId,
    });
  });
}

// Compensation listener — mark order as failed
async function handleSagaFailure(event: PaymentFailed | StockReservationFailed) {
  await db.query(
    `UPDATE orders SET status = 'failed', failure_reason = $1 WHERE id = $2`,
    [event.reason, event.orderId]
  );
}
```

**Payment Service — charges customer:**

```typescript
// Listens to: OrderCreated
// Publishes: PaymentProcessed | PaymentFailed
// Also listens to: StockReservationFailed (to trigger refund)
async function handleOrderCreated(event: OrderCreated): Promise<void> {
  try {
    const paymentId = await chargeCustomer(event.customerId, event.total);

    await db.transaction(async (tx) => {
      await tx.query(
        `INSERT INTO payments (id, order_id, amount, status) VALUES ($1, $2, $3, 'charged')`,
        [paymentId, event.orderId, event.total]
      );

      await insertOutbox(tx, {
        type: 'PaymentProcessed',
        orderId: event.orderId,
        paymentId,
        amount: event.total,
        correlationId: event.correlationId,
      });
    });
  } catch (error) {
    await publishEvent({
      type: 'PaymentFailed',
      orderId: event.orderId,
      reason: (error as Error).message,
      correlationId: event.correlationId,
    });
  }
}

// Compensating action: refund when inventory fails
async function handleStockReservationFailed(event: StockReservationFailed): Promise<void> {
  const payment = await db.query(
    'SELECT id, amount FROM payments WHERE order_id = $1 AND status = $2',
    [event.orderId, 'charged']
  );

  if (payment.rows.length === 0) return; // nothing to refund (idempotent)

  const { id: paymentId, amount } = payment.rows[0];

  await refundCustomer(paymentId, amount); // call payment gateway

  await db.transaction(async (tx) => {
    await tx.query(
      `UPDATE payments SET status = 'refunded' WHERE id = $1`,
      [paymentId]
    );

    await insertOutbox(tx, {
      type: 'PaymentRefunded',
      orderId: event.orderId,
      paymentId,
      correlationId: event.correlationId,
    });
  });
}
```

**Inventory Service — reserves stock:**

```typescript
// Listens to: PaymentProcessed
// Publishes: StockReserved | StockReservationFailed
async function handlePaymentProcessed(event: PaymentProcessed): Promise<void> {
  try {
    // Attempt to reserve all items
    const order = await getOrderItems(event.orderId);
    const reservationId = crypto.randomUUID();

    await db.transaction(async (tx) => {
      for (const item of order.items) {
        const result = await tx.query(
          `UPDATE inventory SET reserved = reserved + $1
           WHERE product_id = $2 AND (quantity - reserved) >= $1`,
          [item.quantity, item.productId]
        );
        if (result.rowCount === 0) {
          throw new Error(`Insufficient stock for ${item.productId}`);
        }
      }

      await insertOutbox(tx, {
        type: 'StockReserved',
        orderId: event.orderId,
        reservationId,
        correlationId: event.correlationId,
      });
    });
  } catch (error) {
    await publishEvent({
      type: 'StockReservationFailed',
      orderId: event.orderId,
      reason: (error as Error).message,
      correlationId: event.correlationId,
    });
  }
}
```

**Failure flow when inventory fails:**

```
1. OrderService publishes OrderCreated
2. PaymentService charges customer, publishes PaymentProcessed
3. InventoryService tries to reserve stock — FAILS (out of stock)
4. InventoryService publishes StockReservationFailed
5. PaymentService hears StockReservationFailed → refunds customer, publishes PaymentRefunded
6. OrderService hears StockReservationFailed → updates order status to 'failed'
```

**Key observations:**

- Every handler is idempotent — checking current state before acting (e.g., "is the payment still in 'charged' status?").
- The `correlationId` flows through every event, enabling distributed tracing across the entire saga.
- Compensation is a forward action (refund), not a rollback. Between steps 3 and 5, the customer has been charged but the order is failing — this transient inconsistency is inherent to sagas.
- If the refund in step 5 fails, it needs its own retry mechanism. This is why saga failure handling is harder than transaction rollback (as discussed in question 9).

</details>

<details>
<summary>15. Implement CQRS with a read model projection — show a write side that publishes events on state changes, a projection that listens to events and builds a denormalized read model, and the query handler that reads from the projection. Demonstrate the eventual consistency gap and how to handle it from the client's perspective</summary>

Building on the CQRS concepts from question 6, here's a complete implementation.

**Write side — processes commands, emits events via outbox:**

The write side uses the same transactional outbox pattern from Q13 — domain write + outbox record in one transaction. Here the outbox record carries an `OrderPlaced` event that the projection will consume:

```typescript
// write-side/order-commands.ts
interface PlaceOrderCommand {
  orderId: string;
  customerId: string;
  items: Array<{ productId: string; name: string; quantity: number; price: number }>;
}

async function placeOrder(pool: Pool, cmd: PlaceOrderCommand): Promise<void> {
  const total = cmd.items.reduce((sum, i) => sum + i.price * i.quantity, 0);

  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    await client.query(
      `INSERT INTO orders (id, customer_id, status, total, created_at)
       VALUES ($1, $2, 'placed', $3, NOW())`,
      [cmd.orderId, cmd.customerId, total]
    );

    for (const item of cmd.items) {
      await client.query(
        `INSERT INTO order_items (order_id, product_id, name, quantity, price)
         VALUES ($1, $2, $3, $4, $5)`,
        [cmd.orderId, item.productId, item.name, item.quantity, item.price]
      );
    }

    // Outbox — same pattern as Q13's createOrder
    await client.query(
      `INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
       VALUES ('Order', $1, 'OrderPlaced', $2)`,
      [cmd.orderId, JSON.stringify({
        orderId: cmd.orderId,
        customerId: cmd.customerId,
        items: cmd.items,
        total,
        placedAt: new Date().toISOString(),
      })]
    );

    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
// Status updates follow the same pattern — domain UPDATE + outbox INSERT in one transaction (see Q13).
```

**Read model schema — denormalized for fast queries:**

```sql
-- Denormalized: one table with everything the UI needs, no joins
CREATE TABLE order_read_model (
  order_id VARCHAR(100) PRIMARY KEY,
  customer_id VARCHAR(100) NOT NULL,
  status VARCHAR(50) NOT NULL,
  total DECIMAL(10, 2) NOT NULL,
  item_count INTEGER NOT NULL,
  items JSONB NOT NULL,           -- pre-serialized for the API response
  placed_at TIMESTAMPTZ NOT NULL,
  last_updated TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_order_read_customer ON order_read_model (customer_id, placed_at DESC);
```

**Projection — listens to events and updates read model:**

```typescript
// read-side/order-projection.ts
async function handleOrderPlaced(event: OrderPlacedEvent): Promise<void> {
  await readDb.query(
    `INSERT INTO order_read_model
     (order_id, customer_id, status, total, item_count, items, placed_at, last_updated)
     VALUES ($1, $2, 'placed', $3, $4, $5, $6, NOW())
     ON CONFLICT (order_id) DO NOTHING`,  -- idempotent
    [
      event.orderId,
      event.customerId,
      event.total,
      event.items.length,
      JSON.stringify(event.items),
      event.placedAt,
    ]
  );
}

async function handleOrderStatusChanged(event: OrderStatusChangedEvent): Promise<void> {
  await readDb.query(
    `UPDATE order_read_model
     SET status = $1, last_updated = $2
     WHERE order_id = $3 AND last_updated < $2`,  -- idempotent: only apply if newer
    [event.status, event.changedAt, event.orderId]
  );
}
```

**Query handler — reads from projection:**

```typescript
// read-side/order-queries.ts
async function getCustomerOrders(customerId: string): Promise<OrderView[]> {
  const { rows } = await readDb.query(
    `SELECT order_id, status, total, item_count, items, placed_at
     FROM order_read_model
     WHERE customer_id = $1
     ORDER BY placed_at DESC
     LIMIT 50`,
    [customerId]
  );
  return rows;
}

async function getOrder(orderId: string): Promise<OrderView | null> {
  const { rows } = await readDb.query(
    'SELECT * FROM order_read_model WHERE order_id = $1',
    [orderId]
  );
  return rows[0] ?? null;
}
```

**Handling the eventual consistency gap from the client:**

The problem: A user places an order, the API returns success, the client redirects to the order list page, but the projection hasn't processed the event yet — the new order is missing.

**Approach 1 — Return the write-side data in the command response:**

```typescript
// API endpoint
app.post('/orders', async (req, res) => {
  const orderId = crypto.randomUUID();
  await placeOrder(pool, { orderId, ...req.body });

  // Return the data the client needs immediately — don't force a read-model query
  res.status(201).json({
    orderId,
    status: 'placed',
    total: req.body.items.reduce((s, i) => s + i.price * i.quantity, 0),
    _consistency: 'strong', // signal to the client
  });
});
```

**Approach 2 — Client-side optimistic update:**

```typescript
// React client
const { mutate } = usePlaceOrder();
const queryClient = useQueryClient();

async function handlePlaceOrder(data: OrderFormData) {
  const result = await mutate(data);

  // Optimistically add to the cached order list before the projection catches up
  queryClient.setQueryData(['orders'], (old: OrderView[]) => [
    { orderId: result.orderId, status: 'placed', total: result.total, placedAt: new Date() },
    ...old,
  ]);
}
```

**Approach 3 — Polling with timeout (for critical reads):**

```typescript
async function waitForProjection(orderId: string, maxWaitMs = 3000): Promise<OrderView | null> {
  const start = Date.now();
  while (Date.now() - start < maxWaitMs) {
    const order = await getOrder(orderId);
    if (order) return order;
    await new Promise((r) => setTimeout(r, 100));
  }
  return null; // projection didn't catch up in time — fall back to write-side data
}
```

In practice, approach 1 (return data from the command response) combined with approach 2 (optimistic UI) handles most cases cleanly without added latency.

</details>

<details>
<summary>16. Implement event schema evolution for an event-sourced system — show how to version event types, handle backward-compatible changes (adding optional fields) vs breaking changes (renaming fields), and demonstrate how a consumer handles both v1 and v2 of an event during the migration period</summary>

Building on the schema evolution concepts from question 4, here's the implementation.

**Versioned event type definitions:**

```typescript
// events/order-created.ts

// V1 — original schema
interface OrderCreatedV1 {
  type: 'OrderCreated';
  version: 1;
  orderId: string;
  customerId: string;
  amount: number;       // no currency — assumed USD
  items: Array<{ productId: string; quantity: number }>;
  timestamp: string;
}

// V2 — backward-compatible: added optional currency field
interface OrderCreatedV2 {
  type: 'OrderCreated';
  version: 2;
  orderId: string;
  customerId: string;
  amount: number;
  currency: string;     // NEW: explicit currency
  items: Array<{ productId: string; quantity: number; unitPrice: number }>; // NEW: unitPrice
  timestamp: string;
}

// V3 — breaking change: renamed customerId to buyerId, removed amount (derived from items)
interface OrderCreatedV3 {
  type: 'OrderCreated';
  version: 3;
  orderId: string;
  buyerId: string;      // RENAMED from customerId
  currency: string;
  items: Array<{ productId: string; quantity: number; unitPrice: number }>;
  timestamp: string;
}

// The "current" shape your application code works with
type CurrentOrderCreated = OrderCreatedV3;
```

**Upcasters — transform old versions to the current schema:**

```typescript
// upcasters/order-created.ts

type AnyOrderCreated = OrderCreatedV1 | OrderCreatedV2 | OrderCreatedV3;

function upcastOrderCreated(event: AnyOrderCreated): CurrentOrderCreated {
  let current = event as any;

  // V1 → V2: add currency, add unitPrice to items
  if (current.version === 1) {
    current = {
      ...current,
      version: 2,
      currency: 'USD', // default for legacy events
      items: current.items.map((item: any) => ({
        ...item,
        unitPrice: 0, // unknown — flag for manual review if needed
      })),
    };
  }

  // V2 → V3: rename customerId to buyerId, drop amount
  if (current.version === 2) {
    const { customerId, amount, ...rest } = current;
    current = {
      ...rest,
      version: 3,
      buyerId: customerId,
    };
  }

  return current as CurrentOrderCreated;
}
```

**Event store reader with upcasting:**

```typescript
// event-store/reader.ts

interface StoredEvent {
  stream_id: string;
  event_type: string;
  version: number;
  payload: Record<string, unknown>;
  sequence: number;
}

const upcasterRegistry: Record<string, (event: any) => any> = {
  OrderCreated: upcastOrderCreated,
  // Register upcasters for other event types...
};

async function loadEvents(streamId: string): Promise<CurrentEvent[]> {
  const { rows } = await db.query<StoredEvent>(
    'SELECT * FROM events WHERE stream_id = $1 ORDER BY sequence ASC',
    [streamId]
  );

  return rows.map((row) => {
    const raw = { type: row.event_type, version: row.version, ...row.payload };
    const upcaster = upcasterRegistry[row.event_type];
    return upcaster ? upcaster(raw) : raw;
  });
}
```

**Consumer handling both V1 and V2 during migration period:**

During a rolling deployment, some producers are still emitting V1 while others emit V2. The consumer must handle both:

```typescript
// consumer/order-created-handler.ts

async function handleOrderCreatedMessage(rawMessage: string): Promise<void> {
  const parsed = JSON.parse(rawMessage);
  const version = parsed.version ?? 1; // V1 events might not have a version field

  // Upcast to current version regardless of what arrives
  const event = upcastOrderCreated({ ...parsed, version } as AnyOrderCreated);

  // Business logic always works with the current schema
  await db.query(
    `INSERT INTO order_projections (order_id, buyer_id, currency, total, created_at)
     VALUES ($1, $2, $3, $4, $5)
     ON CONFLICT (order_id) DO NOTHING`,
    [
      event.orderId,
      event.buyerId,
      event.currency,
      event.items.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0),
      event.timestamp,
    ]
  );
}
```

**Guidelines for schema evolution:**

| Change type | Backward compatible? | Strategy |
|------------|---------------------|----------|
| Add optional field | Yes | Consumers ignore unknown fields. Add a default in the upcaster for old events. |
| Add required field | No | Make it optional first, backfill, then make it required in a later version. |
| Remove field | Depends | Safe if no consumer uses it. Upcaster can drop it. |
| Rename field | No | Upcaster maps old name to new name. Deploy consumer changes before producer changes. |
| Change field type | No | Upcaster converts (e.g., string to number). Requires a new version. |

**Deployment order for breaking changes:**

1. Deploy consumers that can handle both V1 and V2 (upcasting V1 to V2).
2. Deploy producers that emit V2.
3. After all V1 producers are gone and old events are past your replay window, optionally remove V1 handling.

For event-sourced systems where old events live forever, you can never remove upcasters — V1 events are permanent. The upcaster chain (V1 to V2 to V3 to ...) grows over time. This is the real cost of event sourcing schema evolution.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>17. Tell me about a time you introduced event-driven architecture to a system — what problem were you solving, what did you choose (event sourcing, CQRS, simple pub/sub), and what unexpected challenges did you encounter?</summary>

**What the interviewer is looking for:**

- Ability to articulate a clear problem that justified the complexity of event-driven architecture (not just "it's modern").
- Thoughtful technology and pattern selection — why you chose what you chose.
- Honest reflection on challenges — shows real experience, not textbook knowledge.
- Evidence of weighing tradeoffs rather than blindly adopting patterns.

**Suggested structure (STAR-ish):**

1. **Context**: What system were you working on? What was the scale?
2. **Problem**: What specific pain point drove the change? (tight coupling, scaling bottleneck, need for audit trail, etc.)
3. **Decision**: What pattern did you choose and why? What alternatives did you consider?
4. **Implementation**: Key design decisions — event schema, broker choice, idempotency approach.
5. **Unexpected challenges**: What surprised you? (debugging difficulty, eventual consistency confusion, schema evolution pain, operational overhead)
6. **Outcome**: Did it solve the problem? What would you do differently?

**Example outline to personalize:**

> "We had an order processing system where adding new post-order workflows (notifications, analytics, fraud scoring) required deploying the order service every time. We introduced GCP Pub/Sub as a simple pub/sub layer — the order service published `OrderPlaced` events and each downstream service subscribed independently.
>
> The unexpected challenge was debugging. When a customer reported missing notifications, tracing the issue across three async services with no shared request ID was painful. We didn't invest in correlation IDs or distributed tracing upfront, and it cost us. We retrofitted OpenTelemetry trace propagation through message headers, which dramatically improved our ability to diagnose issues.
>
> Looking back, the decoupling was absolutely worth it — we went from coordinated deploys to independent team velocity. But I'd front-load the observability investment next time instead of treating it as an afterthought."

**Key points to hit:**

- Name the specific problem events solved (not generic benefits).
- Show awareness of what you gave up (consistency, debuggability).
- Demonstrate learning from the experience.

</details>

<details>
<summary>18. Describe a time you debugged a complex issue in an event-driven system — how did you trace the problem across services, what tools did you use, and what observability improvements did you add afterward?</summary>

**What the interviewer is looking for:**

- Systematic debugging approach — not "I looked at logs until I found it."
- Familiarity with distributed tracing tools and techniques.
- Proactive improvement after the incident — closing the observability gap.
- Understanding of why event-driven systems are inherently harder to debug.

**Suggested structure:**

1. **The symptom**: What was the user-visible problem? (data inconsistency, missing records, duplicates, delayed processing)
2. **Initial investigation**: Where did you start? What did you check first?
3. **The tracing challenge**: Why was this hard to debug in an event-driven context? (no single request trace, events processed at different times, no obvious causal chain)
4. **How you found it**: What tool or technique led you to the root cause? (correlation IDs, consumer lag metrics, DLQ inspection, log aggregation with structured fields)
5. **Root cause**: What was actually wrong? (out-of-order processing, missing idempotency, schema mismatch, consumer rebalancing, visibility timeout too short)
6. **Improvements added**: What observability or reliability changes did you make afterward?

**Example outline to personalize:**

> "We had a bug where some orders showed 'payment processed' but their inventory was never reserved. The issue only affected ~2% of orders, making it hard to reproduce.
>
> I started with our log aggregation (Cloud Logging), filtering by affected order IDs. I could see the `PaymentProcessed` event being emitted but no corresponding `StockReserved` or `StockReservationFailed` event from the inventory service. The inventory consumer logs showed it processing the event — but for a different order.
>
> The root cause was a partition key mismatch. The payment service was publishing to a topic partitioned by `customerId`, but the inventory consumer expected ordering by `orderId`. When a customer had concurrent orders, events could arrive interleaved, and the inventory service's version check rejected the 'out of order' event.
>
> Afterward, we added: (1) a correlation ID flowing through all saga events, (2) a dashboard tracking saga completion rates with alerts on anomalies, and (3) a reconciliation job that detected orders stuck in intermediate states."

**Key points to hit:**

- Show a methodical debugging process, not trial and error.
- Name specific tools (OpenTelemetry, Jaeger, Cloud Logging, Prometheus, Grafana).
- The improvement should be systemic (better observability), not just a point fix.

</details>

<details>
<summary>19. Tell me about a time you dealt with message ordering, duplication, or data consistency issues in an async system — what was the problem, how did you solve it, and what patterns did you adopt to prevent recurrence?</summary>

**What the interviewer is looking for:**

- Deep understanding of at-least-once delivery implications in practice.
- Familiarity with patterns like idempotency keys, version-based ordering, and deduplication.
- Evidence of designing for reliability, not just fixing bugs.
- Ability to explain the tradeoff between correctness and performance.

**Suggested structure:**

1. **The problem**: What went wrong? (duplicate charges, missing data, corrupted state from out-of-order processing)
2. **Impact**: How did it affect users or business? Was it silent or user-facing?
3. **Root cause analysis**: What caused the ordering/duplication/consistency issue? (consumer crash before ack, rebalancing, visibility timeout, no idempotency check, wrong partition key)
4. **The fix**: What specific pattern or code change resolved it?
5. **Prevention**: What systemic patterns did you adopt to prevent the same class of bug across other consumers?

**Example outline to personalize:**

> "Our notification service was sending duplicate emails for the same order event. A customer received 3 'Your order has shipped' emails within a minute.
>
> Root cause: The SQS visibility timeout was set to 30 seconds, but our email-sending consumer sometimes took 45 seconds (slow SMTP provider). SQS would redeliver the message while the first processing was still in progress. Since we had no idempotency check, each delivery triggered a new email.
>
> Immediate fix: Added an idempotency table — before sending, check if this event ID was already processed and record it in the same transaction as marking the notification as sent.
>
> Prevention: We adopted a team-wide standard — every consumer must have an idempotency check, and we added a shared library that wraps consumer handlers with automatic dedup. We also tuned visibility timeouts to 3x the p99 processing time and added metrics to flag when processing time approaches the timeout."

**Key points to hit:**

- Be specific about the root cause — "duplicate delivery" is a symptom, not a cause.
- Show the fix addresses the root cause and the class of bug, not just the symptom.
- Mention any standards or patterns you established for the team.

</details>

<details>
<summary>20. Describe a time you had to choose between choreography and orchestration for a distributed workflow — what were the requirements, what did you choose, and looking back, was it the right call?</summary>

**What the interviewer is looking for:**

- Ability to evaluate architectural patterns against concrete requirements, not abstract preference.
- Understanding of the tradeoffs covered in question 9 (hidden coupling, visibility, complexity).
- Honest retrospective — what worked, what didn't, what you'd change.
- Awareness of how the choice affected team productivity and operational burden.

**Suggested structure:**

1. **The workflow**: What was the business process? How many steps? How many services involved?
2. **Requirements that influenced the choice**: Was visibility important? Complex failure handling? Independent team ownership? Low latency?
3. **What you chose and why**: Which pattern, and what was the deciding factor?
4. **How it played out**: Did the choice hold up as the workflow evolved? What was easy? What was painful?
5. **Retrospective**: Was it the right call? What would you choose today with the benefit of hindsight?

**Example outline to personalize:**

> "We had an order fulfillment workflow: order creation, payment processing, inventory reservation, and shipping label generation. Four steps, four services, two teams.
>
> We chose choreography because the flow was linear and each team wanted full ownership of their service. No team wanted to own a central orchestrator. Each service published events and reacted to upstream events independently.
>
> It worked well for the first 6 months. Then product asked for a 'fraud check' step between payment and inventory, plus conditional branching — high-value orders needed manual review before shipping. Adding these to choreography was painful: the fraud service needed to intercept `PaymentProcessed` before inventory saw it, and conditional logic was spread across three services.
>
> Looking back, choreography was the right call initially — it let both teams move fast with a simple flow. But I'd plan for the migration to orchestration earlier. The signals were there: product was already talking about 'workflow rules' and 'approval gates' in the roadmap. Today I'd start with choreography for the MVP but extract an orchestrator as soon as the workflow gets conditional or exceeds 4-5 steps."

**Key points to hit:**

- Show the decision was based on concrete requirements, not dogma.
- Demonstrate understanding of when each pattern breaks down.
- The retrospective should show growth — not "I was wrong" or "I was right," but "here's what I learned about when to switch."

</details>
