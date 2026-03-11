# Event-Driven Architecture

> **34 questions** — 17 theory, 17 practical

- Event-driven architecture vs synchronous request-response: decoupling benefits, eventual consistency tradeoffs, debugging complexity
- Core building blocks: events, commands, producers, consumers, brokers, topics/queues
- Eventual consistency: when to tolerate it vs when to require strong consistency
- Event ordering: partition-level ordering, handling out-of-order events, version-based conflict resolution, when ordering matters vs when it doesn't
- Event sourcing: storing events vs current state, event store vs message broker (different infrastructure for different purposes), event schema versioning and upcasting, snapshot optimization, replay complexity
- CQRS: separating reads from writes, read model projections, and when to avoid it
- Idempotent consumers: idempotency keys, deduplication windows, transactional processing, dead letter queues as a recovery mechanism
- Transactional outbox pattern for solving the dual-write problem
- Sagas: choreography vs orchestration, choreography's hidden coupling, compensating actions, and failure handling
- Fan-out and event routing: topic-per-event-type vs coarse topics, content-based filtering, multiple consumer groups
- Backpressure: what happens when producers outpace consumers, buffering vs dropping vs rate-limiting strategies, consumer scaling
- Multi-stage event pipelines: chaining processors, intermediate topics, and pipeline failure handling
- Testing event-driven systems: async flow verification, contract testing between producers/consumers, integration tests with embedded brokers
- Observability: consumer lag, processing latency, DLQ depth, distributed tracing across async flows
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
<summary>10. How should you design event routing and fan-out — what are the tradeoffs of topic-per-event-type vs coarse-grained topics, how does content-based filtering work, and how do multiple consumer groups enable different services to independently process the same events?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How do multi-stage event pipelines work — why would you chain processors through intermediate topics instead of handling everything in one consumer, how do you handle pipeline failures (retry, skip, dead letter), and what happens when a middle stage fails?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. How do you test event-driven systems — what makes async flow verification harder than testing synchronous APIs, how do you write contract tests between producers and consumers, and when should you use embedded brokers vs mocks in integration tests?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What observability metrics do event-driven systems need — what do consumer lag, processing latency, and DLQ depth each tell you about system health, how do these metrics degrade as specific problems emerge, and what alert thresholds should you set for each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Why is distributed tracing harder in event-driven systems than in synchronous request-response — how does trace context propagation work through message headers, what breaks when you lose the trace context, and how does the resulting trace visualization differ from a synchronous call chain?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. When should you NOT use event-driven architecture — what signals indicate a team is over-engineering with events when synchronous calls would suffice, what's the minimum scale or complexity that justifies the operational overhead, and what are the common failure modes of premature event-driven adoption?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. How does event ordering work at the partition level, and when does ordering actually matter vs when can you safely ignore it — how do you handle out-of-order events when they arrive (version-based conflict resolution, last-write-wins, causal ordering), and what design decisions determine whether your system needs strict ordering or can tolerate reordering?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. What happens when producers outpace consumers in an event-driven system and how do you design for backpressure — what are the tradeoffs between buffering (letting the broker absorb the spike), dropping messages (load shedding), and rate-limiting producers, how do you decide which strategy fits your use case, and when should you scale consumers horizontally vs vertically?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Event Patterns & Implementation

<details>
<summary>18. Implement an idempotent event consumer in Node.js — show the consumer that processes an order-created event, uses an idempotency key to prevent duplicate processing, wraps the work in a database transaction, and routes failures to a dead letter queue. Explain what happens when the same event is delivered twice</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Implement the transactional outbox pattern — show the code that writes a domain change and an outbox record in a single database transaction, then a separate process that reads the outbox and publishes events to the broker. Compare the polling approach vs CDC (Change Data Capture) for reading the outbox</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Implement a saga using choreography for an order fulfillment workflow (order → payment → inventory → shipping) — show the events each service publishes and consumes, implement a compensating action (refund payment when inventory fails), and demonstrate the failure flow when a middle step fails</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Implement a saga using orchestration for the same order workflow — show the orchestrator that directs each step, handles responses, and triggers compensation on failure. Compare this with the choreography version: which is easier to understand, debug, and extend?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Implement CQRS with a read model projection — show a write side that publishes events on state changes, a projection that listens to events and builds a denormalized read model, and the query handler that reads from the projection. Demonstrate the eventual consistency gap and how to handle it from the client's perspective</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Design event routing for a system with multiple consumers — show topic-per-event-type design, fan-out to multiple consumer groups (order events consumed by shipping, analytics, and notifications independently), and implement content-based filtering so a consumer only processes events matching specific criteria. Explain what problems arise if you use a single coarse-grained topic for all events instead</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Build a multi-stage event pipeline (e.g., ingest → validate → enrich → store) using intermediate topics — show the pipeline with error handling at each stage, demonstrate what happens when the enrichment stage fails (dead letter, retry, skip), and how to replay failed messages through the pipeline</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Implement event schema evolution for an event-sourced system — show how to version event types, handle backward-compatible changes (adding optional fields) vs breaking changes (renaming fields), and demonstrate how a consumer handles both v1 and v2 of an event during the migration period</summary>

<!-- Answer will be added later -->

</details>

## Practical — Testing & Observability

<details>
<summary>26. Write integration tests for an event-driven flow using an embedded or containerized broker — show a test that publishes an event, waits for the consumer to process it, and asserts on the resulting state. Demonstrate how to avoid flaky tests caused by async timing without using hardcoded sleeps</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Implement distributed tracing across an async event flow — show how to propagate trace context through message headers from producer to consumer, how to create child spans in the consumer, and how the resulting trace visualization helps debug a slow or failing multi-service flow</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Set up monitoring for an event-driven system — show the Prometheus metrics or instrumentation code for consumer lag, processing latency (time from publish to process), and DLQ depth, define alert thresholds for each metric, and explain what each metric tells you about system health and what symptoms indicate each is degrading</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Implement contract testing between an event producer and consumer — show how to define the event schema contract, write a producer test that verifies it publishes conforming events, and a consumer test that verifies it can handle the contract. What happens when the producer wants to evolve the schema?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. A consumer is processing messages increasingly slowly and the consumer lag is growing — walk through the diagnosis: check if it's a slow downstream dependency, database contention, message processing errors causing retries, or insufficient consumer instances. Show the metrics and commands you'd use to identify the bottleneck</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>31. Tell me about a time you introduced event-driven architecture to a system — what problem were you solving, what did you choose (event sourcing, CQRS, simple pub/sub), and what unexpected challenges did you encounter?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>32. Describe a time you debugged a complex issue in an event-driven system — how did you trace the problem across services, what tools did you use, and what observability improvements did you add afterward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>33. Tell me about a time you dealt with message ordering, duplication, or data consistency issues in an async system — what was the problem, how did you solve it, and what patterns did you adopt to prevent recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you had to choose between choreography and orchestration for a distributed workflow — what were the requirements, what did you choose, and looking back, was it the right call?</summary>

<!-- Answer framework will be added later -->

</details>
