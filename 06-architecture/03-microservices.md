# Microservices

> **22 questions** — 14 theory, 5 practical, 3 experience

- Monolith vs microservices: honest tradeoffs and "monolith first"
- Conway's Law and how team structure shapes service decomposition
- Service boundaries using DDD: bounded contexts, aggregates, domain events
- Distributed monolith anti-pattern: signs of bad decomposition (lock-step deploys, chatty cross-service calls, shared state), how to detect and refactor
- Synchronous vs asynchronous communication: REST/gRPC vs events/messages
- Sagas: choreography vs orchestration, compensating transactions, and retry safety
- Resilience patterns: circuit breakers, retries with backoff, bulkheads, timeout budgets -- composing them to prevent cascading failures
- Data consistency across services: eventual consistency tradeoffs, idempotency keys, read-your-own-writes, dual-write problems, transactional outbox pattern
- API gateways: routing, auth, rate limiting, BFF pattern, and single-point-of-failure risks
- Observability across services: distributed tracing, correlation IDs, and structured logging for cross-service debugging
- Database-per-service: why shared databases couple services, cross-service query strategies (API composition, CQRS, data replication), when a shared database is the pragmatic choice
- Incremental migration: Strangler Fig pattern, branch-by-abstraction, parallel run validation, common phased migration mistakes
- Independent deployment: API versioning, backward-compatible changes, rollback triggers

---

## Foundational

<details>
<summary>1. What are the honest tradeoffs between monoliths and microservices — what does a monolith give you (simplicity, consistency, debuggability) that microservices take away, why does "monolith first" remain good advice for most teams, and what signals tell you it's genuinely time to decompose?</summary>

**What monoliths give you that microservices take away:**

- **Simplicity**: One codebase, one deploy, one process. No network calls between modules, no distributed tracing needed, no service discovery.
- **Strong consistency**: ACID transactions span the entire domain. No eventual consistency headaches, no sagas, no dual-write problems.
- **Debuggability**: A stack trace shows the full call chain. You can step through code in a single debugger session. Reproducing bugs locally is straightforward.
- **Refactoring ease**: Renaming a function, moving code between modules, or changing a data model is a single commit with compiler/type-checker support. In microservices, the same change may require coordinated deploys across multiple repos.
- **Operational simplicity**: One CI pipeline, one deployment artifact, one set of logs, one database to back up and monitor.

**What microservices give you that monoliths don't:**

- **Independent deployment**: Teams can ship changes to their service without coordinating with every other team.
- **Independent scaling**: Scale the hot path (e.g., search) without scaling everything else.
- **Technology flexibility**: Each service can use the best-fit language, database, or framework.
- **Fault isolation**: A memory leak in one service doesn't crash the entire system (if you have proper resilience patterns).
- **Team autonomy**: Small teams own small services end-to-end, reducing coordination overhead as the organization grows.

**Why "monolith first" is good advice:**

You don't know your domain boundaries well enough at the start. Drawing microservice boundaries wrong is far more expensive than refactoring modules within a monolith. A well-structured monolith (clear module boundaries, dependency injection, domain separation) gives you 80% of the organizational benefits while keeping operational complexity low. You can always extract services later once boundaries are proven stable.

**Signals it's time to decompose:**

- **Deploy conflicts**: Multiple teams stepping on each other in the same codebase, merge conflicts slowing everyone down, release trains causing weeks of delay.
- **Scaling bottlenecks**: One module needs 10x the compute but you're scaling the entire monolith to get it.
- **Team growth**: Beyond ~8-10 engineers on one codebase, coordination cost starts dominating. Conway's Law kicks in.
- **Blast radius**: A bug in one area (e.g., reporting) crashes the entire system, and you need fault isolation.
- **Different deployment cadences**: One part of the system changes daily, another quarterly. Coupling them in one deploy is wasteful and risky.

The key insight: microservices solve **organizational scaling** problems more than technical ones. If you don't have those problems yet, you're paying the distributed systems tax for nothing.

</details>

<details>
<summary>2. How does Conway's Law shape microservice decomposition — why do team boundaries and communication structures directly influence service boundaries, what goes wrong when you design services that don't align with team structure, and how should you use this insight when splitting a monolith?</summary>

Conway's Law states: "Organizations design systems that mirror their communication structures." This isn't just an observation -- it's a force. Fight it and you lose.

**Why team boundaries influence service boundaries:**

A service boundary is an API contract. Whoever owns both sides of that contract can change it freely. When two teams share ownership of a service, every change requires cross-team coordination -- meetings, PRs across repos, aligned release schedules. The path of least resistance is always to put code where your team already owns it, so the system naturally mirrors the org chart.

**What goes wrong when services don't align with teams:**

- **Shared service ownership**: Two teams co-own a service. Neither feels fully responsible. Changes require cross-team PRs, leading to slow iteration and diffused accountability.
- **One team, many services**: A single team owns 8 microservices. They deploy them in lockstep because that's easier than managing 8 independent release cycles. You've built a distributed monolith.
- **Cross-cutting changes**: A feature that should be one team's responsibility requires touching 4 services owned by 3 teams. Coordination cost dominates, and the feature takes 3x longer than it should.

**How to use this insight when splitting a monolith:**

1. **Start with the team structure, not the domain model.** If you have 3 teams, aim for roughly 3 service clusters. Each team should own 1-3 services they can deploy independently.
2. **Apply the "Inverse Conway Maneuver"**: Deliberately restructure teams to match the architecture you want. If you want separate order and inventory services, create separate order and inventory teams first.
3. **One service, one team** is the ideal. One team can own multiple services, but a service should never be co-owned by multiple teams. Cross-team services become coordination bottlenecks.
4. **Measure team cognitive load.** If a team can't hold their entire service domain in their heads, the service is too big. If they're context-switching between unrelated domains, the boundaries are wrong.

The practical implication: the "right" decomposition for a 5-person startup is completely different from the right decomposition for a 200-person engineering org, even for the same product.

</details>

<details>
<summary>3. How do you define service boundaries using Domain-Driven Design — what are bounded contexts, aggregates, and domain events, how do they map to microservice boundaries, and what's the difference between decomposing by business capability vs by subdomain?</summary>

**Bounded Contexts:**

A bounded context is a boundary within which a particular domain model is consistent and meaningful. The same word can mean different things in different contexts -- "Product" in the Catalog context has a name, description, and images; "Product" in the Inventory context is just a SKU with a quantity. Each bounded context has its own ubiquitous language and its own data model. A microservice maps naturally to a bounded context -- it owns its model, its data, and its language.

**Aggregates:**

An aggregate is a cluster of domain objects treated as a single unit for data changes. It has a root entity, and all modifications go through that root. For example, an `Order` aggregate contains `OrderLines`, `ShippingAddress`, and `PaymentInfo` -- you never modify an `OrderLine` directly, you go through the `Order`. Aggregates define the transactional boundary: everything inside one aggregate is consistent (strong consistency), everything across aggregates is eventually consistent. This maps directly to microservice data ownership -- one service owns the aggregate and exposes it via API.

**Domain Events:**

Domain events represent something meaningful that happened in the business domain: `OrderPlaced`, `PaymentCaptured`, `InventoryReserved`. They're the primary communication mechanism between bounded contexts. When the Order service places an order, it emits `OrderPlaced`. The Inventory service reacts by reserving stock. The services are decoupled -- they don't call each other directly, they communicate through events.

**How they map to microservice boundaries:**

- One bounded context = one microservice (or a small cluster of closely related services)
- Aggregates define what data a service owns and its transactional boundary
- Domain events define the contract between services

**Business capability vs subdomain decomposition:**

- **By business capability**: Organize services around what the business does -- "Order Management," "Payment Processing," "Shipping." These are stable over time because business capabilities rarely change even when the implementation does. This is the most common approach.
- **By subdomain**: DDD categorizes subdomains as core (competitive advantage), supporting (necessary but not differentiating), and generic (commodity -- auth, email). This helps prioritize: invest heavily in core subdomain services, use off-the-shelf solutions for generic ones.

In practice, the two approaches converge. Business capabilities map closely to subdomains. The key insight is that you decompose by **business boundaries**, not technical layers. A "database service" or "validation service" is almost always the wrong boundary -- those are shared libraries, not services.

</details>

## Conceptual Depth

<details>
<summary>4. What is the distributed monolith anti-pattern and how do you detect it -- what are the telltale signs of bad decomposition (lock-step deploys, chatty cross-service calls, shared state that couples services), how does it differ from a well-designed microservice architecture, and how do you refactor your way out of one?</summary>

A distributed monolith gives you all the operational complexity of microservices (network latency, partial failures, distributed debugging) with none of the benefits (independent deployment, fault isolation, team autonomy). It's the worst of both worlds.

**Telltale signs:**

- **Lock-step deploys**: Service A can't be deployed without also deploying Service B. If "deploy day" means deploying 5 services together, you have a distributed monolith.
- **Chatty cross-service calls**: A single user request triggers 10-15+ synchronous calls between services. This indicates the boundaries are wrong -- data that's accessed together should live together.
- **Shared database**: Multiple services read/write the same tables. Schema changes require coordinated deploys. The database is the hidden coupling point.
- **Shared libraries with business logic**: A common library contains domain models or business rules. Updating it forces all consuming services to rebuild and redeploy.
- **Temporal coupling**: Service A must call Service B, then C, then D in exact order. If B is down, A can't function at all -- no graceful degradation.
- **Coordinated releases**: A feature requires changes in 3+ services deployed in a specific order. You need a "release coordinator" role.

**How it differs from well-designed microservices:**

| Distributed Monolith | Well-Designed Microservices |
|---|---|
| Deploy together | Deploy independently |
| Failure cascades across services | Failures are isolated (circuit breakers, fallbacks) |
| Shared database or tightly coupled schemas | Database-per-service |
| Synchronous call chains | Async events for most cross-service communication |
| Feature changes touch many services | Most features are one-service changes |

**How to refactor out of it:**

1. **Map the coupling**: Use distributed tracing to visualize call graphs. Identify which services are tightly coupled by measuring deploy correlation (services that always deploy together).
2. **Merge the most coupled services**: Sometimes the fix is fewer, larger services. If two services can't deploy independently, they should be one service. This feels like going backward but it's the pragmatic move.
3. **Replace synchronous chains with events**: If Service A calls B which calls C, ask: does A really need a synchronous answer from C? Often the answer is no -- publish an event and let C process it asynchronously.
4. **Duplicate data to remove shared state**: Give each service its own copy of the data it needs. Accept eventual consistency. The outbox pattern (covered in question 9) makes this safe.
5. **Extract shared libraries carefully**: Shared libraries should contain utilities (HTTP clients, logging), not business logic. If a shared library change forces 5 redeploys, the business logic belongs in a service.
6. **Do it incrementally**: Pick the most painful coupling point, fix it, measure improvement, repeat. Never attempt a big-bang restructuring.

</details>

<details>
<summary>5. When should microservices communicate synchronously (REST/gRPC) vs asynchronously (events/messages) — what are the tradeoffs in coupling, latency, and failure handling, how do you decide which communication style for which interaction, and why do most mature architectures use both?</summary>

**Synchronous (REST/gRPC) tradeoffs:**

- **Coupling**: Temporal coupling -- caller is blocked until the callee responds. If the callee is down, the caller fails too.
- **Latency**: Adds network round-trip per call. Call chains multiply latency (A->B->C = 3 round-trips minimum).
- **Failure handling**: Caller must handle timeouts, retries, and circuit breakers. Partial failures are visible and must be dealt with immediately.
- **Advantage**: Simple request-response semantics. The caller gets an immediate answer. Easy to reason about, debug, and trace.

**Asynchronous (events/messages) tradeoffs:**

- **Coupling**: No temporal coupling. Producer and consumer don't need to be running at the same time. The message broker absorbs the difference.
- **Latency**: Higher latency for the end-to-end flow (message sits in a queue), but the producer returns immediately -- the user isn't waiting.
- **Failure handling**: Built-in retry via the message broker (dead letter queues, redelivery). But failures are invisible and delayed -- you need monitoring to catch stuck messages.
- **Advantage**: Better fault isolation and natural load leveling (consumers process at their own pace).

**Decision framework -- when to use which:**

Use **synchronous** when:
- The caller needs an immediate response to proceed (e.g., "Is this user authenticated?" or "What's the current price?")
- The operation is a query (read path) -- most reads are inherently synchronous
- The interaction is between a client-facing API and its backing service

Use **asynchronous** when:
- The caller doesn't need to wait for the result (e.g., "Send a confirmation email," "Update analytics")
- You're broadcasting a state change that multiple services care about (e.g., `OrderPlaced` triggers inventory, shipping, and notification)
- You need to absorb traffic spikes (queue buffers burst load)
- The operation can tolerate seconds or minutes of delay

**Why mature architectures use both:**

Because the read path and write path have fundamentally different needs. A typical request flow might be: user places an order (synchronous API call to Order service) -> Order service writes to its database and publishes `OrderPlaced` event (asynchronous) -> Payment service picks up the event and processes payment asynchronously -> if the user queries order status, that's synchronous again.

The rule of thumb: **synchronous for queries and immediate commands, asynchronous for reactions and side effects.** Most features naturally decompose this way.

</details>

<details>
<summary>6. How do sagas handle distributed transactions across microservices — what's the difference between choreography and orchestration, how do compensating transactions undo partial work when a step fails, and why must every step in a saga be retry-safe (idempotent)?</summary>

A saga is a sequence of local transactions across services, where each step either succeeds and triggers the next, or fails and triggers compensating transactions to undo previous steps. It replaces distributed (2PC) transactions, which don't scale in microservices.

**Choreography vs Orchestration:**

**Choreography** -- each service listens for events and decides what to do next. No central coordinator.

```
OrderService emits OrderCreated
  -> PaymentService hears it, charges card, emits PaymentCaptured
    -> InventoryService hears it, reserves stock, emits StockReserved
      -> ShippingService hears it, schedules delivery
```

Pros: Simple for 2-3 step flows, no single point of failure, services are fully decoupled.
Cons: Hard to understand the full flow (logic is scattered across services), difficult to add steps or change order, no single place to see "where is this order in the process?"

**Orchestration** -- a central orchestrator (usually its own service) directs the flow.

```
OrderOrchestrator:
  1. Tell PaymentService to charge card -> wait for response
  2. Tell InventoryService to reserve stock -> wait for response
  3. Tell ShippingService to schedule delivery -> wait for response
  On failure at any step -> trigger compensations in reverse order
```

Pros: The entire flow is visible in one place, easy to add/remove/reorder steps, clear error handling logic.
Cons: The orchestrator is a coordination point (not a single point of failure if built correctly), can become a "god service" if it contains business logic that belongs in the individual services.

**Compensating transactions:**

Unlike database rollbacks, compensating transactions are new forward-moving operations that logically undo the effect of a previous step. If the inventory reservation fails after payment was captured, the compensation is "issue a refund" -- not "undo the charge." The data from the original charge still exists; you're adding a new refund record.

Key properties of compensating transactions:
- They must be **semantically reversible** (not all operations are -- you can't unsend an email)
- They should be **idempotent** (safe to run multiple times)
- They should be **retryable** (if the compensation itself fails, you retry it)

**Why every step must be idempotent:**

Messages can be delivered more than once. The broker retries on timeout, the consumer crashes after processing but before acknowledging, network hiccups cause duplicate delivery. If `ReserveStock` runs twice for the same order, you'd reserve double the inventory without idempotency. The fix: every step uses a unique saga/order ID to detect duplicates.

```typescript
// Idempotent inventory reservation
async function reserveStock(orderId: string, items: OrderItem[]) {
  const existing = await db.reservation.findUnique({ where: { orderId } });
  if (existing) return existing; // already processed, return existing result

  return db.reservation.create({
    data: { orderId, items, status: 'reserved' },
  });
}
```

**When to use which:** Choreography for simple (2-3 step) flows where the steps are unlikely to change. Orchestration for complex flows (4+ steps), flows that need visibility/monitoring, or flows where the step order might evolve.

</details>

<details>
<summary>7. How do circuit breakers, retries with exponential backoff, bulkheads, and timeout budgets compose together to prevent cascading failures across microservices — what order should you adopt these patterns, how do they interact (e.g., retries behind a circuit breaker vs in front of it), and what's the typical failure cascade that happens without them?</summary>

**The typical failure cascade without these patterns:**

1. Service C becomes slow (not down -- slow is worse than down)
2. Service B calls C, waits on the default 30s timeout. B's thread pool fills up with waiting requests.
3. Service A calls B, which is now also slow because it's waiting on C. A's thread pool fills up too.
4. Every service in the chain becomes unresponsive. Users see timeouts everywhere. The entire system is down because one service got slow.

This is the **cascade**: slowness propagates upstream, consuming resources (connections, threads, memory) at every hop until everything is exhausted.

**The four patterns and how they compose:**

**1. Timeouts** (adopt first -- highest impact, lowest effort):
Set explicit timeouts on every outbound call. Never rely on defaults. A 2-3 second timeout is usually right for service-to-service calls. This caps how long a slow dependency can hold your resources.

**2. Retries with exponential backoff** (adopt second):
Retry transient failures (5xx, network errors) with increasing delays: 100ms, 200ms, 400ms, etc. Add jitter to prevent thundering herd. Limit to 2-3 retries max. **Critical**: retries must respect the timeout budget (see below).

**3. Circuit breakers** (adopt third):
Track failure rate for each downstream dependency. When failures exceed a threshold (e.g., 50% of requests in the last 10 seconds), "open" the circuit -- fail immediately without making the call. After a cooldown period, allow a test request through ("half-open"). If it succeeds, close the circuit. This prevents piling onto an already-struggling service.

**4. Bulkheads** (adopt when you have multiple downstream dependencies):
Isolate resources per dependency. If you have a connection pool of 100 and two downstream services, dedicate 50 to each. When Service C is slow and exhausts its 50 connections, Service D still has its 50 available. Without bulkheads, one slow dependency consumes all shared resources.

**How they compose (the correct layering):**

```
Request
  -> Timeout budget (outermost: caps total time)
    -> Bulkhead (resource isolation per dependency)
      -> Circuit breaker (fail fast if dependency is broken)
        -> Retry with backoff (retry transient failures)
          -> Actual HTTP call with per-call timeout
```

**Key interactions:**

- **Retries sit INSIDE the circuit breaker.** The circuit breaker sees each retry as a separate call. If retries keep failing, they increment the circuit breaker's failure count, opening it faster. If retries were outside the circuit breaker, you'd retry against an open circuit -- pointless.
- **Timeout budgets constrain retries.** If Service A has a 3-second budget for the entire request and the first call to B takes 2 seconds, you only have 1 second left for the retry. Don't retry if the remaining budget can't accommodate it.
- **Circuit breakers and bulkheads complement each other.** Bulkheads prevent resource exhaustion during the window before the circuit opens. The circuit breaker stops the bleeding once it detects the pattern.

**Timeout budget example:**

```typescript
async function handleRequest(req: Request) {
  const budget = new TimeoutBudget(3000); // 3s total

  // First call gets whatever budget remains
  const userData = await callWithBudget(userService, budget);

  // Second call gets the remaining budget
  const orderData = await callWithBudget(orderService, budget);

  return { ...userData, ...orderData };
}

async function callWithBudget(service: Service, budget: TimeoutBudget) {
  const remaining = budget.remaining();
  if (remaining <= 0) throw new TimeoutError('Budget exhausted');

  return service.call({ timeout: Math.min(remaining, 2000) }); // cap per-call too
}
```

</details>

<details>
<summary>8. How do you implement eventual consistency safely across microservices -- why are idempotency keys essential for reliable message processing, what is the dual-write problem and why does writing to a database and publishing an event as two separate operations cause data loss, how does read-your-own-writes work when the read model lags behind the write model, and when should you abandon eventual consistency and require stronger guarantees?</summary>

**Idempotency keys -- why they're essential:**

In a distributed system, messages will be delivered more than once. The broker retries on timeout, the consumer crashes after processing but before ACKing, network partitions cause redelivery. Without idempotency, "deduct $50" processed twice means the customer loses $100.

An idempotency key is a unique identifier (usually from the producer) that lets the consumer detect and skip duplicates. The pattern is the same as shown in question 6: check for an existing record by the idempotency key, return early if found, otherwise create. A unique constraint on the idempotency key column ensures atomicity -- the second insert fails with a conflict rather than creating a duplicate.

**The dual-write problem:**

When a service needs to update its database AND publish an event, doing both as separate operations is unsafe:

```typescript
// DANGEROUS: dual write
await db.order.update({ where: { id }, data: { status: 'confirmed' } });
await messageBroker.publish('OrderConfirmed', { orderId: id }); // what if this fails?
```

If the app crashes between the two operations, the database is updated but the event is never published. Other services never learn about the change. The data is now permanently inconsistent -- and you won't know it happened.

Publishing first, then writing to the DB has the inverse problem: event published, but the write fails. Now downstream services think something happened that didn't.

The fix is the transactional outbox pattern (covered in detail in question 9).

**Read-your-own-writes:**

When using eventual consistency with a separate read model (e.g., CQRS), there's a lag between writing and the read model catching up. A user creates an order, then immediately views "my orders" -- and the order isn't there yet because the read model hasn't processed the event.

Solutions:
- **Read from the write model for the current user's data**: After a write, temporarily route that specific user's reads to the primary/write database for a short window (e.g., 5 seconds).
- **Return the write result directly**: After creating the order, return the created order in the API response. The client can display it immediately without querying the read model.
- **Version-aware reads**: Include a version/timestamp in the write response. The client passes it on subsequent reads: "give me data at least as fresh as version X." If the read model is behind, either wait briefly or fall back to the write model.
- **Optimistic UI**: The client assumes the write succeeded and displays the result immediately. The read model catches up in the background.

**When to abandon eventual consistency:**

- **Financial transactions**: Account balances, payment processing -- a user spending more than their balance because of a lag is unacceptable. Use synchronous calls or distributed locks.
- **Inventory for high-demand items**: If you have 5 concert tickets left and accept eventual consistency, you might sell 50 tickets before the stock update propagates. For limited inventory, use synchronous reservation with a lock.
- **Security-critical state**: Revoking a user's access must take effect immediately, not "eventually." Permission checks should hit the authoritative source.
- **Ordering guarantees**: When events must be processed in strict order (e.g., `AccountOpened` before `DepositMade`), eventual consistency with unordered queues won't work.

The general rule: eventual consistency for the 90% of operations where seconds of lag are fine, strong consistency for the 10% where correctness is non-negotiable.

</details>

<details>
<summary>9. Why does the transactional outbox pattern exist and how does it solve the dual-write problem — how does writing events to a local outbox table in the same database transaction guarantee atomicity, what reads the outbox (polling vs CDC/change data capture), and what are the tradeoffs compared to alternatives like event sourcing or relying on the message broker's transactional support?</summary>

The transactional outbox exists because you can't atomically write to a database and publish to a message broker in one operation (the dual-write problem described in question 8). The outbox solves this by keeping both writes in the same database transaction.

**How it works:**

Instead of publishing events directly to the message broker, write them to an `outbox` table in the same database transaction as the business data change:

```typescript
async function confirmOrder(orderId: string) {
  await db.$transaction(async (tx) => {
    // Business data change
    await tx.order.update({
      where: { id: orderId },
      data: { status: 'confirmed' },
    });

    // Event written to same DB in same transaction
    await tx.outbox.create({
      data: {
        id: crypto.randomUUID(),
        aggregateType: 'Order',
        aggregateId: orderId,
        eventType: 'OrderConfirmed',
        payload: JSON.stringify({ orderId, status: 'confirmed' }),
        createdAt: new Date(),
        published: false,
      },
    });
  });
  // If the transaction commits, both the order update AND the event are persisted.
  // If it rolls back, neither is persisted. Atomicity guaranteed.
}
```

A separate process reads the outbox and publishes events to the message broker. Two approaches:

**Polling:**
A background job queries the outbox table for unpublished events on a schedule (e.g., every 100ms), publishes them to the broker, then marks them as published.

```typescript
async function pollOutbox() {
  const events = await db.outbox.findMany({
    where: { published: false },
    orderBy: { createdAt: 'asc' },
    take: 100,
  });

  for (const event of events) {
    await messageBroker.publish(event.eventType, event.payload);
    await db.outbox.update({
      where: { id: event.id },
      data: { published: true },
    });
  }
}
```

Pros: Simple to implement, works with any database. Cons: Adds latency (polling interval), puts load on the database, and you need to handle the case where publish succeeds but the `published = true` update fails (consumers must be idempotent -- which they should be anyway).

**CDC (Change Data Capture):**
Tools like Debezium read the database transaction log (WAL in PostgreSQL, binlog in MySQL) and emit each outbox row insert as an event to Kafka automatically. No polling, no additional database queries.

Pros: Near-real-time, no polling load on the DB, captures events in transaction order. Cons: Operational complexity (running Debezium/Kafka Connect), tied to specific database engines, harder to set up and debug.

**Tradeoffs vs alternatives:**

| Approach | Pros | Cons |
|---|---|---|
| **Transactional outbox** | Works with any DB + broker combo, simple mental model, reliable | Extra table, extra process (poller/CDC), at-least-once delivery |
| **Event sourcing** | Events ARE the source of truth (no dual write by design), full audit trail | Major paradigm shift, complex read models (CQRS), steep learning curve |
| **Broker transactional support** (e.g., Kafka transactions) | No outbox table needed | Ties you to a specific broker, limited to brokers that support transactions, still need the DB write to be atomic with the broker write |

The transactional outbox is the pragmatic default for most teams. It solves the dual-write problem reliably without requiring a paradigm shift. Use event sourcing when the domain genuinely benefits from an event-based model (audit-heavy systems, complex temporal queries). Use broker transactions only when you're already deep in the Kafka ecosystem.

</details>

<details>
<summary>10. Why does each microservice need its own database and what are the cross-service query strategies — what coupling does a shared database create, how do you handle queries that need data from multiple services (API composition, CQRS, data replication), and when is a shared database actually the pragmatic choice?</summary>

**Why shared databases couple services:**

When two services share a database, they share a schema. This creates several problems:

- **Schema coupling**: Service A adds a column, Service B's queries break. Every schema migration requires coordinating multiple teams.
- **Performance coupling**: Service A runs a heavy analytical query that locks tables, Service B's writes time out. One service's workload affects another's.
- **Deploy coupling**: A schema migration forces both services to deploy together (the column rename must match the new code in both services).
- **Hidden data dependencies**: Service A starts reading a column that Service B "owns." There's no API contract -- just a shared table. You can't evolve either service's data model independently.

The database becomes the integration layer, which is exactly the coupling microservices are supposed to eliminate.

**Cross-service query strategies:**

**1. API Composition (simplest):**
A service (or an API gateway) calls multiple services and merges the results in memory.

```typescript
// Order details page needs data from 3 services
async function getOrderDetails(orderId: string) {
  const [order, payment, shipping] = await Promise.all([
    orderService.getOrder(orderId),
    paymentService.getPaymentForOrder(orderId),
    shippingService.getShipmentForOrder(orderId),
  ]);

  return { ...order, payment, shipping };
}
```

Pros: Simple, no data duplication. Cons: Multiple network calls per request (latency), availability is the product of all services' availability (if any is down, the query fails), can't do cross-service joins, sorting, or pagination efficiently.

**2. CQRS (Command Query Responsibility Segregation):**
Maintain a separate read model that denormalizes data from multiple services. Services publish events; a read-model consumer aggregates them into a queryable view.

Example: An "Order Dashboard" read model subscribes to events from Order, Payment, and Shipping services. It builds a denormalized view with all the data needed for the dashboard in one table. Queries hit this single read model -- no fan-out.

Pros: Fast reads (single query, no fan-out), can be optimized for specific query patterns. Cons: Eventual consistency (read model lags behind writes), additional infrastructure (event consumers, read model database), more code to maintain.

**3. Data Replication:**
Service A subscribes to Service B's events and maintains a local copy of the data it needs. For example, the Order service keeps a local `products` table with product names and prices, updated via events from the Catalog service.

Pros: No runtime dependency on the source service (reads are local). Cons: Stale data risk, storage duplication, need to handle schema evolution in events.

**When a shared database is pragmatic:**

- **Early-stage products**: When you have 2-3 services and one team, the overhead of database-per-service isn't worth it. Use a shared database with clear schema ownership (each service owns specific tables).
- **Reporting/analytics**: A read-only analytics database that receives data from multiple services via ETL or CDC is effectively a shared read-only database. This is fine because it doesn't create write-path coupling.
- **Tightly coupled bounded contexts**: If two "services" always change together and share the same data model, they might be one service that you've prematurely split. A shared database is the symptom pointing you toward merging them.
- **Transitional state during migration**: When strangling a monolith, the new service and the old monolith may share a database temporarily. This is acceptable as a migration step, not a permanent architecture.

</details>

<details>
<summary>11. How do you migrate from a monolith to microservices incrementally — how does the Strangler Fig pattern work (routing new traffic to the new service while the old code still handles existing flows), how does branch-by-abstraction let you replace internal implementations without changing callers, how does parallel run validate that the new service produces the same results as the old code, and what are the common mistakes in phased migrations?</summary>

**Strangler Fig Pattern:**

Named after the vine that gradually wraps around a tree until the tree dies. You place a routing layer (often an API gateway or reverse proxy) in front of the monolith. New features and migrated functionality route to the new service; everything else continues to hit the monolith. Over time, more routes shift to new services until the monolith handles nothing and can be decommissioned.

```
Client -> API Gateway -> /orders/* -> New Order Service
                      -> /everything-else -> Monolith
```

The key is that both the monolith and the new service run simultaneously. You migrate one route at a time, validate it works, then move on. The monolith shrinks incrementally -- no big-bang cutover.

**Branch by Abstraction:**

Used when the code you're replacing is called internally (not via HTTP routes). You introduce an abstraction layer (interface) in the monolith that wraps the existing implementation:

1. **Create the abstraction**: Define an interface for the module you want to extract (e.g., `PaymentProcessor`).
2. **Make existing code use the abstraction**: All callers go through the interface, not the concrete implementation.
3. **Build the new implementation**: Create a new implementation that calls the external microservice.
4. **Switch**: Swap the implementation behind the interface -- from the local monolith code to the remote service call. Feature flags make this toggleable.
5. **Clean up**: Remove the old implementation from the monolith.

```typescript
// Step 1-2: Abstraction in the monolith
interface PaymentProcessor {
  charge(orderId: string, amount: number): Promise<PaymentResult>;
}

// Old implementation (local monolith code)
class LocalPaymentProcessor implements PaymentProcessor { /* ... */ }

// Step 3: New implementation (calls external service)
class RemotePaymentProcessor implements PaymentProcessor {
  async charge(orderId: string, amount: number) {
    return fetch(`${PAYMENT_SERVICE_URL}/charges`, {
      method: 'POST',
      body: JSON.stringify({ orderId, amount }),
    }).then(r => r.json());
  }
}

// Step 4: Toggle via feature flag
const processor = featureFlags.useRemotePayment
  ? new RemotePaymentProcessor()
  : new LocalPaymentProcessor();
```

**Parallel Run Validation:**

Before cutting over, run both implementations simultaneously and compare results. The monolith code is the "source of truth"; the new service is the "candidate." Both process the same request, but only the monolith's result is returned to the user.

```typescript
async function getOrderDetailsWithValidation(orderId: string) {
  const [monolithResult, serviceResult] = await Promise.allSettled([
    localOrderRepo.getOrderDetails(orderId),
    remoteOrderService.getOrderDetails(orderId),
  ]);

  // Compare results, log discrepancies
  if (monolithResult.status === 'fulfilled' && serviceResult.status === 'fulfilled') {
    if (!deepEqual(monolithResult.value, serviceResult.value)) {
      logger.warn('Result mismatch', { orderId, monolithResult, serviceResult });
    }
  }

  // Always return the monolith result (source of truth)
  if (monolithResult.status === 'fulfilled') return monolithResult.value;
  throw monolithResult.reason;
}
```

**Gotcha with parallel runs**: Be careful with side effects. If both implementations write to a database or charge a credit card, you'll double-process. Parallel runs work best for read operations or when the candidate writes to a shadow/staging environment.

**Common mistakes in phased migrations:**

- **Sharing the database too long**: The monolith and new service share a DB "temporarily" that becomes permanent. Set a deadline and enforce it.
- **Migrating too much at once**: Extracting 3 services simultaneously instead of one at a time. Each extraction has unexpected edge cases; doing multiple in parallel multiplies risk.
- **Not migrating data**: You extract the service but leave its data in the monolith's database. The service still depends on the monolith for reads. Plan data migration as part of the extraction.
- **Ignoring the seam**: Picking a module to extract because it's "easy" rather than because it's at a natural domain boundary. Easy extractions with wrong boundaries create distributed monoliths.
- **No rollback plan**: Every migration step should be reversible. If the new service has issues, you need to route traffic back to the monolith within minutes, not hours.

</details>

<details>
<summary>12. What role does an API gateway play in a microservice architecture — how does it handle routing, authentication, and rate limiting, how does the BFF (Backend for Frontend) pattern differ from a general-purpose gateway, and when does the gateway become a single point of failure or a bottleneck?</summary>

An API gateway is a single entry point that sits between clients and backend services, handling cross-cutting concerns so individual services don't have to.

**Core responsibilities:**

- **Routing**: Maps external URLs to internal services. `/api/orders/*` routes to the Order service, `/api/users/*` to the User service. Clients don't need to know where services live or how many there are.
- **Authentication/Authorization**: Validates JWT tokens, API keys, or OAuth tokens once at the gateway. Backend services receive a verified identity (e.g., via a header like `X-User-Id`) and skip redundant auth checks. This centralizes auth logic instead of reimplementing it in every service.
- **Rate limiting**: Enforces request limits per client, per API key, or per endpoint. Protects backend services from being overwhelmed. Typically implemented with a sliding window counter in Redis.
- **Other cross-cutting concerns**: TLS termination, request/response transformation, CORS handling, request logging, response caching, load balancing.

**BFF (Backend for Frontend) vs general-purpose gateway:**

A general-purpose gateway exposes the same API to all clients (web, mobile, third-party). This becomes problematic because different clients need different data:
- Mobile needs minimal payloads (battery, bandwidth)
- Web dashboard needs rich, aggregated data
- Third-party API needs stable, versioned contracts

The **BFF pattern** creates a dedicated gateway per client type. Each BFF is a thin backend owned by the frontend team that consumes it:

```
Mobile App  -> Mobile BFF  -> Order Service, User Service
Web App     -> Web BFF     -> Order Service, User Service, Analytics Service
Partner API -> Partner BFF -> Order Service (versioned, stable contract)
```

Each BFF aggregates, transforms, and filters data specifically for its client. The mobile BFF returns compact responses; the web BFF returns richer data with related entities pre-joined. This avoids the "one API fits all" problem where the general-purpose gateway becomes bloated with client-specific logic.

**When the gateway becomes a problem:**

- **Single point of failure**: If the gateway goes down, nothing works. Mitigation: run multiple instances behind a load balancer, use health checks, and ensure the gateway itself is stateless so any instance can handle any request.
- **Performance bottleneck**: Every request passes through it. If the gateway does heavy processing (complex auth, response transformation, aggregation), it adds latency to every call. Mitigation: keep the gateway thin (routing + auth + rate limiting only), move aggregation to BFFs or dedicated composition services.
- **Development bottleneck**: If one team owns the gateway and all routing changes require that team, the gateway becomes an organizational bottleneck. Mitigation: let service teams self-serve routing configuration (e.g., declarative route configs that service teams submit via PR).
- **Feature creep**: The gateway starts accumulating business logic -- request validation, data transformation, orchestration. It becomes a monolith itself. The rule: the gateway should be dumb. If it contains `if/else` business logic, that logic belongs in a service.

</details>

<details>
<summary>13. What observability do you need across microservices — how do distributed tracing, correlation IDs, and structured logging work together for cross-service debugging, why is observability harder in microservices than monoliths, and what's the minimum viable observability setup?</summary>

**Why observability is harder in microservices:**

In a monolith, a request is a single process with a single stack trace and a single log stream. In microservices, a single user request may touch 5-10 services, each with its own logs, its own process, and its own failure modes. When something goes wrong, you need to reconstruct the full journey of a request across all those services. Without proper observability, debugging becomes "grep through logs from 8 services hoping the timestamps line up."

**The three pillars and how they work together:**

**1. Correlation IDs:**
A unique request ID (e.g., UUID) generated at the entry point (API gateway or first service) and propagated through every service in the chain via HTTP headers (commonly `X-Request-Id` or W3C `traceparent`). Every log line, every error, every metric includes this ID. When a user reports "my order failed," you search for that correlation ID across all services and see the complete request path.

```typescript
// Middleware that extracts or generates a correlation ID
function correlationMiddleware(req: Request, res: Response, next: NextFunction) {
  const correlationId = req.headers['x-request-id'] || crypto.randomUUID();
  req.correlationId = correlationId;
  res.setHeader('x-request-id', correlationId);

  // Attach to async context so all downstream code can access it
  asyncLocalStorage.run({ correlationId }, () => next());
}
```

**2. Structured Logging:**
Log as JSON objects, not free-text strings. Every log entry includes the correlation ID, service name, timestamp, and structured fields for the event. This makes logs searchable and aggregatable in tools like ELK, Datadog, or Grafana Loki.

```typescript
// Bad: unstructured
console.log(`Order ${orderId} failed for user ${userId}`);

// Good: structured
logger.error({
  event: 'order_failed',
  orderId,
  userId,
  correlationId: getCorrelationId(),
  reason: 'payment_declined',
  duration_ms: 342,
});
```

**3. Distributed Tracing:**
Takes correlation IDs further by recording the timing and hierarchy of every operation. A trace is a tree of spans -- each span represents one operation (an HTTP call, a DB query, a queue publish). You can see that the request took 2.3 seconds total: 200ms in the Order service, 1.8 seconds waiting on Payment, 300ms in Inventory.

Tools like Jaeger, Zipkin, or cloud-native solutions (AWS X-Ray, Datadog APM) visualize this as a waterfall diagram. OpenTelemetry is the standard instrumentation library that works across all of them.

**How they work together:**
- Correlation ID ties all logs for a request together across services
- Structured logging makes those logs searchable and parseable
- Distributed tracing adds timing, hierarchy, and visual debugging on top

When debugging "why was this request slow?": tracing shows you which service was the bottleneck. Logs for that service (filtered by correlation ID) show you the specific error or slow query.

**Minimum viable observability setup:**

1. **Structured JSON logging** with correlation IDs propagated via headers -- this alone is transformative and costs almost nothing to implement.
2. **Centralized log aggregation** -- all services ship logs to one place (ELK stack, CloudWatch Logs, Grafana Loki). Being able to search across all services by correlation ID is the baseline.
3. **Health check endpoints** -- every service exposes `/health` so you know what's up and what's down.
4. **Basic metrics** -- request rate, error rate, latency percentiles (the RED method) per service. Prometheus + Grafana is the standard open-source stack.
5. **Alerting on error rate spikes** -- if a service's error rate jumps from 0.1% to 5%, page someone.

Add distributed tracing when you have enough services (5+) that log-based debugging becomes impractical. For 2-3 services, structured logs with correlation IDs are usually sufficient.

</details>

<details>
<summary>14. How do you achieve independent deployment in microservices — what API versioning strategies enable services to deploy independently, what makes a change backward-compatible vs breaking, what should trigger a rollback, and why is independent deployability the single most important microservice benefit?</summary>

**Why independent deployability is the most important benefit:**

If you can't deploy services independently, you don't have microservices -- you have a distributed monolith (as covered in question 4). Independent deployment is what gives you: faster release cycles (teams ship without waiting for others), smaller blast radius (a bad deploy affects one service, not the entire system), and team autonomy (each team owns their release cadence). Every other microservice benefit -- fault isolation, independent scaling, technology flexibility -- depends on being able to deploy independently.

**API versioning strategies:**

**1. URL path versioning** (`/api/v1/orders`, `/api/v2/orders`):
Simple and explicit. Clients know exactly which version they're using. Works well for external/public APIs. Downside: maintaining multiple versions of route handlers in the codebase.

**2. Header versioning** (`Accept: application/vnd.myapp.v2+json`):
Keeps URLs clean. Better for internal service-to-service communication. Downside: less discoverable, harder to test in a browser.

**3. Additive-only changes (no versioning needed):**
The best strategy is to avoid breaking changes entirely. If you only make additive changes, you never need versioning. This is the default approach for internal services.

**Backward-compatible (non-breaking) changes:**

- Adding a new field to a response (existing clients ignore it)
- Adding a new optional field to a request
- Adding a new endpoint
- Widening the accepted input (e.g., accepting both string and number for a field)

**Breaking changes:**

- Removing or renaming a response field
- Removing or renaming an endpoint
- Adding a new required field to a request
- Changing the type of an existing field
- Changing the meaning/behavior of an existing field

**How to handle breaking changes safely:**

1. **Expand and contract (two-phase)**: First deploy the new version that supports both old and new formats (expand). Wait for all consumers to migrate. Then deploy again to remove the old format (contract).
2. **Parallel endpoints**: Deploy the new endpoint alongside the old one. Migrate consumers one by one. Deprecate and remove the old endpoint once all consumers have moved.
3. **Consumer-driven contract testing**: Each consumer defines the contract it expects. Before deploying a provider change, run the consumer contract tests to catch breaking changes before they reach production.

```typescript
// Expand phase: support both old and new field names
interface OrderResponse {
  id: string;
  total: number;          // new field name
  totalAmount?: number;   // old field name, still populated for backward compat
}

function toOrderResponse(order: Order): OrderResponse {
  return {
    id: order.id,
    total: order.total,
    totalAmount: order.total, // populate both during migration
  };
}
```

**Rollback triggers:**

- Error rate exceeds threshold (e.g., >1% 5xx responses in the first 5 minutes)
- Latency p99 spikes significantly (e.g., 3x baseline)
- Health check failures
- Key business metric anomaly (e.g., order completion rate drops)
- Any data corruption or consistency violation

Automate rollbacks where possible. Canary deployments (route 5% of traffic to the new version, monitor, then gradually increase) catch issues before they affect all users. If the canary shows problems, automatic rollback kicks in.

</details>

## Practical — Service Design & Communication

<details>
<summary>15. Design the service decomposition for an e-commerce system — identify the bounded contexts (orders, inventory, payments, shipping, users), define the API contracts between them, decide which communications are synchronous vs async, and explain why you drew the boundaries where you did</summary>

**Bounded Contexts and Services:**

| Service | Owns | Core Aggregate |
|---|---|---|
| **Catalog Service** | Product info, categories, pricing, search | Product |
| **User Service** | Registration, profiles, authentication | User |
| **Order Service** | Order lifecycle, order lines, order status | Order |
| **Inventory Service** | Stock levels, reservations, warehouse locations | StockItem |
| **Payment Service** | Payment processing, refunds, payment methods | Payment |
| **Shipping Service** | Shipment tracking, carrier selection, delivery scheduling | Shipment |
| **Notification Service** | Email, SMS, push notifications | (stateless, no core aggregate) |

**Why these boundaries:**

Each service maps to a distinct business capability with its own data, its own lifecycle, and its own team. The key test: can a feature be implemented by changing only one service? "Add a new payment method" touches only Payment. "Add a product attribute" touches only Catalog. "Change shipping carrier" touches only Shipping. Cross-service features (like placing an order) are coordinated via events, not coupled code.

**API Contracts between services:**

Each service exposes standard REST CRUD endpoints scoped to its domain. Representative examples:

```
Order Service:
  POST /orders, GET /orders/:id, PATCH /orders/:id/status

Inventory Service:
  POST /reservations, DELETE /reservations/:orderId, GET /stock/:sku

Payment Service:
  POST /charges, POST /refunds
```

The other services (Shipping, Catalog, User) follow the same pattern — thin REST interfaces that expose only what other services need. The key constraint: services never expose internal data models, only the operations other services need to trigger.

**Communication patterns — synchronous vs async:**

**Synchronous (request needs an immediate answer):**
- User browses catalog -> **Catalog Service** (query, needs immediate response)
- User views order details -> **Order Service** -> API composition with Payment and Shipping (as covered in question 10)
- Checkout validates stock -> **Inventory Service** `GET /stock/:sku` (must know availability NOW)
- Authentication -> **User Service** `POST /auth/token`

**Asynchronous (reactions and side effects):**
- Order placed -> `OrderPlaced` event:
  - Payment Service reacts: charge the card
  - Inventory Service reacts: reserve stock
  - Notification Service reacts: send confirmation email
- Payment captured -> `PaymentCaptured` event:
  - Shipping Service reacts: schedule shipment
  - Order Service reacts: update status to "paid"
- Payment failed -> `PaymentFailed` event:
  - Inventory Service reacts: release reservation (compensating action)
  - Order Service reacts: update status to "payment_failed"
  - Notification Service reacts: send payment failure email
- Shipment delivered -> `ShipmentDelivered` event:
  - Order Service reacts: update status to "delivered"

**The order placement flow (orchestrated saga):**

```
1. Client -> POST /orders (sync) -> Order Service creates order, status: "pending"
2. Order Service emits OrderPlaced event (async)
3. Payment Service charges card
   - Success: emits PaymentCaptured
   - Failure: emits PaymentFailed -> Inventory releases reservation (compensation)
4. Inventory Service reserves stock
   - Success: emits StockReserved
   - Failure: emits ReservationFailed -> Payment issues refund (compensation)
5. Shipping Service schedules delivery -> emits ShipmentScheduled
6. Order Service updates status to "confirmed"
```

**Why the read path is sync and the write path is async:** Users expect immediate responses when browsing or querying. But order processing, payment, and shipping are naturally asynchronous -- the user doesn't sit and wait for the carrier to be assigned. Show "order received" immediately, then update status via events as each step completes.

</details>

<details>
<summary>16. Implement a saga for an order placement workflow that spans payment, inventory, and shipping services — show the event flow for choreography, implement compensating actions (refund on inventory failure), and demonstrate how to handle the failure case where the compensation itself fails</summary>

**Choreography event flow:**

```
Happy path:
  OrderService  --OrderPlaced-->     PaymentService
  PaymentService --PaymentCaptured--> InventoryService
  InventoryService --StockReserved--> ShippingService
  ShippingService --ShipmentScheduled--> OrderService (marks order "confirmed")

Payment failure:
  OrderService --OrderPlaced--> PaymentService
  PaymentService --PaymentFailed--> OrderService (marks order "failed")

Inventory failure (needs compensation):
  OrderService --OrderPlaced--> PaymentService
  PaymentService --PaymentCaptured--> InventoryService
  InventoryService --ReservationFailed--> PaymentService (triggers refund)
  PaymentService --RefundIssued--> OrderService (marks order "failed")
```

**Implementation — event handlers in each service:**

```typescript
// === Shared event types ===
interface OrderPlaced {
  type: 'OrderPlaced';
  orderId: string;
  userId: string;
  items: { sku: string; quantity: number; price: number }[];
  totalAmount: number;
}

interface PaymentCaptured {
  type: 'PaymentCaptured';
  orderId: string;
  paymentId: string;
  amount: number;
}

interface ReservationFailed {
  type: 'ReservationFailed';
  orderId: string;
  reason: string;
  paymentId: string; // needed for refund compensation
}

// === Payment Service ===
// Each handler uses the idempotent check-then-create pattern from question 6
async function handleOrderPlaced(event: OrderPlaced) {
  const existing = await db.payment.findUnique({ where: { idempotencyKey: event.orderId } });
  if (existing) return; // idempotent — already processed

  try {
    const payment = await chargeCard(event.userId, event.totalAmount);
    await db.payment.create({
      data: { id: payment.id, idempotencyKey: event.orderId, orderId: event.orderId, amount: event.totalAmount, status: 'captured' },
    });
    await publish({ type: 'PaymentCaptured', orderId: event.orderId, paymentId: payment.id, amount: event.totalAmount });
  } catch (err) {
    await publish({ type: 'PaymentFailed', orderId: event.orderId, reason: err.message });
  }
}

// Compensation: refund when inventory reservation fails
async function handleReservationFailed(event: ReservationFailed) {
  const payment = await db.payment.findUnique({
    where: { idempotencyKey: event.orderId },
  });
  if (!payment || payment.status === 'refunded') return; // idempotent

  await issueRefund(event.paymentId, payment.amount);
  await db.payment.update({
    where: { id: payment.id },
    data: { status: 'refunded' },
  });
  await publish({ type: 'RefundIssued', orderId: event.orderId, paymentId: event.paymentId });
}

// === Inventory Service ===
async function handlePaymentCaptured(event: PaymentCaptured) {
  const existing = await db.reservation.findUnique({
    where: { orderId: event.orderId },
  });
  if (existing) return; // idempotent

  try {
    // Attempt to reserve all items
    await db.$transaction(async (tx) => {
      for (const item of await getOrderItems(event.orderId)) {
        const stock = await tx.stock.findUnique({ where: { sku: item.sku } });
        if (!stock || stock.available < item.quantity) {
          throw new Error(`Insufficient stock for ${item.sku}`);
        }
        await tx.stock.update({
          where: { sku: item.sku },
          data: { available: { decrement: item.quantity } },
        });
      }
      await tx.reservation.create({
        data: { orderId: event.orderId, status: 'reserved' },
      });
    });
    await publish({ type: 'StockReserved', orderId: event.orderId });
  } catch (err) {
    await publish({
      type: 'ReservationFailed',
      orderId: event.orderId,
      reason: err.message,
      paymentId: event.paymentId,
    });
  }
}
```

**Handling the case where compensation itself fails:**

The refund call to the payment gateway might fail (network timeout, gateway down). This is the hardest problem in sagas. Strategies:

**1. Retry with exponential backoff and dead letter queue:**

```typescript
async function handleReservationFailed(event: ReservationFailed) {
  const payment = await db.payment.findUnique({
    where: { idempotencyKey: event.orderId },
  });
  if (!payment || payment.status === 'refunded') return;

  try {
    await issueRefund(event.paymentId, payment.amount);
    await db.payment.update({
      where: { id: payment.id },
      data: { status: 'refunded' },
    });
  } catch (err) {
    // Mark as needing manual intervention after max retries
    if (event.retryCount >= MAX_RETRIES) {
      await db.payment.update({
        where: { id: payment.id },
        data: { status: 'refund_failed' },
      });
      // Send to dead letter queue for manual resolution
      await publish({
        type: 'CompensationFailed',
        orderId: event.orderId,
        action: 'refund',
        reason: err.message,
      });
      // Alert on-call engineer
      await alerting.page('Compensation failed - manual refund needed', { event });
      return;
    }
    // Re-throw to trigger broker retry with backoff
    throw err;
  }
}
```

**2. Compensation log table:**
Track every compensation attempt in a dedicated table. A background reconciliation job periodically scans for stuck compensations and retries them. This decouples the retry mechanism from the message broker.

**3. Manual resolution as the final safety net:**
After all automated retries are exhausted, flag the order for human review. An admin dashboard shows orders in `refund_failed` state with all context needed to resolve manually. This isn't a design failure -- it's acknowledging that some failure modes require human judgment.

The key principle: design compensations to be idempotent and retryable. Most compensation failures are transient (network blips, temporary outages). With enough retries and backoff, they self-resolve. The dead letter queue + alerting catches the rare cases that don't.

</details>

<details>
<summary>17. Implement resilience patterns for a service that calls two downstream dependencies — show circuit breaker configuration (with a library like opossum), retry with exponential backoff, timeout budgets (if service A has 3s budget, downstream calls must share it), and bulkhead isolation to prevent one slow dependency from affecting the other</summary>

This builds on the composition concepts from question 7 with a concrete implementation.

**Timeout Budget:**

```typescript
class TimeoutBudget {
  private startTime: number;

  constructor(private totalMs: number) {
    this.startTime = Date.now();
  }

  remaining(): number {
    return Math.max(0, this.totalMs - (Date.now() - this.startTime));
  }

  hasTime(): boolean {
    return this.remaining() > 0;
  }
}
```

**Retry with exponential backoff and jitter:**

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxRetries: number; baseDelayMs: number; budget: TimeoutBudget }
): Promise<T> {
  const { maxRetries, baseDelayMs, budget } = options;
  let lastError: Error;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    if (!budget.hasTime()) throw new Error('Timeout budget exhausted');

    try {
      return await fn();
    } catch (err) {
      lastError = err as Error;

      if (attempt === maxRetries) break;

      // Exponential backoff with jitter
      const delay = baseDelayMs * Math.pow(2, attempt);
      const jitter = delay * 0.5 * Math.random();
      const waitMs = Math.min(delay + jitter, budget.remaining());

      if (waitMs <= 0) break;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
    }
  }

  throw lastError!;
}
```

**Circuit breakers with opossum:**

```typescript
import CircuitBreaker from 'opossum';

// Separate circuit breaker per dependency
const paymentBreaker = new CircuitBreaker(callPaymentService, {
  timeout: 2000,                  // per-call timeout: 2s
  errorThresholdPercentage: 50,   // open circuit when 50% of requests fail
  resetTimeout: 10000,            // try again after 10s in half-open state
  volumeThreshold: 5,             // minimum 5 requests before tripping
});

const inventoryBreaker = new CircuitBreaker(callInventoryService, {
  timeout: 1500,
  errorThresholdPercentage: 50,
  resetTimeout: 10000,
  volumeThreshold: 5,
});

// Fallbacks for graceful degradation
paymentBreaker.fallback(() => ({
  status: 'unavailable',
  message: 'Payment service temporarily unavailable',
}));

inventoryBreaker.fallback(() => ({
  status: 'unavailable',
  message: 'Inventory check unavailable — showing cached data',
}));

// Monitor circuit state changes
paymentBreaker.on('open', () =>
  logger.warn('Payment circuit OPEN — failing fast')
);
paymentBreaker.on('halfOpen', () =>
  logger.info('Payment circuit HALF-OPEN — testing')
);
paymentBreaker.on('close', () =>
  logger.info('Payment circuit CLOSED — recovered')
);
```

**Bulkhead isolation:**

In Node.js (single-threaded), bulkheads don't isolate thread pools like in Java. Instead, you limit concurrent in-flight requests per dependency using a semaphore:

```typescript
class Semaphore {
  private current = 0;
  private queue: Array<() => void> = [];

  constructor(private maxConcurrent: number) {}

  async acquire(): Promise<void> {
    if (this.current < this.maxConcurrent) {
      this.current++;
      return;
    }
    // Wait in line
    return new Promise((resolve) => this.queue.push(resolve));
  }

  release(): void {
    this.current--;
    const next = this.queue.shift();
    if (next) {
      this.current++;
      next();
    }
  }
}

// Isolate: max 10 concurrent calls to payment, max 10 to inventory
const paymentBulkhead = new Semaphore(10);
const inventoryBulkhead = new Semaphore(10);
```

**Composing everything — the request handler:**

```typescript
async function handleOrderRequest(req: Request, res: Response) {
  const budget = new TimeoutBudget(3000); // 3s total budget

  try {
    // Both calls run in parallel, each with its own bulkhead + circuit breaker + retry
    const [paymentResult, inventoryResult] = await Promise.all([
      resilientCall(paymentBreaker, paymentBulkhead, budget, {
        maxRetries: 2,
        baseDelayMs: 100,
      }),
      resilientCall(inventoryBreaker, inventoryBulkhead, budget, {
        maxRetries: 2,
        baseDelayMs: 100,
      }),
    ]);

    res.json({ payment: paymentResult, inventory: inventoryResult });
  } catch (err) {
    res.status(503).json({ error: 'Service temporarily unavailable' });
  }
}

async function resilientCall(
  breaker: CircuitBreaker,
  bulkhead: Semaphore,
  budget: TimeoutBudget,
  retryOpts: { maxRetries: number; baseDelayMs: number }
) {
  await bulkhead.acquire();
  try {
    return await withRetry(
      () => breaker.fire(), // circuit breaker wraps the actual call
      { ...retryOpts, budget }
    );
  } finally {
    bulkhead.release();
  }
}
```

**The composition order (matching the layering from question 7):**

```
Request (3s budget)
  -> Promise.all (parallel calls)
    -> Bulkhead (max 10 concurrent per dependency)
      -> Retry (2 retries with backoff, respects budget)
        -> Circuit Breaker (opossum, fails fast if dependency is broken)
          -> HTTP call (with per-call timeout via opossum's timeout option)
```

If the inventory service goes down: its circuit opens after 5 failed requests, subsequent calls fail instantly (no waiting), the payment service continues working normally because the bulkhead isolates their resources. When inventory recovers, the half-open state lets one test request through, and if it succeeds, the circuit closes.

</details>

<details>
<summary>18. Set up an API gateway that routes requests to multiple backend services -- show the routing configuration, implement authentication at the gateway layer, add rate limiting, and explain what happens when the gateway becomes a performance bottleneck or single point of failure and how to mitigate these risks</summary>

**Gateway implementation with Express (lightweight, for understanding the concepts):**

```typescript
import express from 'express';
import { createProxyMiddleware } from 'http-proxy-middleware';
import jwt from 'jsonwebtoken';
import Redis from 'ioredis';

const app = express();
const redis = new Redis();

// === 1. Routing Configuration ===
const routes: Record<string, { target: string; requiresAuth: boolean }> = {
  '/api/orders':    { target: 'http://order-service:3001',    requiresAuth: true },
  '/api/products':  { target: 'http://catalog-service:3002',  requiresAuth: false },
  '/api/payments':  { target: 'http://payment-service:3003',  requiresAuth: true },
  '/api/users':     { target: 'http://user-service:3004',     requiresAuth: true },
  '/api/auth':      { target: 'http://user-service:3004',     requiresAuth: false },
};

// === 2. Authentication Middleware ===
function authenticate(req: express.Request, res: express.Response, next: express.NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Missing token' });

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as { userId: string; roles: string[] };
    // Forward verified identity to downstream services
    req.headers['x-user-id'] = payload.userId;
    req.headers['x-user-roles'] = payload.roles.join(',');
    // Remove the raw token — downstream services trust the gateway's headers
    delete req.headers.authorization;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// === 3. Rate Limiting Middleware (fixed window counter with Redis) ===
function rateLimit(maxRequests: number, windowSeconds: number) {
  return async (req: express.Request, res: express.Response, next: express.NextFunction) => {
    // Rate limit by API key or IP
    const key = `ratelimit:${req.headers['x-api-key'] || req.ip}`;
    const current = await redis.incr(key);

    if (current === 1) {
      await redis.expire(key, windowSeconds);
    }

    res.setHeader('X-RateLimit-Limit', maxRequests);
    res.setHeader('X-RateLimit-Remaining', Math.max(0, maxRequests - current));

    if (current > maxRequests) {
      return res.status(429).json({
        error: 'Rate limit exceeded',
        retryAfter: await redis.ttl(key),
      });
    }
    next();
  };
}

// === 4. Wire it all together ===
// Apply rate limiting globally
app.use(rateLimit(100, 60)); // 100 requests per minute

// Set up proxy routes
for (const [path, config] of Object.entries(routes)) {
  const middlewares: express.RequestHandler[] = [];

  if (config.requiresAuth) {
    middlewares.push(authenticate);
  }

  middlewares.push(
    createProxyMiddleware({
      target: config.target,
      changeOrigin: true,
      timeout: 5000,
      proxyTimeout: 5000,
    }) as express.RequestHandler
  );

  app.use(path, ...middlewares);
}

app.listen(8080);
```

**Production gateway configuration (NGINX example for comparison):**

Note: The NGINX config below uses a per-second rate (10r/s) rather than the per-minute rate in the Express example — this is intentional, as NGINX's `limit_req_zone` only supports per-second rates natively. Both achieve the same goal with different granularity.

```nginx
upstream order_service {
    server order-service-1:3001;
    server order-service-2:3001;  # load balancing across instances
}

upstream catalog_service {
    server catalog-service-1:3002;
    server catalog-service-2:3002;
}

# Rate limiting zone: 10 requests/second per IP
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;

    # Apply rate limiting with burst allowance
    limit_req zone=api_limit burst=20 nodelay;

    location /api/orders/ {
        proxy_pass http://order_service;
        proxy_set_header X-Request-Id $request_id;
        proxy_connect_timeout 2s;
        proxy_read_timeout 5s;
    }

    location /api/products/ {
        proxy_pass http://catalog_service;
        proxy_set_header X-Request-Id $request_id;
        proxy_connect_timeout 2s;
        proxy_read_timeout 5s;
    }
}
```

**When the gateway becomes a bottleneck or SPOF and how to mitigate:**

**Single point of failure mitigations:**
- **Multiple instances behind a load balancer**: Run 3+ gateway instances. A cloud load balancer (ALB, NLB) or DNS round-robin distributes traffic. If one instance dies, the others absorb the load.
- **Stateless gateway**: All state (rate limit counters, sessions) lives in Redis, not in the gateway process. Any instance can handle any request.
- **Health checks**: The load balancer health-checks each gateway instance and removes unhealthy ones from rotation automatically.

**Performance bottleneck mitigations:**
- **Keep the gateway thin**: Only routing, auth, and rate limiting. No business logic, no response transformation, no request aggregation. Heavy work belongs in services or BFFs (as covered in question 12).
- **Horizontal scaling**: Gateway instances are stateless, so you can add more behind the load balancer to handle increased traffic.
- **Connection pooling**: Reuse connections to backend services instead of opening a new TCP connection per request.
- **Caching**: Cache public, read-heavy responses at the gateway (e.g., product catalog data) to reduce backend load.
- **Offload TLS termination**: Use the load balancer for TLS termination so the gateway doesn't spend CPU on encryption.

**When to use a managed gateway (Kong, AWS API Gateway, Envoy) instead of building your own:** Almost always in production. The Express example above illustrates the concepts, but production gateways need graceful shutdown, connection draining, circuit breaking, observability integration, and operational maturity that managed solutions provide out of the box.

</details>

## Practical — Migration & Deployment

<details>
<summary>19. You suspect your microservice architecture has become a distributed monolith -- two services must always deploy together and one service makes 15+ synchronous calls to another per request. Walk through how you would diagnose the coupling (dependency mapping, deploy frequency analysis, call graph tracing), identify which boundary is wrong, and refactor toward proper service independence without a big-bang rewrite.</summary>

**Step 1: Diagnose the coupling with data, not gut feelings.**

**Deploy frequency analysis:**
Pull deploy history from your CI/CD system. Look for services that always deploy within the same time window. If Service A and Service B deploy within 30 minutes of each other 90% of the time, they're coupled.

```sql
-- Example: find services that always deploy together
SELECT a.service_name, b.service_name, COUNT(*) as co_deploys
FROM deploys a
JOIN deploys b ON a.deploy_date = b.deploy_date AND a.service_name < b.service_name
GROUP BY a.service_name, b.service_name
ORDER BY co_deploys DESC;
```

**Call graph tracing:**
Use distributed tracing (Jaeger/Datadog) to visualize the call graph for a single request. If one request to Service A triggers 15+ synchronous calls to Service B, the trace will show it clearly -- a waterfall of sequential calls, each adding latency.

Look for:
- **Fan-out pattern**: One service calling the same downstream service repeatedly in a loop (N+1 problem at the service level)
- **Chatty calls**: Multiple small calls that could be one batch call
- **Hidden dependencies**: Service A calls B, which calls C, which calls A again (circular dependency)

**Dependency mapping:**
Build a service dependency graph from tracing data or API gateway logs. Visualize which services talk to which. Look for:
- Services with high fan-in (many services depend on them) -- potential bottleneck
- Bidirectional dependencies (A calls B and B calls A) -- definitely wrong
- Clusters of tightly connected services -- potential merge candidates

**Step 2: Identify which boundary is wrong.**

The 15 synchronous calls per request are the strongest signal. Ask: **why does Service A need to call Service B so often?**

Common root causes:
- **Wrong data ownership**: Service A needs data that lives in Service B. The data should either move to A (if A uses it most) or A should maintain a local copy via events (data replication, as covered in question 10).
- **Split too fine**: Service A and B are really one bounded context that was prematurely split. The "boundary" between them isn't a real domain boundary -- it's a technical split (e.g., "API service" and "data service").
- **Missing aggregate**: Service A orchestrates logic that should be encapsulated in one place. The chatty calls are A reaching into B to assemble something that B should expose as a single operation.

**Step 3: Refactor incrementally.**

**Option A -- Merge the coupled services (if they're one bounded context):**
This is the right move if the two services share the same data model, always change together, and are owned by the same team. Merge them. It's not a step backward -- it's correcting a premature split.

1. Route both services' traffic through one of them
2. Migrate the other's logic into the surviving service
3. Migrate the data
4. Decommission the empty service

**Option B -- Replicate data to eliminate runtime calls (if they're separate bounded contexts):**
If the services genuinely belong to different domains but A needs B's data for read operations:

1. Have Service B publish events when its data changes (using the outbox pattern from question 9)
2. Service A subscribes and maintains a local read-only copy of the data it needs
3. Replace the 15 synchronous calls with local database reads
4. Accept eventual consistency for this data (which is fine for reads in most cases)

**Option C -- Batch API to reduce chattiness (quick win while planning the real fix):**
If the 15 calls are "get item by ID" in a loop, replace with a single batch endpoint:

```typescript
// Before: 15 calls
for (const id of itemIds) {
  const item = await serviceB.getItem(id); // 15 HTTP round-trips
}

// After: 1 call
const items = await serviceB.getItemsBatch(itemIds); // 1 HTTP round-trip
```

This doesn't fix the coupling, but it reduces latency and makes the system livable while you plan the proper refactoring.

**Step 4: Validate the fix.**

After refactoring, check:
- Can each service deploy independently? Deploy A without B and verify nothing breaks.
- Has the call graph simplified? Check tracing -- the 15-call fan-out should be gone.
- Has deploy correlation decreased? Track over 2-4 weeks to confirm A and B no longer deploy in lockstep.
- Has latency improved? Fewer synchronous hops should mean lower p50/p99 latency.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>20. Tell me about a time you decomposed a monolith into microservices (or decided not to) — what drove the decision, how did you identify service boundaries, and what challenges did you face during the migration?</summary>

**What the interviewer is looking for:**
- Structured decision-making (not "microservices are better"), showing you understand the tradeoffs from question 1
- Domain-driven thinking when identifying boundaries (not random technical splits)
- Awareness of migration complexity and practical challenges
- Willingness to choose NOT to decompose when it's the right call

**Suggested structure (STAR + tradeoffs):**

1. **Situation**: What was the system, what was the team size, what was the pain point?
2. **Decision process**: What drove the decision to decompose (or not)? What signals did you evaluate? (Deploy conflicts? Scaling bottlenecks? Team growth?)
3. **Boundary identification**: How did you identify where to cut? (DDD sessions, Conway's Law alignment, looking at change frequency)
4. **Migration approach**: Strangler Fig? Big bang? What pattern did you use and why?
5. **Challenges and what you learned**: What went wrong? Data migration issues? Distributed consistency surprises? Performance regressions from network calls?

**Example outline (decomposed):**

"Our e-commerce platform was a Django monolith serving 50 requests/second with 12 engineers. Deploy conflicts were the main pain -- the checkout team and catalog team would step on each other constantly, and releases took 2 weeks of coordination. We identified three bounded contexts: catalog (high read traffic, rarely changed), checkout (high write traffic, changed daily), and user management (stable, low change rate). We extracted catalog first using the Strangler Fig pattern -- new search endpoints went to the new service, old product pages still hit the monolith. The hardest part was data migration: the monolith's product table had checkout-specific columns mixed in, and untangling that took 3 weeks of dual-writes before we could cut over. Key lesson: we should have invested in the outbox pattern earlier -- we lost events during the dual-write window and had to reconcile manually."

**Example outline (decided NOT to decompose):**

"We had a 4-person team with a Node.js monolith. Two senior engineers pushed for microservices because 'that's what modern teams do.' I pushed back: we had no deploy conflicts (one team, one codebase), no scaling bottlenecks (300 req/s on a single instance), and our biggest pain points were actually missing tests and unclear module boundaries -- not architectural. Instead, we invested in modularizing the monolith: clear directory structure by domain, dependency injection, integration tests at module boundaries. A year later, we still hadn't needed microservices. The lesson: the question isn't 'should we use microservices?' -- it's 'what organizational or scaling problem are we actually trying to solve?'"

**Key points to hit:**
- Show you evaluated concrete signals, not just followed hype
- Mention Conway's Law / team structure as a factor
- Be honest about what went wrong -- interviewers want real experience, not a success story
- If you chose not to decompose, explain what you did instead and why it was sufficient

</details>

<details>
<summary>21. Describe a cascading failure you experienced in a microservice architecture — what triggered it, how did you diagnose the chain of failures, and what resilience patterns did you add to prevent recurrence?</summary>

**What the interviewer is looking for:**
- That you understand cascading failures at a visceral level -- not just textbook definitions but real debugging under pressure
- Systematic diagnosis: how you traced the root cause through multiple services
- That you know the resilience patterns from question 7 and have applied them in practice
- Post-incident improvement: you didn't just fix the symptom, you added structural defenses

**Suggested structure:**

1. **The trigger**: What initially went wrong? (Database slow query, downstream service deployment, traffic spike, memory leak)
2. **The cascade**: How did it spread? Which services were affected and in what order? What were the symptoms?
3. **Diagnosis**: How did you figure out the root cause? (Tracing, dashboards, logs, process of elimination)
4. **Immediate fix**: What did you do to stop the bleeding? (Kill the slow query, restart pods, feature-flag off the problematic flow)
5. **Structural fix**: What resilience patterns did you add afterward?
6. **What you learned**: What would you do differently next time?

**Example outline:**

"We had a product recommendation service that called a machine learning inference API. During a traffic spike on Black Friday, the ML service started responding in 8-12 seconds instead of the usual 200ms. Our product pages called the recommendation service synchronously with no timeout configured (defaulting to 30s). The product page service's connection pool filled up waiting for recommendation responses. Since the same connection pool was shared across all routes, the entire product service became unresponsive -- not just the pages with recommendations. Checkout, search, everything went down.

Diagnosis: We noticed product service pods were healthy (no crashes) but all returning 504s. Distributed tracing showed every in-flight request was blocked on the recommendation call. The ML service dashboards confirmed the latency spike.

Immediate fix: We restarted the product service pods and feature-flagged off recommendations, which restored service within 2 minutes.

Structural fixes we added: (1) Explicit 500ms timeout on the recommendation call. (2) Circuit breaker using opossum -- if recommendation fails 3 times in 10 seconds, we skip recommendations entirely and show a generic product list. (3) Bulkhead -- dedicated connection pool for recommendation calls so a slow recommendation service can't starve other routes. (4) Made recommendations async -- the page loads without them, and a client-side request fetches recommendations after initial render.

Key lesson: the absence of a timeout was the real bug. A slow dependency without a timeout is worse than a down dependency because it silently consumes resources instead of failing fast."

**Key points to hit:**
- Emphasize that slow is worse than down (this shows depth of understanding)
- Name specific resilience patterns and explain why each one matters for your scenario
- Show the diagnosis was systematic, not random restarts
- Mention the post-incident review and structural improvements -- not just the firefighting

</details>

<details>
<summary>22. Describe a time you dealt with a data consistency issue across microservices — what was the scenario, how did you handle it (saga, eventual consistency, compensating action), and what would you do differently?</summary>

**What the interviewer is looking for:**
- Real understanding of distributed data problems -- not just "we used eventual consistency"
- That you can articulate the specific consistency guarantee that was violated and why it mattered
- Practical problem-solving: how you detected the inconsistency, how you fixed the immediate issue, and what you changed to prevent it
- Awareness of the tradeoffs from questions 8 and 9 (dual-write problem, outbox pattern, idempotency)

**Suggested structure:**

1. **The scenario**: What services were involved? What data became inconsistent?
2. **How you discovered it**: Customer complaint? Monitoring alert? Manual audit?
3. **Root cause**: Why did the inconsistency happen? (Dual-write problem, missing idempotency, out-of-order events, race condition)
4. **Immediate resolution**: How did you fix the inconsistent data?
5. **Structural fix**: What pattern did you adopt to prevent recurrence?
6. **What you'd do differently**: With hindsight, what would you change?

**Example outline (dual-write problem):**

"Our order service updated the order status in its database and then published an `OrderConfirmed` event to RabbitMQ. During a deployment, the service crashed between the database write and the event publish. The order was marked 'confirmed' in the order database, but the inventory service never received the event, so stock was never reserved. The customer received a confirmation email (triggered by the DB status change) but their order couldn't be fulfilled because inventory was never allocated.

We discovered it when the warehouse team reported orders without matching reservations. A manual audit found 23 orders over 2 weeks with this mismatch.

Root cause: classic dual-write problem (as described in question 8). Two separate operations -- DB write and event publish -- with no atomicity guarantee.

Immediate fix: We wrote a reconciliation script that compared orders in 'confirmed' status against inventory reservations and manually published the missing events for the 23 affected orders.

Structural fix: We implemented the transactional outbox pattern (question 9). Order confirmation now writes both the status change and the event to the same database in one transaction. A polling publisher reads the outbox and publishes to RabbitMQ. We also added idempotency keys on the inventory service side so replayed events don't double-reserve.

What I'd do differently: I'd add a reconciliation check from day one -- a scheduled job that compares order statuses with downstream service states and alerts on mismatches. The outbox pattern prevents new inconsistencies, but a reconciliation job catches edge cases we haven't thought of yet. Also, I'd have pushed for the outbox pattern before going to production with event-driven flows instead of treating it as a 'nice to have.'"

**Example outline (eventual consistency and user experience):**

"Our user profile service published `ProfileUpdated` events consumed by the notification preferences service. A user changed their email address, then immediately tried to update notification settings. The preferences service still had the old email because the event hadn't been processed yet. The user saw their old email in the preferences UI and thought the update had failed.

Root cause: eventual consistency lag between profile and preferences services, combined with no read-your-own-writes handling on the frontend.

Fix: We implemented optimistic UI updates -- after a profile change, the frontend uses the locally-known new value instead of re-fetching from the (potentially stale) read model. We also added a version header so the preferences service could detect stale reads and fall back to the profile service for fresh data when needed."

**Key points to hit:**
- Be specific about which consistency guarantee was violated (don't just say "data was wrong")
- Reference the relevant patterns by name (outbox, idempotency, saga) -- shows you know the vocabulary
- Show the business impact of the inconsistency (this is what makes it a real story, not a textbook exercise)
- "What I'd do differently" should show growth -- not "I'd do the same thing" but a genuine improvement

</details>
