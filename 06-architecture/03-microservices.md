# Microservices

> **36 questions** — 18 theory, 18 practical

- Monolith vs microservices: honest tradeoffs and "monolith first"
- Conway's Law and how team structure shapes service decomposition
- Service boundaries using DDD: bounded contexts, aggregates, domain events
- Decomposition strategies: by business capability vs by subdomain
- Distributed monolith anti-pattern: signs of bad decomposition (lock-step deploys, chatty cross-service calls, shared state), how to detect and refactor
- Database-per-service pattern and cross-service query strategies, stateless share-nothing processes (twelve-factor)
- Synchronous vs asynchronous communication: REST/gRPC vs events/messages
- Sagas: choreography vs orchestration, compensating transactions, and retry safety
- Resilience patterns: circuit breakers, retries with backoff, bulkheads, timeout budgets -- composing them to prevent cascading failures
- Data consistency across services: eventual consistency tradeoffs, idempotency keys, read-your-own-writes, dual-write problems, transactional outbox pattern
- API gateways: routing, auth, rate limiting, BFF pattern, and single-point-of-failure risks
- Service discovery: client-side vs server-side, and how Kubernetes changes the equation
- Observability across services: distributed tracing, correlation IDs, and structured logging for cross-service debugging
- Migration patterns: Strangler Fig for incremental decomposition, branch-by-abstraction, parallel run for validation
- Testing microservices: contract tests (Pact), component tests, and the shifted testing pyramid
- Independent deployment: API versioning, backward-compatible changes, rollback triggers
- Shared libraries and code reuse: when to share vs duplicate, semantic versioning of internal packages, avoiding distributed monolith through shared code coupling

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
<summary>4. How do you choose between decomposing by business capability ('order management') vs by subdomain ('pricing') -- how do these strategies map to DDD bounded contexts, what signals help you pick one approach over the other, and what goes wrong when service boundaries cut across domain boundaries?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What is the distributed monolith anti-pattern and how do you detect it -- what are the telltale signs of bad decomposition (lock-step deploys, chatty cross-service calls, shared state that couples services), how does it differ from a well-designed microservice architecture, and how do you refactor your way out of one?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why does each microservice need its own database and what are the cross-service query strategies — what coupling does a shared database create, how do you handle queries that need data from multiple services (API composition, CQRS, data replication), and when is a shared database actually the pragmatic choice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. When should microservices communicate synchronously (REST/gRPC) vs asynchronously (events/messages) — what are the tradeoffs in coupling, latency, and failure handling, how do you decide which communication style for which interaction, and why do most mature architectures use both?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do sagas handle distributed transactions across microservices — what's the difference between choreography and orchestration, how do compensating transactions undo partial work when a step fails, and why must every step in a saga be retry-safe (idempotent)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How do circuit breakers, retries with exponential backoff, bulkheads, and timeout budgets compose together to prevent cascading failures across microservices — what order should you adopt these patterns, how do they interact (e.g., retries behind a circuit breaker vs in front of it), and what's the typical failure cascade that happens without them?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How do you implement eventual consistency safely across microservices -- why are idempotency keys essential for reliable message processing, what is the dual-write problem and why does writing to a database and publishing an event as two separate operations cause data loss, how does read-your-own-writes work when the read model lags behind the write model, and when should you abandon eventual consistency and require stronger guarantees?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why does the transactional outbox pattern exist and how does it solve the dual-write problem — how does writing events to a local outbox table in the same database transaction guarantee atomicity, what reads the outbox (polling vs CDC/change data capture), and what are the tradeoffs compared to alternatives like event sourcing or relying on the message broker's transactional support?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What role does an API gateway play in a microservice architecture — how does it handle routing, authentication, and rate limiting, how does the BFF (Backend for Frontend) pattern differ from a general-purpose gateway, and when does the gateway become a single point of failure or a bottleneck?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. How does service discovery work in microservices — what's the difference between client-side discovery (service registry) and server-side discovery (load balancer), how does Kubernetes change the equation with built-in DNS-based discovery, and when do you still need a service mesh?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. What observability do you need across microservices — how do distributed tracing, correlation IDs, and structured logging work together for cross-service debugging, why is observability harder in microservices than monoliths, and what's the minimum viable observability setup?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. How do you migrate from a monolith to microservices incrementally -- how does the Strangler Fig pattern work (routing new traffic to the new service while the old code still handles existing flows), how does branch-by-abstraction let you replace internal implementations without changing callers, how does parallel run validate that the new service produces the same results as the old code, and what are the common mistakes in phased migrations?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. How does the testing pyramid shift for microservices — why do contract tests (Pact) become essential, what do component tests cover that unit and integration tests don't, and why does E2E testing across all services become prohibitively expensive?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How do you achieve independent deployment in microservices — what API versioning strategies enable services to deploy independently, what makes a change backward-compatible vs breaking, what should trigger a rollback, and why is independent deployability the single most important microservice benefit?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. When should microservices share code vs duplicate it — what are the risks of shared libraries (version coupling, coordinated releases, diamond dependency problems), how do you version shared packages safely, and when is copying code between services the better choice?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Service Design & Communication

<details>
<summary>19. Design the service decomposition for an e-commerce system — identify the bounded contexts (orders, inventory, payments, shipping, users), define the API contracts between them, decide which communications are synchronous vs async, and explain why you drew the boundaries where you did</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Implement a saga for an order placement workflow that spans payment, inventory, and shipping services — show the event flow for choreography, implement compensating actions (refund on inventory failure), and demonstrate how to handle the failure case where the compensation itself fails</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Implement resilience patterns for a service that calls two downstream dependencies — show circuit breaker configuration (with a library like opossum), retry with exponential backoff, timeout budgets (if service A has 3s budget, downstream calls must share it), and bulkhead isolation to prevent one slow dependency from affecting the other</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Implement cross-service data querying without a shared database — show the API composition pattern (service A queries B and C then merges results), compare with a CQRS approach (build a denormalized read model from events), and explain the consistency tradeoffs of each</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Set up an API gateway that routes requests to multiple backend services -- show the routing configuration, implement authentication at the gateway layer, add rate limiting, and explain what happens when the gateway becomes a performance bottleneck or single point of failure and how to mitigate these risks</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Implement the BFF (Backend for Frontend) pattern where a mobile-specific gateway aggregates multiple service calls into a single response -- show the code for the aggregation layer, explain how BFF differs from a general-purpose API gateway, and when does maintaining multiple BFF gateways become more expensive than the problem it solves?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Migration & Deployment

<details>
<summary>25. Implement the Strangler Fig pattern to extract a service from a monolith — show the proxy/router that gradually redirects traffic from the monolith to the new service, how to run both in parallel during migration, and how to handle the database split (shared DB phase → separate DB)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Set up independent deployment with API versioning — show how to version an API (URL path versioning or header versioning), implement a backward-compatible change (adding a field) and a breaking change (removing a field) with proper deprecation, and define the rollback criteria and process</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. You suspect your microservice architecture has become a distributed monolith -- two services must always deploy together and one service makes 15+ synchronous calls to another per request. Walk through how you would diagnose the coupling (dependency mapping, deploy frequency analysis, call graph tracing), identify which boundary is wrong, and refactor toward proper service independence without a big-bang rewrite.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Testing & Observability

<details>
<summary>28. Set up contract testing with Pact between two microservices — show the consumer test (defining the expected interaction), the provider verification test, and demonstrate what happens when the provider makes a breaking change. How do you integrate this into CI?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Implement distributed tracing and correlation IDs across three services in a request chain — show how the first service generates a correlation ID, how it propagates through HTTP headers and message headers, how each service includes it in structured logs, and how you use the trace to debug a slow request</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Write a component test for a microservice that verifies it handles incoming requests, interacts with its database, and publishes events correctly — show the test setup (real database, mocked external services), the test itself, and explain how component tests differ from integration tests and unit tests</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. A request that spans three services is intermittently timing out — walk through the debugging process using distributed traces: identify which service is slow, check if it's a cascading failure (one slow service backing up the others), determine if the circuit breaker should have tripped, and apply the fix</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Design a shared library strategy for common code across 5 microservices — decide what goes in the shared library vs what gets duplicated, show the versioning approach (semver, independent releases), demonstrate how a breaking change in the shared library doesn't force all services to deploy simultaneously, and explain the alternative of code generation</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>33. Tell me about a time you decomposed a monolith into microservices (or decided not to) — what drove the decision, how did you identify service boundaries, and what challenges did you face during the migration?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a cascading failure you experienced in a microservice architecture — what triggered it, how did you diagnose the chain of failures, and what resilience patterns did you add to prevent recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>35. Tell me about a time you had to debug a problem that spanned multiple microservices — what tools and techniques did you use, how long did it take, and what observability improvements did you make afterward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Describe a time you dealt with a data consistency issue across microservices — what was the scenario, how did you handle it (saga, eventual consistency, compensating action), and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>
