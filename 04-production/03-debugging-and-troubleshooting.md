# Debugging & Troubleshooting

> **38 questions** — 19 theory, 19 practical

- Why distributed debugging is harder — partial failures, non-deterministic ordering, clock skew, the failing service isn't always the broken service
- Systematic debugging methodology — hypothesize, instrument, test, narrow; mental models (bisection, working backward, fanning out); cognitive traps to avoid; why observability alone doesn't answer "why"
- Production vs staging debugging — why bugs don't reproduce locally (concurrency, load, config drift, data volume), debug logging, feature flags, traffic mirroring, canary analysis
- Log analysis for debugging — structured log queries (Kibana/Loki/CloudWatch), filtering and correlation patterns, extracting signal from high-volume logs, improving error messages for debuggability
- Correlation IDs vs distributed tracing — propagation across HTTP, queues, background jobs, gap handling
- Error tracking and aggregation — grouping exceptions, linking errors to deploys, sampling strategies, Sentry as a practical example
- Node.js memory leak detection — common patterns (closures, event listeners, caches, globals), heap snapshots, Chrome DevTools analysis, programmatic memory monitoring (process.memoryUsage(), v8.getHeapStatistics()) for trend detection and health checks
- CPU profiling and flame graphs — on-CPU vs off-CPU, reading flame graphs, identifying event loop blocking vs slow I/O, common Node.js bottlenecks (JSON serialization, regex, sync computation)
- Kubernetes debugging — CrashLoopBackOff (exit codes, ephemeral containers), OOMKilled (cgroup limits), networking failures (DNS, endpoints, network policies), rollout failures (probes, missing configs, image pull)
- Database debugging — slow query identification (pg_stat_statements, EXPLAIN ANALYZE), lock contention diagnosis
- Silent failure debugging — event-driven pipelines where no component errors (dead letter queues, message shape mismatches, idempotency key collisions), detecting data loss in async flows, synthetic event tracing
- Cache inconsistency debugging — stale data across layers (CDN, Redis, app cache, read replicas), identifying which cache layer is stale, cache key collisions, TTL mismatches, replication lag vs cache staleness
- Timeout chain debugging — tracing timeout budgets across service hops, identifying retry amplification, distinguishing upstream timeouts from downstream slowness
- Peak-hour error diagnosis — connection pool exhaustion, downstream degradation, resource limits, rate limiting
- Incident response — severity levels (SEV1-SEV4), first 15 minutes triage, mitigation vs fix, escalation, communication during outages
- Blameless postmortems — timeline format, root cause analysis, contributing factors, actionable items with owners

---

## Foundational

<details>
<summary>1. Why is debugging distributed systems fundamentally harder than debugging a single service — what specific properties of distributed systems (partial failures, non-deterministic ordering, clock skew, misleading error locations) make traditional debugging approaches break down, and why is the service reporting the error often not the service that's actually broken?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What does a systematic debugging methodology look like for production issues -- how do the mental models of bisection, working backward from symptoms, and fanning out across dependencies guide your investigation differently, when do you apply each one, and why does jumping straight to "check the logs" without a hypothesis waste time?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What cognitive traps do engineers fall into when debugging under pressure -- how do confirmation bias, anchoring on recent changes, and premature narrowing lead you down wrong paths, and why does having dashboards and logs (observability) still not answer the question "why is this happening" without a structured investigation?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. Why do production bugs frequently fail to reproduce in staging or local environments — what specific factors (concurrency under load, config drift, data volume and shape, race conditions that only appear at scale) cause this gap, and what strategies (debug logging, feature flags, traffic mirroring, canary analysis) bridge it without requiring full production access?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How should log levels be used in practice to maintain a high signal-to-noise ratio — what are the common mistakes teams make with log levels (logging everything at INFO, not distinguishing WARN from ERROR, missing context in messages), why do generic error messages like "something went wrong" actively harm debugging, and what makes a structured log entry actually useful when you're searching across thousands of requests?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. What is the difference between correlation IDs and distributed tracing, why do you need both in a microservices architecture, and what does each one tell you during debugging that the other cannot?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How does trace context propagate across HTTP calls, message queues, and background jobs -- what mechanisms carry the context in each transport, and what happens when propagation breaks (a service doesn't forward the header, a queue consumer starts a new context) -- how do you detect and handle those gaps?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why is error tracking and aggregation (tools like Sentry) valuable beyond what logs and metrics provide -- how does intelligent exception grouping work to reduce noise, why is linking error spikes to specific deploys critical for fast diagnosis, and what would you miss without a dedicated error tracking tool?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What sampling strategies does error tracking use to prevent cost explosion without losing visibility into rare but important errors -- how do you configure sampling so high-volume low-severity errors don't blow your event quota while rare critical errors are always captured, and what are the tradeoffs of client-side vs server-side error tracking?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. What are the most common Node.js memory leak patterns — why do closures that capture large scopes, forgotten event listeners, unbounded in-memory caches, and accidental globals cause leaks, why are these leaks particularly insidious in long-running server processes compared to request-response scripts, what signals (heap growth over time, increased GC pauses, eventual OOM) indicate a leak is present, and how do process.memoryUsage() and v8.getHeapStatistics() help you detect memory trends programmatically (e.g., in health checks or custom monitoring) before a leak escalates to an OOM kill?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How do flame graphs work for CPU profiling and what do they reveal that other profiling approaches don't — what is the difference between on-CPU and off-CPU flame graphs, how do you read them to distinguish event loop blocking from slow I/O waits, and what are the most common Node.js CPU bottlenecks (JSON serialization of large payloads, catastrophic regex backtracking, synchronous computation in the main thread)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. How do you approach database debugging when queries are slow or transactions are blocking each other — what does pg_stat_statements tell you that application-level metrics don't, how does EXPLAIN ANALYZE reveal the actual execution plan vs what the planner estimated, and how do you diagnose lock contention (identifying blocking queries, understanding lock types, determining whether the issue is application design or database configuration)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why are silent failures in event-driven systems particularly dangerous and difficult to debug — what happens when messages are dropped, consumers fail silently, or a dead letter queue fills up without alerting, and why does the fact that no single component reports an error make these failures invisible until a downstream effect (missing data, stale state, customer complaint) surfaces much later?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. How should incident response be structured — what do severity levels (SEV1 through SEV4) represent and who decides the severity, what should the first 15 minutes of a SEV1 incident look like (triage, roles, communication), why is mitigation (stop the bleeding) more important than root-cause fixing during an active incident, and how do escalation and stakeholder communication work without creating chaos?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. What makes a blameless postmortem effective vs a checkbox exercise — why does the timeline format matter, how do you distinguish root cause from contributing factors, what makes action items actually get completed (owners, deadlines, tracking), and how do you build a culture where postmortems lead to systemic improvements rather than finger-pointing or filing tickets that never get prioritized?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Why are caching inconsistency bugs so difficult to reason about — when you have multiple cache layers (CDN, Redis, application-level cache) plus database read replicas, how does stale data propagate through these layers, why can invalidating one layer still leave stale data in another, and what mental model helps you systematically trace which layer is serving the wrong data?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. What causes systems to fail specifically during peak hours when they work fine the rest of the day — how do connection pool exhaustion, downstream service degradation, resource limit hits, and rate limiting interact to create cascading failures, and why do these issues often not appear in load testing?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. How do you debug timeout chains across multiple service hops -- how do timeout budgets propagate (or fail to propagate) through a call chain, why does retry amplification turn a single slow service into a cascading failure, and how do you distinguish whether the problem is an upstream timeout set too low or a downstream service that's genuinely slow?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. What are the common categories of Kubernetes networking failures and why are they hard to debug — how do DNS resolution failures, endpoint registration delays, network policy misconfigurations, and kube-proxy/iptables rule issues each manifest differently, and why does "the pod is running but can't be reached" have so many possible root causes?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging Workflows & Tooling

<details>
<summary>20. Walk through how you would use structured log queries in Kibana, Loki, or CloudWatch Logs Insights to isolate the root cause of a failing request that spans three services — show example query syntax for filtering by correlation ID, time range, and error level, demonstrate how you narrow from thousands of log lines to the relevant handful, and explain what filtering patterns (exclude health checks, focus on a single trace, aggregate by error type) you apply first.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Show how to implement correlation ID propagation in a Node.js/TypeScript microservices setup that spans HTTP calls, message queue consumers, and background jobs — demonstrate the middleware that generates or extracts the ID, how it gets passed to downstream HTTP calls and queue message headers, how async context (AsyncLocalStorage) keeps it available without threading it through every function parameter, and what breaks if a single service in the chain doesn't propagate it.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Walk through the process of detecting and diagnosing a Node.js memory leak using heap snapshots — show the exact steps to take a heap snapshot in production (or a production-like environment), how to load it into Chrome DevTools, how to compare two snapshots taken minutes apart to identify growing objects, and how to trace a retained object back to the code that's leaking (a closure, an event listener, an unbounded Map). What do you look for in the "Retained Size" vs "Shallow Size" columns?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Show how to generate and read a flame graph for a Node.js application that has high CPU usage — walk through the profiling commands (clinic flame, 0x, or --prof), explain how to interpret the resulting flame graph to identify whether the bottleneck is JSON.parse/stringify on large payloads, a regex with catastrophic backtracking, synchronous file I/O, or heavy computation in the event loop, and demonstrate the difference between what an on-CPU flame graph shows vs what you'd see if the issue were slow I/O (off-CPU).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Given a PostgreSQL database where users report slow page loads, walk through the exact steps to identify the problematic queries using pg_stat_statements (which columns matter, how to sort for impact), then take the worst query and run EXPLAIN ANALYZE — show how to read the output to identify sequential scans, bad join strategies, or row estimate mismatches, and demonstrate how to diagnose lock contention by querying pg_locks and pg_stat_activity to find blocking transactions.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Infrastructure & System Debugging

<details>
<summary>25. A pod is stuck in CrashLoopBackOff — walk through the exact kubectl commands to diagnose the root cause step by step: checking pod status and restart count, reading events, pulling logs from the previous crashed container (--previous flag), interpreting exit codes (1 vs 137 vs 143), and when you'd use ephemeral debug containers to inspect a crashing container that exits too fast to exec into. What are the three most common causes and the fix for each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. A container keeps getting OOMKilled — walk through how you detect this (kubectl describe, events, exit code 137), explain how Kubernetes cgroup memory limits work, how to determine whether the memory limit is too low or the application has a genuine memory leak, how native memory (not tracked by the Node.js heap) and sidecar containers contribute to the total cgroup usage, and what the systematic fix looks like for each scenario.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. A Deployment rollout is stuck and new pods aren't becoming Ready — walk through the debugging steps for each common cause: failing health check probes (readiness/liveness misconfiguration), missing ConfigMaps or Secrets referenced by the pod spec, ImagePullBackOff errors (wrong tag, missing registry credentials), and scheduling failures (insufficient resources, unmet node affinity). Show the kubectl commands that reveal each issue and the fix.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Messages in your event-driven pipeline are being silently dropped — consumers are running, the queue looks healthy, but downstream data is missing. Walk through how you systematically debug this: checking consumer lag and DLQ depth, verifying message schemas and deserialization, inspecting consumer acknowledgment behavior, tracing a single message through the pipeline end to end, and identifying whether the issue is a poison message, a silent exception being swallowed, or an idempotency check incorrectly rejecting messages.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Users see stale data after an update, but the database has the correct value — walk through how you systematically check each cache layer (CDN cache headers and purge, Redis cached value and TTL, application-level cache invalidation, database read replica lag) to find which layer is serving stale data, what commands and tools you use at each step, and how you fix the immediate issue and prevent recurrence.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Incident Scenarios

<details>
<summary>30. Your service starts returning 503 errors during peak traffic and the error rate climbs from 1% to 30% over 15 minutes — walk through how you diagnose connection pool exhaustion (application metrics, database connection counts, thread/worker saturation), how you distinguish it from downstream service degradation vs hitting Kubernetes resource limits vs external rate limiting, and what immediate mitigation steps you take before the root cause is fully understood.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>31. You need to debug an issue that only occurs in production under real traffic — walk through the concrete techniques you'd use without disrupting users: enabling debug-level logging for a specific user or request path using feature flags, setting up traffic mirroring to replay production traffic against a debug instance, using canary deployments to test a fix on a small percentage of traffic, and what guardrails prevent debug tooling itself from causing a new incident.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>32. Set up error tracking for a Node.js backend using Sentry (or equivalent) — show the SDK initialization with environment and release tagging, demonstrate how to enrich errors with user context and request metadata, configure source maps for readable stack traces, set up alert rules that link error spikes to deploys, and configure sampling so high-volume low-severity errors don't blow your event quota while rare critical errors are always captured.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>33. You're the on-call engineer and a SEV1 alert fires at 2 AM — your main API is returning errors for all users. Walk through the exact steps for the first 15 minutes: how you assess impact (dashboards, error rates, affected users), how you communicate status (incident channel, status page, stakeholders), how you decide between rolling back the last deploy vs scaling up vs toggling a feature flag, how you assign roles (incident commander, communicator), and what you do if the first mitigation attempt doesn't work.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>34. Write a blameless postmortem for an incident where a database migration caused 45 minutes of downtime — show the structure: incident summary, timeline with timestamps, root cause analysis that goes beyond the surface ("bad migration" → why wasn't it caught in review, why wasn't it tested against production-size data, why didn't the rollback work), contributing factors, impact assessment, and action items with specific owners and deadlines. What distinguishes a postmortem that actually prevents recurrence from one that just checks a process box?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>35. Tell me about a time you debugged a production issue that couldn't be reproduced in staging or locally — what made it production-specific, how did you approach the investigation without full local reproducibility, and what did you change about your debugging or deployment process afterward?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>36. Describe a time you led or participated in an incident response and the subsequent postmortem — what was the incident, how was it triaged and mitigated, what did the postmortem reveal as root cause vs contributing factors, and did the action items actually get implemented?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>37. Tell me about a time you improved debugging or observability tooling for your team — what gap did you identify (missing traces, poor log structure, no correlation IDs), what did you implement, and how did it measurably improve incident response or debugging speed?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>38. Describe a time you diagnosed a distributed system failure where the root cause was far from the symptoms — how did you trace from the user-visible impact back through multiple services to find the actual broken component, and what made the investigation difficult?</summary>

<!-- Answer framework will be added later -->

</details>
