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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How does Conway's Law shape microservice decomposition — why do team boundaries and communication structures directly influence service boundaries, what goes wrong when you design services that don't align with team structure, and how should you use this insight when splitting a monolith?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. How do you define service boundaries using Domain-Driven Design — what are bounded contexts, aggregates, and domain events, how do they map to microservice boundaries, and what's the difference between decomposing by business capability vs by subdomain?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. What is the distributed monolith anti-pattern and how do you detect it -- what are the telltale signs of bad decomposition (lock-step deploys, chatty cross-service calls, shared state that couples services), how does it differ from a well-designed microservice architecture, and how do you refactor your way out of one?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. When should microservices communicate synchronously (REST/gRPC) vs asynchronously (events/messages) — what are the tradeoffs in coupling, latency, and failure handling, how do you decide which communication style for which interaction, and why do most mature architectures use both?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How do sagas handle distributed transactions across microservices — what's the difference between choreography and orchestration, how do compensating transactions undo partial work when a step fails, and why must every step in a saga be retry-safe (idempotent)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How do circuit breakers, retries with exponential backoff, bulkheads, and timeout budgets compose together to prevent cascading failures across microservices — what order should you adopt these patterns, how do they interact (e.g., retries behind a circuit breaker vs in front of it), and what's the typical failure cascade that happens without them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do you implement eventual consistency safely across microservices -- why are idempotency keys essential for reliable message processing, what is the dual-write problem and why does writing to a database and publishing an event as two separate operations cause data loss, how does read-your-own-writes work when the read model lags behind the write model, and when should you abandon eventual consistency and require stronger guarantees?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. Why does the transactional outbox pattern exist and how does it solve the dual-write problem — how does writing events to a local outbox table in the same database transaction guarantee atomicity, what reads the outbox (polling vs CDC/change data capture), and what are the tradeoffs compared to alternatives like event sourcing or relying on the message broker's transactional support?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why does each microservice need its own database and what are the cross-service query strategies — what coupling does a shared database create, how do you handle queries that need data from multiple services (API composition, CQRS, data replication), and when is a shared database actually the pragmatic choice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How do you migrate from a monolith to microservices incrementally — how does the Strangler Fig pattern work (routing new traffic to the new service while the old code still handles existing flows), how does branch-by-abstraction let you replace internal implementations without changing callers, how does parallel run validate that the new service produces the same results as the old code, and what are the common mistakes in phased migrations?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What role does an API gateway play in a microservice architecture — how does it handle routing, authentication, and rate limiting, how does the BFF (Backend for Frontend) pattern differ from a general-purpose gateway, and when does the gateway become a single point of failure or a bottleneck?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What observability do you need across microservices — how do distributed tracing, correlation IDs, and structured logging work together for cross-service debugging, why is observability harder in microservices than monoliths, and what's the minimum viable observability setup?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. How do you achieve independent deployment in microservices — what API versioning strategies enable services to deploy independently, what makes a change backward-compatible vs breaking, what should trigger a rollback, and why is independent deployability the single most important microservice benefit?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Service Design & Communication

<details>
<summary>15. Design the service decomposition for an e-commerce system — identify the bounded contexts (orders, inventory, payments, shipping, users), define the API contracts between them, decide which communications are synchronous vs async, and explain why you drew the boundaries where you did</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Implement a saga for an order placement workflow that spans payment, inventory, and shipping services — show the event flow for choreography, implement compensating actions (refund on inventory failure), and demonstrate how to handle the failure case where the compensation itself fails</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Implement resilience patterns for a service that calls two downstream dependencies — show circuit breaker configuration (with a library like opossum), retry with exponential backoff, timeout budgets (if service A has 3s budget, downstream calls must share it), and bulkhead isolation to prevent one slow dependency from affecting the other</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Set up an API gateway that routes requests to multiple backend services -- show the routing configuration, implement authentication at the gateway layer, add rate limiting, and explain what happens when the gateway becomes a performance bottleneck or single point of failure and how to mitigate these risks</summary>

<!-- Answer will be added later -->

</details>

## Practical — Migration & Deployment

<details>
<summary>19. You suspect your microservice architecture has become a distributed monolith -- two services must always deploy together and one service makes 15+ synchronous calls to another per request. Walk through how you would diagnose the coupling (dependency mapping, deploy frequency analysis, call graph tracing), identify which boundary is wrong, and refactor toward proper service independence without a big-bang rewrite.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>20. Tell me about a time you decomposed a monolith into microservices (or decided not to) — what drove the decision, how did you identify service boundaries, and what challenges did you face during the migration?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>21. Describe a cascading failure you experienced in a microservice architecture — what triggered it, how did you diagnose the chain of failures, and what resilience patterns did you add to prevent recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>22. Describe a time you dealt with a data consistency issue across microservices — what was the scenario, how did you handle it (saga, eventual consistency, compensating action), and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>
