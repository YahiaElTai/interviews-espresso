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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Why do event-driven systems need distinct building blocks (events vs commands, producers vs consumers, brokers, topics vs queues) — what role does each play, why is the distinction between events and commands important for system design, and how do topics and queues differ in how they route messages?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. When should you tolerate eventual consistency vs require strong consistency — what types of operations can safely be eventually consistent (notifications, analytics, search indexing), what must be strongly consistent (financial transactions, inventory), and how do you communicate consistency boundaries to the rest of the team?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. Why would you store events instead of current state in an event-sourced system — what benefits does this provide (complete audit trail, replay, temporal queries), what makes schema evolution hard (versioned event types, upcasting), how do snapshots optimize replay performance as event volume grows, and what replay complexity issues remain even with snapshots?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why is an event store (like EventStoreDB or a custom append-only table) a different piece of infrastructure from a message broker (like Kafka or RabbitMQ) — what problem does each solve, when do you need both in an event-sourced system, and when is a message broker alone sufficient without a dedicated event store?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why would you separate read and write models using CQRS — how do read model projections work, what consistency challenges arise between the write side and read side, and when is CQRS unnecessary complexity for your use case?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why must consumers in event-driven systems be idempotent — how do idempotency keys and deduplication windows work, what does transactional processing (consuming + writing in one transaction) guarantee, and how do dead letter queues serve as a recovery mechanism for messages that repeatedly fail?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What is the dual-write problem and how does the transactional outbox pattern solve it — why can't you reliably write to a database and publish an event in two separate operations, how does the outbox pattern make this atomic, and what are the approaches for reading from the outbox (polling, CDC)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How do sagas coordinate multi-step distributed workflows — what's the difference between choreography (each service reacts to events) and orchestration (central coordinator directs steps), why does choreography create hidden coupling as the workflow grows, how do compensating actions handle failures, and what makes saga failure handling harder than transaction rollback?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. When should you NOT use event-driven architecture — what signals indicate a team is over-engineering with events when synchronous calls would suffice, what's the minimum scale or complexity that justifies the operational overhead, and what are the common failure modes of premature event-driven adoption?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How does event ordering work at the partition level, and when does ordering actually matter vs when can you safely ignore it — how do you handle out-of-order events when they arrive (version-based conflict resolution, last-write-wins, causal ordering), and what design decisions determine whether your system needs strict ordering or can tolerate reordering?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Event Patterns & Implementation

<details>
<summary>12. Implement an idempotent event consumer in Node.js — show the consumer that processes an order-created event, uses an idempotency key to prevent duplicate processing, wraps the work in a database transaction, and routes failures to a dead letter queue. Explain what happens when the same event is delivered twice</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Implement the transactional outbox pattern — show the code that writes a domain change and an outbox record in a single database transaction, then a separate process that reads the outbox and publishes events to the broker. Compare the polling approach vs CDC (Change Data Capture) for reading the outbox</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Implement a saga using choreography for an order fulfillment workflow (order → payment → inventory → shipping) — show the events each service publishes and consumes, implement a compensating action (refund payment when inventory fails), and demonstrate the failure flow when a middle step fails</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Implement CQRS with a read model projection — show a write side that publishes events on state changes, a projection that listens to events and builds a denormalized read model, and the query handler that reads from the projection. Demonstrate the eventual consistency gap and how to handle it from the client's perspective</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Implement event schema evolution for an event-sourced system — show how to version event types, handle backward-compatible changes (adding optional fields) vs breaking changes (renaming fields), and demonstrate how a consumer handles both v1 and v2 of an event during the migration period</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>17. Tell me about a time you introduced event-driven architecture to a system — what problem were you solving, what did you choose (event sourcing, CQRS, simple pub/sub), and what unexpected challenges did you encounter?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>18. Describe a time you debugged a complex issue in an event-driven system — how did you trace the problem across services, what tools did you use, and what observability improvements did you add afterward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>19. Tell me about a time you dealt with message ordering, duplication, or data consistency issues in an async system — what was the problem, how did you solve it, and what patterns did you adopt to prevent recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>20. Describe a time you had to choose between choreography and orchestration for a distributed workflow — what were the requirements, what did you choose, and looking back, was it the right call?</summary>

<!-- Answer framework will be added later -->

</details>
