# Scaling & Reliability

> **36 questions** — 17 theory, 19 practical

- Horizontal vs vertical scaling — tradeoffs, hidden costs of horizontal (state management, coordination, consistency, DB connection exhaustion, cache stampede), when vertical wins
- Stateless service design — shared-nothing architecture, externalizing session state to Redis/database, why statelessness is a prerequisite for horizontal scaling
- Database scaling — read replicas (routing reads vs writes, replication lag handling), partitioning, sharding as last resort
- Multi-region scaling — active-active vs active-passive, latency-based routing, regional failover, data replication challenges across regions
- Capacity planning — traffic estimation, identifying scaling triggers (CPU, queue depth, latency), headroom planning, when to scale proactively vs reactively
- Reliability as a design concern — redundancy (no single points of failure), fault isolation (blast radius containment, bulkhead boundaries), designing for failure from day one vs bolting it on
- Graceful degradation — feature flags, fallback responses, read-only mode, queue shedding, fallback chains, per-dependency timeout and circuit breaker placement
- Resilience patterns in practice — how circuit breaker, retry (with backoff/jitter), and bulkhead compose in real services; retry amplification across layers; retry budgets; when patterns hurt more than help
- Backpressure vs rate limiting — upstream signal propagation, load shedding during saturation, queue depth as backpressure signal
- Load balancing in practice — connection vs request-level, sticky sessions breaking horizontal scale, health-check-driven removal, uneven load distribution
- Deployment reliability — graceful shutdown (SIGTERM handling, connection draining), readiness signaling, zero-downtime checklist, common rollout failure modes (502 spikes, premature termination)
- Advanced deployment strategies — canary deployments (progressive traffic shifting, metric-based rollback), blue-green deployments, feature flags for decoupling deploy from release
- Chaos engineering basics — failure injection, steady-state hypothesis, blast radius limits, when it's premature
- Strangler Fig for scaling — incremental migration of bottleneck components, routing layer for traffic splitting, feature-by-feature extraction to improve scalability without big-bang rewrites

---

## Foundational

<details>
<summary>1. How do scaling and reliability relate to each other as engineering concerns — why doesn't simply adding more instances automatically make a system more reliable, what new failure modes does horizontal scaling introduce, and how should a senior engineer think about the relationship between capacity, redundancy, and fault tolerance when designing a system?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What are the real tradeoffs between horizontal and vertical scaling — beyond the textbook answer of "vertical has limits," what are the hidden costs of horizontal scaling (state management, coordination overhead, consistency challenges, database connection exhaustion, cache stampedes), and when does vertical scaling actually win over horizontal in practice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Why should reliability be treated as a design concern rather than an afterthought — how do redundancy, fault isolation, graceful degradation, and timeouts work together as a reliability strategy, and what happens to systems that bolt these on later instead of designing for them from the start?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. How do read replicas work for database scaling — how do you route reads vs writes correctly, what problems does replication lag introduce (stale reads, read-your-own-write violations), and what strategies exist to handle lag without degrading the user experience?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. What is the difference between partitioning and sharding for database scaling, why is sharding considered a last resort, what operational complexity does it introduce (cross-shard queries, rebalancing, application-level routing), and what should you exhaust before reaching for sharding?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What does graceful degradation look like in practice — how do feature flags, fallback responses, read-only mode, and queue shedding each contribute to keeping a system usable under partial failure, and how do you decide which features to shed first when things start breaking?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. Why should circuit breakers and timeouts be configured per-dependency rather than globally — how do fallback chains work when a primary dependency fails, what does proper per-dependency isolation look like in a service that calls multiple downstream APIs, and what goes wrong when teams use a single global timeout for all outbound calls?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do circuit breaker, retry with exponential backoff and jitter, and bulkhead patterns compose together in a real service — why do you need all three rather than just one, how does each pattern address a different failure mode, and what does the interaction between them look like when a downstream dependency degrades?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What is retry amplification and why does it cause cascading failures across service layers — how does a retry at one layer multiply into exponential load on downstream services, what are retry budgets, and how do you implement them to cap the total retry volume across an entire call chain?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. When do resilience patterns like circuit breakers, retries, and bulkheads hurt more than they help — what are the signs that a team has over-applied these patterns, what failure modes do misconfigured resilience patterns introduce (masking real issues, delaying detection, adding latency), and how do you decide which patterns a given service actually needs?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How does backpressure differ from rate limiting as a strategy for handling overload — what does upstream signal propagation mean in practice, when should a system shed load vs propagate backpressure, how does queue depth serve as a backpressure signal, and what happens when a system has rate limiting but no backpressure mechanism?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What are the practical differences between connection-level and request-level load balancing — why do sticky sessions break horizontal scaling, how do health-check-driven removal and reinsertion work, and what causes uneven load distribution even when you're using a "balanced" algorithm?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What is chaos engineering, why does it require a steady-state hypothesis before injecting failures, what does blast radius control mean in practice, and when is chaos engineering premature — what prerequisites should a team have in place before it's worth running failure experiments?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. When and why should you use the Strangler Fig pattern to improve a system's scalability or reliability — how does the routing layer work to incrementally shift traffic from the old component to the new one, what makes feature-by-feature extraction safer than a big-bang rewrite, and what are the risks of running the old and new implementations in parallel during the migration?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Why is stateless service design a prerequisite for horizontal scaling -- what does shared-nothing architecture mean in practice, where should session state be externalized (Redis, database, signed tokens), what are the tradeoffs between these options, and what subtle statefulness (in-memory caches, local file writes, sticky sessions) do teams miss when they think their services are stateless?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. What are the tradeoffs between active-active and active-passive multi-region architectures -- how does latency-based routing work, what makes regional failover harder than it sounds, and what data replication challenges (conflict resolution, eventual consistency, write routing) make active-active significantly more complex than active-passive?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How do you approach capacity planning for a growing service -- how do you estimate future traffic, what metrics should trigger scaling decisions (CPU, queue depth, latency percentiles), how much headroom should you maintain, and when should you scale proactively vs reactively?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Implementation & Configuration

<details>
<summary>18. Implement a resilience layer for a Node.js/TypeScript service that calls two downstream APIs — show how you'd compose circuit breaker, retry with exponential backoff and jitter, and bulkhead isolation using a library like cockatiel or opossum, explain the configuration choices (thresholds, timeouts, max concurrent calls), and show what happens in the code path when the circuit opens</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Show how you'd implement graceful degradation in a Node.js service that depends on a recommendation engine and a payment service — demonstrate fallback responses when the recommendation engine is down (cached/default recommendations), feature flag-driven degradation to read-only mode, and explain how you'd prioritize which capabilities to shed under increasing failure</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Implement a backpressure mechanism in a Node.js service that consumes from a message queue — show how you'd use queue depth as a signal to slow down consumption, how to propagate backpressure upstream (e.g., returning HTTP 429 or pausing consumers), and what load shedding looks like when the queue hits a critical depth threshold</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Configure request-level load balancing with proper health checks for a set of backend instances — show the configuration (e.g., Nginx or cloud LB), explain how health-check-driven removal works, how to tune check intervals and thresholds to avoid flapping, and what configuration prevents uneven load distribution caused by slow instances or connection reuse</summary>

<!-- Answer will be added later -->

</details>

## Practical — Database Scaling

<details>
<summary>22. Set up read replica routing in a Node.js/TypeScript application — show how you'd configure the database client to send writes to the primary and reads to replicas, handle the replication lag problem for read-after-write scenarios (e.g., user creates a record then immediately views it), and explain the tradeoffs between routing strategies (sticky connections, primary fallback, causal consistency tokens)</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. A service scaled to 50 instances is exhausting database connections — walk through how you'd diagnose this, show how to configure connection pooling (PgBouncer or application-level pooling), calculate the right pool size given instance count and database connection limits, and explain the tradeoffs between transaction-level and session-level pooling modes</summary>

<!-- Answer will be added later -->

</details>

## Practical — Deployment Reliability

<details>
<summary>24. Configure a zero-downtime deployment for a Node.js service running in Kubernetes — show the Deployment spec with rolling update strategy, the application-level SIGTERM handler that stops accepting new requests and drains in-flight connections, the preStop hook, and proper terminationGracePeriodSeconds. Walk through the full shutdown sequence and explain what happens to requests in transit at each step</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. A rolling deployment is causing intermittent 502 errors — walk through the diagnosis process: why do 502s happen during rollouts (premature pod termination before connections drain, readiness probe misconfiguration, load balancer not updated), how you'd identify which step in the rollout sequence is causing the issue, and what the specific fixes are for each root cause</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Configure readiness probes for a Node.js service that depends on a database and a Redis cache — show how the readiness check verifies downstream dependencies are reachable, explain why readiness probes should check dependency health vs just returning 200, what happens when the probe fails (traffic removed but pod stays running), and how misconfigured readiness probes can cause cascading failures when a shared dependency goes down</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. How do canary and blue-green deployments work, and why would you choose one over a standard rolling update -- walk through progressive traffic shifting in a canary deployment, what metrics should trigger automatic rollback, how do blue-green deployments enable instant rollback, and how do feature flags let you decouple the act of deploying code from the act of releasing a feature to users?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>28. After scaling a service from 3 to 20 instances, you observe a sudden spike of identical requests hitting the database — diagnose this as a cache stampede, explain why it happens specifically during horizontal scale-out (cold caches on new instances, synchronized TTL expiry), and show the concrete fixes: request coalescing, staggered TTLs, cache warming on startup, and probabilistic early recomputation</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Service A calls Service B, which calls Service C. Service C starts responding slowly (2s instead of 200ms). Within minutes, all three services are failing — walk through exactly how retry amplification caused this cascade: how retries at each layer multiplied the load, why timeouts were configured incorrectly, and show the specific changes to retry budgets, per-layer timeouts, and circuit breaker thresholds that would have prevented this</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Your monitoring shows that 3 out of 10 instances are handling 70% of the traffic while the others are nearly idle — diagnose the root causes of this uneven load distribution (long-lived connections, connection-level vs request-level balancing, health check gaps, slow instances not being removed), and show the specific configuration changes that fix each cause</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. A message queue's depth is growing rapidly and consumer lag is increasing — walk through how you'd diagnose whether this is a backpressure problem (consumers can't keep up) vs a traffic spike (producers sending more than normal), what immediate load shedding steps you'd take to prevent the queue from overwhelming downstream services, and how you'd set up monitoring and alerts to catch this earlier next time</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Design and run a basic chaos engineering experiment for a service that depends on a third-party payment API — define the steady-state hypothesis, choose a failure to inject (latency injection, error responses, total unavailability), set blast radius limits, show how you'd run the experiment safely in a staging environment, and explain what you'd do if the experiment reveals the service doesn't degrade gracefully</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>33. Tell me about a time you had to scale a system to handle significantly more load than it was designed for — what was the trigger, what scaling approach did you choose (horizontal, vertical, architectural changes), what unexpected problems did you hit during the scaling process, and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you debugged a cascading failure in a distributed system — what were the symptoms, how did you trace the root cause across services, what was the fix, and what resilience patterns did you put in place afterward to prevent recurrence?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>35. Tell me about a time you implemented graceful degradation for a production service — what dependencies were involved, how did you decide what to degrade and in what order, and how did the degradation strategy perform when it was actually needed?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Describe a time a deployment caused a production incident — what went wrong (downtime, errors, data issues), how did you recover, and what changes to your deployment process or reliability checklist did you make to prevent it from happening again?</summary>

<!-- Answer framework will be added later -->

</details>
