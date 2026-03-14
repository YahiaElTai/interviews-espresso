# Debugging & Troubleshooting

> **27 questions** — 13 theory, 10 practical, 4 experience

- Why distributed debugging is harder — partial failures, non-deterministic ordering, clock skew, the failing service isn't always the broken service
- Systematic debugging methodology — hypothesize, instrument, test, narrow; mental models (bisection, working backward, fanning out); cognitive traps to avoid; why observability alone doesn't answer "why"
- Production vs staging debugging — why bugs don't reproduce locally (concurrency, load, config drift, data volume), debug logging, feature flags, traffic mirroring, canary analysis
- Log analysis for debugging — structured log queries (Kibana/Loki/CloudWatch), filtering and correlation patterns, extracting signal from high-volume logs, improving error messages for debuggability
- Correlation IDs vs distributed tracing — propagation across HTTP, queues, background jobs, gap handling
- Node.js memory leak detection — common patterns (closures, event listeners, caches, globals), heap snapshots, Chrome DevTools analysis, programmatic memory monitoring (process.memoryUsage(), v8.getHeapStatistics()) for trend detection and health checks
- CPU profiling and flame graphs — on-CPU vs off-CPU, reading flame graphs, identifying event loop blocking vs slow I/O, common Node.js bottlenecks (JSON serialization, regex, sync computation)
- Kubernetes debugging — CrashLoopBackOff (exit codes, ephemeral containers), OOMKilled (cgroup limits), networking failures (DNS, endpoints, network policies), rollout failures (probes, missing configs, image pull)
- Database debugging — slow query identification (pg_stat_statements, EXPLAIN ANALYZE), lock contention diagnosis
- Silent failure debugging — event-driven pipelines where no component errors, dead letter queues, message shape mismatches, detecting data loss in async flows
- Timeout chain debugging — tracing timeout budgets across service hops, retry amplification, distinguishing upstream timeouts from downstream slowness
- Incident response — severity levels (SEV1-SEV4), first 15 minutes triage, mitigation vs fix, escalation, communication during outages
- Blameless postmortems — timeline format, root cause analysis, contributing factors, actionable items with owners

---

## Foundational

<details>
<summary>1. Why is debugging distributed systems fundamentally harder than debugging a single service — what specific properties of distributed systems (partial failures, non-deterministic ordering, clock skew, misleading error locations) make traditional debugging approaches break down, and why is the service reporting the error often not the service that's actually broken?</summary>

Four properties of distributed systems break traditional debugging:

**Partial failures.** In a monolith, things either work or crash. In a distributed system, service A can be healthy while service B is degraded and service C is down. The system is simultaneously working and broken, and the user experience depends on which services their request touches. You can't just check "is it up or down."

**Non-deterministic ordering.** Messages arrive out of order, retries interleave with original requests, and two services processing events from the same queue see them in different sequences. A bug that depends on ordering might happen once in 10,000 requests, making it nearly impossible to reproduce deterministically.

**Clock skew.** Each machine has its own clock, and they drift. When you correlate logs across services using timestamps, a 200ms clock skew can make it look like the effect happened before the cause. This is why distributed tracing (with causal ordering via parent-child spans) matters more than log timestamps for reconstructing event sequences.

**Misleading error locations.** This is the most counterintuitive property. Service A times out calling service B and reports an error. But the real problem is service C — B was waiting on C, which had a connection pool exhaustion. The service that logs the error (A) is healthy. The service that's slow (B) is also healthy — it's just waiting on the actually broken service (C). Traditional debugging says "look at the error logs," which sends you to service A, exactly the wrong place. Errors propagate upstream through dependency chains — the service closest to the user catches the timeout first and reports it, but the root cause is often 2-3 hops away. Without distributed tracing showing the full call graph, you'll waste time investigating symptoms instead of causes.

</details>

<details>
<summary>2. What does a systematic debugging methodology look like for production issues -- how do the mental models of bisection, working backward from symptoms, and fanning out across dependencies guide your investigation differently, when do you apply each one, and why does jumping straight to "check the logs" without a hypothesis waste time?</summary>

The core loop is: **hypothesize, instrument, test, narrow**. Before touching a single log, form a hypothesis about what might be wrong, then look for evidence that confirms or disproves it. This keeps you focused instead of scrolling through thousands of log lines hoping something jumps out.

**Three mental models and when to use each:**

**Bisection** — best when you know "it worked before, now it doesn't." Binary search through the change space: was it the last deploy? The one before? A config change? A data migration? You split the timeline (or the set of changes) in half, check whether the issue exists at the midpoint, and eliminate half the possibilities each step. This is the fastest model for regression bugs.

**Working backward from symptoms** — best for diagnosing a specific failure. Start from the user-visible symptom ("checkout returns 500") and trace backward: What API endpoint? What downstream call fails? What does that service's error look like? What was the state of the data at that point? Each step narrows the scope. This is the default model for debugging a particular incident.

**Fanning out across dependencies** — best when the symptom is vague ("things are slow" or "intermittent errors"). Instead of tracing one request, check the health of all dependencies in parallel: Is the database slow? Is a downstream API timing out? Is the message queue backing up? Is a cache miss rate spiking? You're casting a wide net to find which component is degraded. Once you find it, switch to working backward.

**Why "check the logs" without a hypothesis wastes time:**

Production services generate thousands of log lines per second. Without a hypothesis, you're scanning without knowing what to look for. You'll latch onto the first error you see, which is often a symptom rather than the cause — or an unrelated error entirely. A hypothesis gives you a specific question: "I think the payment service is timing out on database calls — let me filter for payment-service logs with database latency above 500ms." That's a query with an answer, not a fishing expedition.

**Common cognitive traps to avoid:**
- **Anchoring on the first clue**: The first error you find feels important because you found it first, not because it's relevant.
- **Confirmation bias**: Once you have a theory, you subconsciously filter for evidence that supports it. Actively look for evidence that disproves your hypothesis.
- **Recency bias**: "We deployed yesterday, so the deploy must be the cause." Maybe — but the issue could be a data-dependent bug that just happened to trigger now.

</details>

## Conceptual Depth

<details>
<summary>3. Why do production bugs frequently fail to reproduce in staging or local environments — what specific factors (concurrency under load, config drift, data volume and shape, race conditions that only appear at scale) cause this gap, and what strategies (debug logging, feature flags, traffic mirroring, canary analysis) bridge it without requiring full production access?</summary>

**Why production bugs don't reproduce locally:**

**Concurrency under load.** Your local machine handles 1 request at a time during testing. Production handles 500 concurrent requests. Race conditions in connection pools, shared state, or database transactions only surface when multiple requests compete for the same resource simultaneously. A bug that appears at p99 under load is invisible at 1 req/s.

**Config drift.** Staging and production inevitably diverge: different environment variables, different feature flag states, different third-party API endpoints (sandbox vs live), different TLS certificates, different DNS resolution paths. A bug caused by a misconfigured production secret or an expired certificate simply cannot exist in staging.

**Data volume and shape.** Staging has 10,000 rows; production has 50 million. A query that works fine on a small dataset triggers a sequential scan on a large one because PostgreSQL's query planner makes different decisions at different data volumes. Similarly, production data has edge cases (null fields, Unicode characters, extremely long strings) that sanitized test data doesn't.

**Race conditions at scale.** When a message queue consumer processes 10,000 messages/second, the probability of two messages for the same entity arriving within the same processing window becomes significant. At 10 messages/second in staging, it never happens.

**Strategies that bridge the gap:**

**Dynamic debug logging with feature flags.** Instead of redeploying with more logging, use feature flags to enable debug-level logging for a specific user ID, request path, or tenant — in production, without redeploying. This gives you detailed traces of the exact failing path without flooding logs for all traffic.

**Traffic mirroring (shadowing).** Copy production traffic to a debug instance that doesn't serve responses to users. You get real production request patterns, real data shapes, and real concurrency — in an environment where you can attach profilers, enable verbose logging, or run with debug flags. The key: the mirror instance must not write to production databases or send real notifications.

**Canary analysis.** Deploy a potential fix to 1-5% of production traffic and compare error rates, latency, and business metrics against the baseline. This lets you test fixes against real traffic without full rollout risk. Automated canary tools (Flagger, Argo Rollouts) can auto-promote or auto-rollback based on metric comparison.

**Reproduce the conditions, not the environment.** If you suspect a concurrency bug, write a load test that simulates the specific concurrent access pattern. If you suspect a data-shape issue, export a sample of the problematic production records (anonymized) into your test environment.

</details>

<details>
<summary>4. How should log levels be used in practice to maintain a high signal-to-noise ratio — what are the common mistakes teams make with log levels (logging everything at INFO, not distinguishing WARN from ERROR, missing context in messages), why do generic error messages like "something went wrong" actively harm debugging, and what makes a structured log entry actually useful when you're searching across thousands of requests?</summary>

**Log level semantics that actually work:**

- **ERROR**: Something failed and requires human attention. A request couldn't be fulfilled, data was lost, or a critical dependency is down. If an ERROR fires and nobody needs to do anything, it should be WARN.
- **WARN**: Something unexpected happened but the system handled it — a retry succeeded, a fallback kicked in, a deprecated feature was used. Investigate if these accumulate, but a single WARN isn't actionable.
- **INFO**: Significant business or operational events — request completed, order placed, migration finished, service started. This is your production default level. Should be readable at a glance.
- **DEBUG**: Detailed internal state — query parameters, cache hit/miss, intermediate computation results. Never on in production by default; enable selectively via feature flags for specific requests.

**Common mistakes:**

- **Everything at INFO**: Teams log every function entry/exit, every DB query, every cache check at INFO. Production log volume explodes, queries are slow, and costs balloon. The signal-to-noise ratio drops to near zero.
- **WARN vs ERROR confusion**: A failed retry attempt is WARN. The final failure after all retries exhausted is ERROR. Logging every retry as ERROR triggers alert fatigue — your team stops reacting to real errors because they're buried in noise.
- **Missing context**: `logger.error("something went wrong")` is worse than useless — it consumes storage, clutters search results, and tells you nothing. The person debugging at 2 AM has no idea what "something" is, which request triggered it, or what the input was.

**What makes a structured log entry useful:**

```typescript
// Bad
logger.error("Payment failed");

// Good
logger.error({
  msg: "Payment processing failed after 3 retries",
  correlationId: "abc-123",
  userId: "user-456",
  orderId: "order-789",
  paymentProvider: "stripe",
  errorCode: "card_declined",
  amount: 9999,
  currency: "EUR",
  duration_ms: 2340,
}, "Payment processing failed after 3 retries");
```

The useful entry has: **what** happened (payment failed), **who** was affected (userId, orderId), **where** in the request flow (correlationId), **why** it failed (errorCode), and **how long** it took (duration_ms). Every field is queryable. You can now write: `correlationId:"abc-123" AND level:error` to find every error in this request's lifecycle, or `errorCode:"card_declined" AND amount:>5000` to understand a pattern.

**The rule of thumb**: Every log entry should be useful to someone who wasn't the person who wrote it, six months from now, at 2 AM, with no context about the code.

</details>

<details>
<summary>5. What is the difference between correlation IDs and distributed tracing, why do you need both in a microservices architecture, and what does each one tell you during debugging that the other cannot?</summary>

**Why you need both:** Correlation IDs and distributed tracing solve different halves of the debugging problem. Correlation IDs give you the detailed narrative — what happened, in what context, with what data. Traces give you the topology and timing — which service called which, how long each step took, and where latency was introduced. Neither alone tells the full story.

**Correlation IDs** are simple string identifiers (usually UUIDs) attached to every log entry for a request. They let you filter all logs across all services for a single request: `correlationId:"abc-123"` returns every log line that request produced, in every service it touched. They're a grouping mechanism — they tell you "these events belong together."

**Distributed tracing** (OpenTelemetry, Jaeger, etc.) models the request as a tree of **spans** — each span represents a unit of work (an HTTP call, a database query, a queue publish) with a start time, duration, status, and parent-child relationships.

| | Correlation ID | Distributed Trace |
|---|---|---|
| **Shows timing/latency** | No | Yes — each span has duration |
| **Shows call graph** | No — flat list of logs | Yes — parent-child span tree |
| **Shows detailed context** | Yes — log messages, variable values, business data | Limited — span attributes and events |
| **Works with logs** | Yes — primary use case | Indirectly — trace-log correlation |
| **Works across async gaps** | Yes, if propagated | Harder — async breaks parent-child |

**What correlation IDs tell you that traces can't:** The detailed narrative. A trace shows "the database call took 3 seconds" but not what the query was, what parameters were passed, or what the application state looked like before the call. Logs with the correlation ID contain that detail.

**What traces tell you that correlation IDs can't:** The topology and timing. With just correlation IDs, you get a flat list of log lines sorted by timestamp. You can't see that service A called B and C in parallel, B took 50ms, but C called D which took 4 seconds — that's why the whole request was slow. Traces make this hierarchy visible at a glance.

In practice, you link them: each trace has a trace ID, and you include that trace ID in your structured log entries. This lets you jump from a trace span ("this DB call was slow") to the corresponding logs ("here's the exact query and parameters").

</details>

<details>
<summary>6. How does trace context propagate across HTTP calls, message queues, and background jobs -- what mechanisms carry the context in each transport, and what happens when propagation breaks (a service doesn't forward the header, a queue consumer starts a new context) -- how do you detect and handle those gaps?</summary>

**HTTP calls** — the W3C Trace Context standard defines two headers:
- `traceparent`: carries trace ID, parent span ID, and trace flags (e.g., `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`)
- `tracestate`: vendor-specific key-value pairs

OpenTelemetry's HTTP instrumentation automatically injects these headers on outbound requests and extracts them on inbound requests. The child service creates a new span with the extracted trace ID as its trace ID and the parent span ID as its parent, maintaining the tree structure.

**Message queues** — there's no universal header standard. Context is propagated through message attributes/headers:

```typescript
// Producer: inject context into message attributes
const messageAttributes = {};
propagation.inject(context.active(), messageAttributes);

await sqsClient.send(new SendMessageCommand({
  QueueUrl: queueUrl,
  MessageBody: JSON.stringify(payload),
  MessageAttributes: {
    traceparent: { DataType: "String", StringValue: messageAttributes.traceparent },
  },
}));

// Consumer: extract context from message attributes
const parentContext = propagation.extract(ROOT_CONTEXT, message.MessageAttributes);
const span = tracer.startSpan("process-message", {}, parentContext);
```

With RabbitMQ, SQS, or Kafka, you store the `traceparent` in message headers/attributes. The consumer extracts it and starts a child span under the original trace.

**Background jobs** — similar to queues. When enqueueing a job (e.g., BullMQ), serialize the trace context into the job data. When the worker picks it up, extract the context and create a child span.

**What breaks when propagation fails:**

- **A service doesn't forward the header**: The trace splits. Downstream services start new, unconnected traces. You see two separate traces in Jaeger — one ending at the non-propagating service, one starting fresh at the service after it. The request flow becomes invisible across that boundary.
- **A queue consumer starts a new context**: The consumer's processing appears as an orphan trace with no connection to the request that produced the message. You lose the ability to trace from "user clicked checkout" through to "order confirmation email sent."

**Detecting gaps:**
- Look for traces that are suspiciously short (end at a service that should be calling others)
- Monitor for orphan traces — spans with no parent that originate in services that should always be called by other services
- Add integration tests that verify trace propagation end-to-end: publish a message, consume it, and assert the trace ID matches

**Handling gaps:**
- For legacy services that can't be instrumented: deploy a sidecar proxy (Envoy, Linkerd) that handles header propagation at the network level
- For queue systems: wrap your publish/consume in helper functions that always inject/extract context, making it impossible to forget
- Log the correlation/trace ID even when span propagation fails — at minimum, you can correlate by searching logs manually

</details>

<details>
<summary>7. What are the most common Node.js memory leak patterns — why do closures that capture large scopes, forgotten event listeners, unbounded in-memory caches, and accidental globals cause leaks, why are these leaks particularly insidious in long-running server processes compared to request-response scripts, what signals (heap growth over time, increased GC pauses, eventual OOM) indicate a leak is present, and how do process.memoryUsage() and v8.getHeapStatistics() help you detect memory trends programmatically (e.g., in health checks or custom monitoring) before a leak escalates to an OOM kill?</summary>

**The four common leak patterns:**

**1. Closures capturing large scopes.** A closure retains a reference to its entire enclosing scope, not just the variables it uses. If the enclosing scope contains a large buffer, response object, or database result set, the closure keeps all of it alive as long as the closure exists — even if it only uses one small variable from that scope.

```typescript
// Leaky: the callback keeps `largeData` alive as long as the timer exists
async function processRequest(req: Request) {
  const largeData = await fetchHugePayload(); // 50MB
  const summary = summarize(largeData);

  // This closure captures the entire scope, including `largeData`
  setInterval(() => {
    reportMetric(summary); // only uses `summary`, but `largeData` is retained
  }, 60000);
}
```

**2. Forgotten event listeners.** Every `emitter.on()` adds a reference. If you add a listener per request but never call `removeListener`, the emitter accumulates references to every handler — and each handler's closure keeps its scope alive. Node.js warns at 11 listeners (the default `maxListeners`), but many teams just bump the limit instead of fixing the leak.

**3. Unbounded in-memory caches.** A `Map` or plain object used as a cache without eviction grows indefinitely. Every unique key adds an entry that's never removed. This is the most common leak pattern in production because it grows slowly — the cache works perfectly for weeks, then the process OOMs.

```typescript
// Leaky: grows forever
const cache = new Map<string, Result>();
function getCached(key: string): Result {
  if (!cache.has(key)) cache.set(key, computeExpensive(key));
  return cache.get(key)!;
}

// Fixed: use an LRU cache with a size limit
import { LRUCache } from "lru-cache";
const cache = new LRUCache<string, Result>({ max: 10000 });
```

**4. Accidental globals.** Assigning to a variable without `const`/`let`/`var` (in non-strict mode) creates a global. Globals are never garbage collected. TypeScript's strict mode and ESLint rules mostly prevent this, but it still appears in JS dependencies.

**Why long-running servers are especially vulnerable:**

A CLI script or Lambda function runs briefly and exits — the OS reclaims all memory. A server process runs for days or weeks. A leak of 1KB per request at 100 req/s is 8.6GB per day. The slow growth means the leak doesn't show up in load tests (which run for minutes) but kills the process in production after hours.

**Signals of a memory leak:**
- **Heap size trends upward** over hours/days (sawtooth pattern with each GC cycle recovering less)
- **GC pauses increase** as the heap grows — V8 spends more time scanning a larger heap
- **RSS (Resident Set Size) grows monotonically** even during low-traffic periods
- **OOMKilled** in Kubernetes (exit code 137) after the process runs for a predictable duration

**Programmatic detection with `process.memoryUsage()` and `v8.getHeapStatistics()`:**

```typescript
import v8 from "node:v8";

// Periodic memory trend check (run every 60s)
let previousHeapUsed = 0;
let consecutiveGrowth = 0;

setInterval(() => {
  const mem = process.memoryUsage();
  const heap = v8.getHeapStatistics();

  // process.memoryUsage() values:
  // - rss: total memory allocated by the OS to the process
  // - heapUsed: actual JS objects on the heap
  // - heapTotal: V8's allocated heap (includes free space)
  // - external: memory for C++ objects bound to JS (Buffers, etc.)

  // v8.getHeapStatistics() adds:
  // - heap_size_limit: V8's max heap (--max-old-space-size)
  // - used_heap_size / total_heap_size: more granular than process.memoryUsage

  const heapUsedMB = Math.round(mem.heapUsed / 1024 / 1024);
  const heapUtilization = heap.used_heap_size / heap.heap_size_limit;

  // Detect leak: heap growing for 10+ consecutive checks
  if (mem.heapUsed > previousHeapUsed * 1.01) {
    consecutiveGrowth++;
  } else {
    consecutiveGrowth = 0;
  }

  if (consecutiveGrowth > 10) {
    logger.warn({ heapUsedMB, heapUtilization }, "Possible memory leak detected");
  }

  // Health check: fail if heap utilization > 85%
  if (heapUtilization > 0.85) {
    logger.error({ heapUsedMB, heapUtilization }, "Heap nearing limit");
  }

  previousHeapUsed = mem.heapUsed;
}, 60_000);
```

The key insight: `process.memoryUsage().heapUsed` tells you what's actually on the JS heap, while `rss` includes native memory (Buffers, C++ addons). If `rss` grows but `heapUsed` is stable, the leak is in native memory — a different investigation path (question 20 covers this in the OOMKilled context).

</details>

<details>
<summary>8. How do flame graphs work for CPU profiling and what do they reveal that other profiling approaches don't — what is the difference between on-CPU and off-CPU flame graphs, how do you read them to distinguish event loop blocking from slow I/O waits, and what are the most common Node.js CPU bottlenecks (JSON serialization of large payloads, catastrophic regex backtracking, synchronous computation in the main thread)?</summary>

**How flame graphs work:**

A profiler samples the call stack at regular intervals (e.g., every 1ms). Each sample captures the full stack trace — which function is running and what called it. A flame graph visualizes these stacked samples:

- **X-axis**: width represents the percentage of total samples where that function appeared. Wider = more CPU time. The x-axis is NOT a timeline — functions are sorted alphabetically, not chronologically.
- **Y-axis**: stack depth. Bottom is the entry point, top is the leaf function actually consuming CPU.
- **Reading it**: Look for wide plateaus at the top — those are functions that are directly consuming the most CPU. A wide bar in the middle that narrows above it means that function calls many different children.

What flame graphs reveal that other profiling approaches don't: the **full call chain** leading to the bottleneck. A flat profile tells you "JSON.stringify took 40% of CPU" but not which caller invoked it with a 5MB payload. The flame graph shows the exact code path.

**On-CPU vs Off-CPU flame graphs:**

- **On-CPU**: Samples when the thread is actively running on a CPU core. Shows where compute time is spent — parsing JSON, running regex, executing business logic. This is the default and most common type.
- **Off-CPU**: Samples when the thread is blocked/waiting — waiting for I/O, sleeping, waiting for a lock. Shows where the process is spending time NOT doing work.

**For Node.js debugging:**
- If the event loop is blocked (high event loop lag, requests queuing up), the on-CPU flame graph reveals the culprit — a synchronous function monopolizing the single thread.
- If latency is high but CPU usage is low, the problem is off-CPU — the event loop is waiting for slow I/O (database, network). An on-CPU flame graph would look normal because the CPU genuinely isn't busy.

**Common Node.js CPU bottlenecks:**

**JSON serialization of large payloads:** `JSON.stringify()` and `JSON.parse()` are synchronous and run on the main thread. A 10MB object can block the event loop for hundreds of milliseconds. In a flame graph, you'll see a wide `JSON.stringify` bar. Fix: stream large responses, paginate API responses, or use worker threads for heavy serialization.

**Catastrophic regex backtracking:** Certain regex patterns with nested quantifiers (e.g., `/(a+)+b/`) cause exponential backtracking on non-matching input. The regex engine tries every possible combination before failing. In a flame graph, you'll see `RegExp` functions consuming enormous CPU. Fix: rewrite the regex to avoid backtracking, use `re2` (linear-time regex), or add input length limits.

**Synchronous computation in the main thread:** Any CPU-intensive work — image processing, encryption, complex sorting of large arrays, template rendering — blocks the event loop. The flame graph shows the specific function. Fix: offload to worker threads (`worker_threads` module) or a separate service.

**Generating a flame graph in Node.js:**

```bash
# Using clinic.js (easiest)
npx clinic flame -- node dist/server.js
# Then send traffic, Ctrl+C, and it opens an interactive flame graph

# Using 0x
npx 0x dist/server.js

# Using built-in V8 profiler
node --prof dist/server.js
node --prof-process isolate-*.log > processed.txt
```

</details>

<details>
<summary>9. How do you approach database debugging when queries are slow or transactions are blocking each other — what does pg_stat_statements tell you that application-level metrics don't, how does EXPLAIN ANALYZE reveal the actual execution plan vs what the planner estimated, and how do you diagnose lock contention (identifying blocking queries, understanding lock types, determining whether the issue is application design or database configuration)?</summary>

**pg_stat_statements — what it tells you that app metrics don't:**

Application metrics show you "this API endpoint is slow" but not which query is responsible — an endpoint might execute 5 queries, and application-level timing includes network latency, connection pool waits, and ORM overhead. `pg_stat_statements` tracks every query at the database level:

```sql
SELECT
  query,
  calls,
  mean_exec_time,        -- average execution time per call
  total_exec_time,       -- total time spent on this query (calls * mean)
  rows,                  -- total rows returned
  shared_blks_hit,       -- cache hits
  shared_blks_read       -- disk reads (cache misses)
FROM pg_stat_statements
ORDER BY total_exec_time DESC  -- sort by total impact
LIMIT 10;
```

Key insight: sort by `total_exec_time`, not `mean_exec_time`. A query averaging 5ms but called 1 million times/day has more total impact than one averaging 2s but called 10 times/day. The columns `shared_blks_hit` vs `shared_blks_read` reveal whether the query is hitting buffer cache or going to disk.

**EXPLAIN ANALYZE — actual vs estimated:**

`EXPLAIN` shows the planner's *plan* (what it thinks it will do). `EXPLAIN ANALYZE` actually *executes* the query and shows what really happened:

```sql
EXPLAIN ANALYZE
SELECT o.* FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.email = 'user@example.com'
AND o.created_at > '2024-01-01';
```

Output to look for:
- **Seq Scan** on a large table: missing index. The planner scans every row. Fix: add an index on the filtered column.
- **Row estimate mismatch**: `rows=1 (actual rows=50000)`. The planner estimated 1 row but got 50,000. This causes bad join strategies — it might choose a nested loop (good for small result sets) when a hash join would be correct. Fix: run `ANALYZE` to update statistics, or adjust `default_statistics_target`.
- **Nested Loop** with high actual rows on the inner side: the planner chose nested loop expecting few rows but is looping thousands of times. Each loop does index lookups, and the total cost explodes.
- **Sort / Disk**: sorting spills to disk when `work_mem` is too low for the result set size.

**Diagnosing lock contention:**

When transactions block each other, queries queue up and latency spikes. Identify what's blocking what:

```sql
-- Find blocked queries and what's blocking them
SELECT
  blocked.pid AS blocked_pid,
  blocked.query AS blocked_query,
  blocked.wait_event_type,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query,
  blocking.state AS blocking_state,
  now() - blocking.query_start AS blocking_duration
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks gl ON gl.pid != blocked.pid
  AND gl.locktype = bl.locktype
  AND gl.database IS NOT DISTINCT FROM bl.database
  AND gl.relation IS NOT DISTINCT FROM bl.relation
  AND gl.granted
JOIN pg_stat_activity blocking ON blocking.pid = gl.pid
WHERE blocked.wait_event_type = 'Lock';
```

**Common lock contention causes and whether they're app design or DB config:**

- **Long-running transactions** holding row locks while doing external calls (API, email) — app design. Fix: do the external call outside the transaction.
- **Frequent updates to the same row** (a counter, a status field) — app design. Fix: use advisory locks, queue the updates, or redesign the data model.
- **Missing indexes causing full-table locks** during UPDATE — can be either. A `SELECT ... FOR UPDATE` without an index escalates to a broader lock. Fix: add the index.
- **Autovacuum blocked by long transactions** — DB config / app design. Long-running `idle in transaction` sessions prevent autovacuum from cleaning dead tuples, table bloat worsens, queries get slower. Fix: set `idle_in_transaction_session_timeout`.

</details>

<details>
<summary>10. Why are silent failures in event-driven systems particularly dangerous and difficult to debug — what happens when messages are dropped, consumers fail silently, or a dead letter queue fills up without alerting, and why does the fact that no single component reports an error make these failures invisible until a downstream effect (missing data, stale state, customer complaint) surfaces much later?</summary>

**Why silent failures are the worst kind of failures:**

In synchronous request-response systems, a failure is immediately visible — the caller gets an error, the user sees a 500, and alerts fire. In event-driven systems, the producer publishes a message and moves on. If the message is lost, the consumer crashes silently, or the consumer processes it incorrectly, **nobody finds out immediately**. Every component reports healthy because from each component's perspective, nothing went wrong.

**How silent failures manifest:**

**Messages dropped.** A message broker configured for at-most-once delivery loses messages during a broker restart or network partition. The producer logged "message published" (success). The consumer never received it, so it has nothing to log. Zero errors anywhere. The impact — a missing order confirmation, an unprocessed refund, a stale inventory count — surfaces hours or days later as a customer complaint.

**Consumers fail silently.** A consumer deserializes a message, but the schema changed and a required field is now `undefined`. The consumer doesn't crash — it processes the message with `undefined`, writes garbage to the database, and acknowledges the message. No error, no retry, but the data is now corrupt. Alternatively, the consumer has a bug in its error handling — it catches the exception, logs nothing, and acks the message to avoid it being redelivered.

**Dead letter queue (DLQ) fills without alerting.** Messages that fail processing repeatedly get moved to the DLQ — this is by design. But if nobody monitors the DLQ depth, poisoned messages accumulate silently. The consumer appears healthy (no errors, processing other messages fine), but a class of messages is being systematically discarded.

**Why no single component reports the error:**

The fundamental issue is that event-driven systems decouple the producer from the consumer in both time and space. The producer's job ends at "message published." The consumer's job starts at "message received." If the message never arrives, neither side sees a failure — the gap between the two is invisible to both. This is the tradeoff of loose coupling: you gain resilience and scalability but lose the synchronous feedback loop that makes failures immediately visible.

**How to detect these failures:**

- **Message flow metrics**: Track messages published vs messages consumed per topic. A growing delta means messages are being lost or consumers are falling behind. Alert on divergence.
- **DLQ monitoring**: Alert when DLQ depth exceeds zero (or a small threshold). Every message in the DLQ represents a failure that needs investigation.
- **End-to-end business assertions**: "Every order created should have a confirmation email sent within 5 minutes." Run a periodic job that checks for orders without confirmations. This catches failures regardless of where in the pipeline they occurred.
- **Consumer lag monitoring**: Track consumer group offset lag. Growing lag means the consumer can't keep up — either it's slow or it's crashing and restarting.
- **Schema validation at the consumer boundary**: Validate message shape before processing. Reject malformed messages loudly (to the DLQ with logging) rather than processing garbage silently.

</details>

<details>
<summary>11. How do you debug timeout chains across multiple service hops — how do timeout budgets propagate (or fail to propagate) through a call chain, why does retry amplification turn a single slow service into a cascading failure, and how do you distinguish whether the problem is an upstream timeout set too low or a downstream service that's genuinely slow?</summary>

**Timeout budget propagation:**

Consider a chain: API Gateway (10s timeout) -> Service A (5s timeout) -> Service B (3s timeout) -> Database. The "budget" is the time remaining for the overall request. Ideally, each hop subtracts its own processing time and passes the remaining budget downstream via a header like `grpc-timeout` or a custom `X-Request-Deadline`.

In practice, most services set independent timeouts without awareness of the upstream budget. If the gateway gives 10s, Service A sets its own 5s timeout on calls to B, and B sets 3s on the database. These work fine until they don't — if A spends 4s on its own logic before calling B, only 6s remain in the gateway budget, but A gives B a fresh 5s timeout. B might succeed at 4.5s, return to A at 8.5s, and the gateway times out at 10s — even though every individual call "succeeded."

**Retry amplification:**

This is the most dangerous timeout-related failure pattern. Service B becomes slow (responding in 4s instead of 1s). Service A has a 3s timeout and retries 3 times. Each retry hits B, which is already overloaded. Now A sends 3x the normal traffic to B, making B even slower. The gateway retries A (which retries B), creating an exponential blowup:

```
Normal: Gateway -> A -> B (1 request to B per user request)
With retries: Gateway (2 retries) -> A (3 retries) -> B
Worst case: 3 * 4 = 12 requests to B per user request
```

A single slow service receiving 12x its normal load will almost certainly fall over completely, taking the entire chain with it.

**Debugging approach — distinguishing upstream timeout too low vs downstream genuinely slow:**

**Step 1: Check the distributed trace.** Look at the actual latency of each span. If B's span shows 200ms response time but A reports a timeout, A's timeout is too low or there's network latency between A and B. If B's span shows 5s, B is genuinely slow.

**Step 2: Check B's metrics independently.** Look at B's p50, p95, p99 latency. If p50 is fine but p99 spiked, it's a tail-latency problem (maybe garbage collection, maybe a specific query pattern). If all percentiles are elevated, B is systemically slow.

**Step 3: Check for retry amplification.** Compare B's request rate to normal. If it's 3-10x higher than usual, retries are piling up. Look at A's retry metrics — how many retries per request? If A is retrying aggressively, the fix is on A (add backoff, reduce retries, add circuit breakers), not on B.

**Step 4: Check timeout configuration across the chain.** Map out every timeout value in the call chain. Common misconfigurations:
- Downstream timeout longer than upstream timeout (the downstream call will always be killed by the upstream before it completes)
- Retry timeout * retry count exceeds upstream timeout (retries will never complete)
- No deadline propagation — each service starts a fresh timer instead of respecting the remaining budget

**Fixes:**
- Use deadline propagation — pass the remaining budget, not fresh timeouts
- Apply retry budgets: limit total retries across the chain, not per-hop
- Circuit breakers: stop calling a service that's failing instead of hammering it with retries
- Set downstream timeouts strictly shorter than upstream: if the gateway gives 10s, A should give B at most 3-4s, leaving room for A's own processing and one retry

</details>

<details>
<summary>12. How should incident response be structured — what do severity levels (SEV1 through SEV4) represent and who decides the severity, what should the first 15 minutes of a SEV1 incident look like (triage, roles, communication), why is mitigation (stop the bleeding) more important than root-cause fixing during an active incident, and how do escalation and stakeholder communication work without creating chaos?</summary>

**Severity levels:**

| Level | Impact | Example | Response |
|---|---|---|---|
| **SEV1** | Full outage or data loss affecting all/most users | API returning 500s for all requests, data corruption | All hands, 24/7 response, exec communication |
| **SEV2** | Major feature degraded, significant user impact | Payments broken but site loads, 30% of requests failing | On-call + relevant team, business hours escalation |
| **SEV3** | Minor feature impacted, workaround exists | Search slow but functional, one region degraded | On-call investigates, fix in next deploy |
| **SEV4** | Cosmetic or minor, no user-facing impact | Dashboard metric incorrect, non-critical job delayed | Track in backlog, fix when convenient |

**Who decides:** The on-call engineer makes the initial severity call. They should err on the side of over-severity — it's easy to downgrade a SEV1 to SEV2, but the cost of treating a SEV1 as SEV3 for the first 30 minutes is enormous. If in doubt, escalate.

**First 15 minutes of a SEV1:**

1. **Minute 0-2: Acknowledge and assess.** Confirm the alert is real (not a monitoring false positive). Check the primary dashboard: error rate, affected endpoints, user impact scope. Open the incident channel (Slack/Teams).
2. **Minute 2-5: Declare the incident and assign roles.**
   - **Incident Commander (IC)**: Coordinates the response, makes decisions, tracks timeline. Does NOT debug — their job is to keep the response organized.
   - **Communicator**: Posts updates to the status page and stakeholder channels on a cadence (every 15-30 minutes). Shields the debugging team from "what's happening?" messages.
   - **Investigators**: The engineers actually debugging and applying fixes.
3. **Minute 5-10: Correlate with recent changes.** Was there a deploy in the last hour? A config change? A database migration? An infrastructure change? Check the deployment timeline.
4. **Minute 10-15: Apply first mitigation.** Rollback the last deploy, toggle a feature flag, scale up, or switch to a fallback. The goal is to stop the bleeding, not to understand the root cause.

**Why mitigation > root-cause fixing:**

During an active SEV1, every minute costs money, trust, and SLA budget. Understanding the root cause might take hours. Rolling back takes minutes. A 90% mitigation applied in 5 minutes is vastly better than a perfect fix applied in 2 hours. Fix the root cause after the bleeding stops — in the postmortem, not during the incident.

The classic mistake: an engineer starts debugging the code instead of rolling back. "I think I found the bug, let me push a fix." That fix needs code review, CI, deployment — 30+ minutes. Meanwhile, the rollback was one command away.

**Escalation and communication without chaos:**

- **Single source of truth**: All updates go to the incident channel. No side conversations, no DMs with "what's the status?"
- **Structured updates**: "Current status: [impact]. Investigating: [what we're looking at]. Next update in: [N minutes]." This answers everyone's questions preemptively.
- **Escalation path**: If the on-call can't mitigate in 15 minutes, escalate to the team lead. If the team lead can't in 30 minutes, escalate to the engineering manager. The IC drives escalation — the debuggers don't stop to decide who to call.
- **Status page**: External communication goes through the status page, not individual customer replies. "We are aware of degraded API performance and are actively working on resolution."

</details>

<details>
<summary>13. What makes a blameless postmortem effective vs a checkbox exercise — why does the timeline format matter, how do you distinguish root cause from contributing factors, what makes action items actually get completed (owners, deadlines, tracking), and how do you build a culture where postmortems lead to systemic improvements rather than finger-pointing or filing tickets that never get prioritized?</summary>

**Why the timeline format matters:**

A timeline forces precision. Instead of "the migration caused downtime," you get:
- 14:02 — Migration deployed to production
- 14:05 — First error alerts fire
- 14:12 — On-call acknowledges, begins investigation
- 14:18 — Root cause identified as locking migration on orders table
- 14:25 — Rollback initiated
- 14:38 — Rollback complete, errors clear

The timeline reveals gaps that narrative summaries hide. The 7-minute gap between alert and acknowledgment — was the on-call asleep? Was the alert poorly routed? The 13 minutes to roll back — was there no rollback script? Each gap becomes a specific improvement opportunity.

**Root cause vs contributing factors:**

The **root cause** is the single change or condition without which the incident wouldn't have happened. Contributing factors are conditions that made the incident worse, longer, or harder to detect. The distinction matters because fixes target different things:

- **Root cause**: "ALTER TABLE added a column with a NOT NULL constraint and a DEFAULT, which acquires an ACCESS EXCLUSIVE lock on a 50M-row table."
- **Contributing factor 1**: "Migration was not tested against production-sized data — staging has 10K rows, the lock completed in <1s there."
- **Contributing factor 2**: "No migration review checklist that flags locking operations."
- **Contributing factor 3**: "Rollback script didn't exist for this migration — the team had to write one live."
- **Contributing factor 4**: "Alerting on query latency had a 5-minute delay, extending the detection window."

Fixing only the root cause ("rewrite this specific migration") prevents this exact incident but not the next one. Fixing the contributing factors ("add a migration checklist," "require rollback scripts," "reduce alerting delay") prevents a class of incidents.

**What makes action items actually get completed:**

- **Specific owner**: "Backend team" owns nothing. "@alice" owns something. One person, not a team.
- **Deadline**: "Next sprint" is vague. "By March 22" is accountable.
- **Tracked in the same system as other work**: If postmortem action items go into a separate doc that nobody checks, they die. They need to be Jira tickets in the team's sprint backlog, with a postmortem label for tracking.
- **Review cadence**: A weekly or biweekly check — "how many postmortem actions from the last 30 days are still open?" — creates accountability.

**Building a culture that prevents finger-pointing:**

- **Language matters**: "The deploy caused an outage" (blameless) vs "Alice's deploy caused an outage" (blame). The system allowed a dangerous migration to reach production — that's a systems problem, not a people problem.
- **Leadership sets the tone**: If managers use postmortems to evaluate performance, engineers will hide information. If managers treat postmortems as learning opportunities and celebrate thorough ones, engineers will be candid.
- **Ask "what" and "how," not "who"**: "What allowed this migration to bypass safety checks?" leads to process improvements. "Who approved this migration?" leads to blame.
- **Celebrate near-misses**: Encourage reporting incidents that were caught before impact. If people only write postmortems for big outages, they'll avoid reporting smaller issues that could prevent the big ones.

</details>

## Practical — Debugging Workflows & Tooling

<details>
<summary>14. Walk through how you would use structured log queries in Kibana, Loki, or CloudWatch Logs Insights to isolate the root cause of a failing request that spans three services — show example query syntax for filtering by correlation ID, time range, and error level, demonstrate how you narrow from thousands of log lines to the relevant handful, and explain what filtering patterns (exclude health checks, focus on a single trace, aggregate by error type) you apply first.</summary>

**The narrowing funnel: thousands of lines to the relevant handful.**

You start broad and filter down. The order matters — each step reduces the haystack before the next.

**Step 1: Start with what you know.** You have a failing request — maybe a user-reported error, an alert, or a trace ID from a monitoring tool. Find the correlation ID or trace ID from the initial error.

**Step 2: Filter by correlation ID across all services.**

```
# CloudWatch Logs Insights — query across multiple log groups
fields @timestamp, @message, service, level, msg
| filter correlationId = "abc-123-def-456"
| sort @timestamp asc
| limit 200

# Kibana KQL
correlationId: "abc-123-def-456"
# Then sort by @timestamp ascending

# Loki LogQL
{namespace="production"} | json | correlationId = "abc-123-def-456" | line_format "{{.timestamp}} {{.service}} {{.level}} {{.msg}}"
```

This returns every log line from every service that touched this request, in chronological order. You can now read the narrative: request entered service A, A called B, B called C, C returned an error, B propagated it, A returned 500 to the user.

**Step 3: If no correlation ID, narrow by time + error level + service.**

```
# CloudWatch — find the initial error
fields @timestamp, service, correlationId, msg, errorCode
| filter level = "error"
| filter @timestamp between "2024-03-15T14:00:00Z" and "2024-03-15T14:15:00Z"
| filter service in ["api-gateway", "order-service", "payment-service"]
| stats count(*) by errorCode, service
| sort count desc
```

This aggregation shows you which errors are most frequent and in which service — revealing the pattern before you dive into individual requests.

**Step 4: Exclude noise before deep-diving.**

```
# Exclude health checks, readiness probes, and metrics endpoints
| filter path NOT LIKE "/health%"
| filter path NOT LIKE "/ready%"
| filter path NOT LIKE "/metrics%"
| filter userAgent NOT LIKE "kube-probe%"
```

Health checks generate enormous log volume and are almost never relevant to a user-facing failure. Filter them out early.

**Step 5: Once you find the failing service, zoom into its logs.**

```
# CloudWatch — focus on the payment service for this request
fields @timestamp, msg, errorCode, duration_ms, downstreamService, downstreamStatus
| filter correlationId = "abc-123-def-456"
| filter service = "payment-service"
| sort @timestamp asc
```

Now you see the full story within the service: it received the request, attempted to call Stripe, got a timeout, retried twice, then returned an error to the caller.

**Step 6: Aggregate to understand if this is a pattern or one-off.**

```
# Is this a widespread issue or a single request?
fields @timestamp, correlationId, errorCode
| filter service = "payment-service"
| filter level = "error"
| filter @timestamp between "2024-03-15T14:00:00Z" and "2024-03-15T14:15:00Z"
| stats count(*) by errorCode
| sort count desc
```

If you see 500 errors with `errorCode: "STRIPE_TIMEOUT"` in this window, it's a systemic issue with the Stripe integration, not a one-off request failure.

</details>

<details>
<summary>15. Show how to implement correlation ID propagation in a Node.js/TypeScript microservices setup that spans HTTP calls, message queue consumers, and background jobs — demonstrate the middleware that generates or extracts the ID, how it gets passed to downstream HTTP calls and queue message headers, how async context (AsyncLocalStorage) keeps it available without threading it through every function parameter, and what breaks if a single service in the chain doesn't propagate it.</summary>

**The core: AsyncLocalStorage for implicit context propagation.**

AsyncLocalStorage provides request-scoped storage that follows the async call chain without passing arguments through every function. Set it once at the request boundary, read it anywhere in the call stack.

```typescript
// context.ts — shared context module
import { AsyncLocalStorage } from "node:async_hooks";
import { randomUUID } from "node:crypto";

interface RequestContext {
  correlationId: string;
  traceId?: string;
}

export const asyncLocalStorage = new AsyncLocalStorage<RequestContext>();

export function getCorrelationId(): string {
  return asyncLocalStorage.getStore()?.correlationId ?? "unknown";
}
```

**HTTP middleware — extract or generate the correlation ID:**

```typescript
// correlation-middleware.ts
import { Request, Response, NextFunction } from "express";
import { asyncLocalStorage } from "./context";
import { randomUUID } from "node:crypto";

const CORRELATION_HEADER = "x-correlation-id";

export function correlationMiddleware(req: Request, res: Response, next: NextFunction) {
  const correlationId = req.headers[CORRELATION_HEADER] as string ?? randomUUID();

  // Set it on the response so the caller can correlate
  res.setHeader(CORRELATION_HEADER, correlationId);

  // Run all downstream handlers inside this async context
  asyncLocalStorage.run({ correlationId }, () => next());
}
```

**Logger that automatically includes the correlation ID:**

```typescript
// logger.ts — uses pino's mixin option (idiomatic way to enrich every log entry)
import pino from "pino";
import { getCorrelationId } from "./context";

export const logger = pino({
  mixin: () => ({ correlationId: getCorrelationId() }),
});
```

**Downstream HTTP calls — forward the correlation ID:**

```typescript
// http-client.ts
import { getCorrelationId } from "./context";

export async function callDownstream(url: string, body: unknown) {
  return fetch(url, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-correlation-id": getCorrelationId(), // propagate to downstream
    },
    body: JSON.stringify(body),
  });
}
```

**Message queue — embed in message attributes:**

```typescript
// queue-producer.ts
import { getCorrelationId } from "./context";

export async function publishMessage(queue: string, payload: unknown) {
  await sqsClient.send(new SendMessageCommand({
    QueueUrl: queue,
    MessageBody: JSON.stringify(payload),
    MessageAttributes: {
      correlationId: {
        DataType: "String",
        StringValue: getCorrelationId(),
      },
    },
  }));
}

// queue-consumer.ts — extract and restore context
export async function handleMessage(message: SQSMessage) {
  const correlationId =
    message.MessageAttributes?.correlationId?.StringValue ?? randomUUID();

  asyncLocalStorage.run({ correlationId }, async () => {
    // All code inside here has the correlation ID available via getCorrelationId()
    logger.info({ queue: "orders" }, "Processing message");
    await processOrder(JSON.parse(message.Body!));
  });
}
```

**Background jobs — same pattern, serialize into job data:**

```typescript
// Enqueue: include correlation ID in job payload
await queue.add("send-email", {
  correlationId: getCorrelationId(),
  to: user.email,
  template: "order-confirmation",
});

// Worker: restore context before processing
worker.on("active", (job) => {
  asyncLocalStorage.run(
    { correlationId: job.data.correlationId ?? randomUUID() },
    () => processJob(job),
  );
});
```

**What breaks if one service doesn't propagate:**

If Service B in the chain A -> B -> C doesn't forward the `x-correlation-id` header, Service C generates a new UUID. Now A and B share one correlation ID, but C has a different one. When debugging, you search for A's correlation ID and see logs from A and B but nothing from C. C's logs exist but under a different ID — they're invisible unless you know to search for them. The request flow has a gap exactly where the propagation broke.

This is why correlation ID propagation should be infrastructure, not application logic — bake it into shared middleware and HTTP client wrappers so individual services can't forget.

</details>

<details>
<summary>16. Walk through the process of detecting and diagnosing a Node.js memory leak using heap snapshots — show the exact steps to take a heap snapshot in production (or a production-like environment), how to load it into Chrome DevTools, how to compare two snapshots taken minutes apart to identify growing objects, and how to trace a retained object back to the code that's leaking (a closure, an event listener, an unbounded Map). What do you look for in the "Retained Size" vs "Shallow Size" columns?</summary>

**Step 1: Confirm the leak exists.**

Before taking heap snapshots, verify with metrics (as covered in question 7): is `heapUsed` trending upward over time? If yes, proceed. Taking heap snapshots is expensive — it pauses the process, so don't do it speculatively in production.

**Step 2: Take heap snapshots.**

Option A — programmatic snapshots via a debug endpoint (safest for production):

```typescript
import v8 from "node:v8";
import fs from "node:fs";

// Protected debug endpoint — never expose publicly
app.get("/debug/heap-snapshot", authMiddleware, (req, res) => {
  const filename = `/tmp/heap-${Date.now()}.heapsnapshot`;
  const snapshotFile = v8.writeHeapSnapshot(filename);
  res.json({ file: snapshotFile });
});
```

Option B — connect Chrome DevTools remotely:

```bash
# Start Node.js with the inspector
node --inspect=0.0.0.0:9229 dist/server.js

# In production: port-forward through kubectl
kubectl port-forward pod/my-app-xyz 9229:9229

# Open chrome://inspect in Chrome, connect to the target
```

Option C — send a signal to trigger a snapshot:

```typescript
process.on("SIGUSR2", () => {
  const file = v8.writeHeapSnapshot();
  console.log(`Heap snapshot written to ${file}`);
});
```
```bash
kubectl exec my-app-xyz -- kill -USR2 1
```

**Step 3: Take two snapshots with a time gap.**

Take snapshot 1, wait 5-10 minutes (while the app handles traffic), take snapshot 2. The leak is whatever grew between the two.

**Step 4: Compare snapshots in Chrome DevTools.**

1. Open Chrome DevTools -> Memory tab
2. Load both `.heapsnapshot` files
3. Select snapshot 2 and change the view to **"Comparison"** with snapshot 1

The comparison view shows:
- **# New**: Objects allocated between snapshot 1 and 2
- **# Deleted**: Objects freed between snapshots
- **# Delta**: Net change (new - deleted). Positive delta = growing = potential leak
- Sort by **Size Delta** descending to find what's growing fastest

**Step 5: Identify the leaking objects.**

Look for object types with large positive deltas. Common culprits:
- `(string)` — strings accumulating (often in a cache Map)
- `(closure)` — closures being retained (event listeners, timers)
- `(array)` — arrays growing without bounds
- A specific constructor name (e.g., `UserSession`) — instances of a class being retained

**Step 6: Trace retainers — find the code responsible.**

Click on the growing object type, then look at the **Retainers** panel (or "Retaining tree"). This shows the chain of references keeping the object alive — the GC path from a root to this object. Example retainer chain:

```
(global) -> Map -> Map entries -> [key: "user-123"] -> UserSession -> (the leaked object)
```

This tells you: a global `Map` is holding `UserSession` objects that are never removed. The fix: add eviction (TTL or LRU) to the Map, or switch to a proper cache library.

**Shallow Size vs Retained Size:**

- **Shallow Size**: Memory consumed by the object itself — its own fields and properties. A `Map` object's shallow size is small (just the Map structure).
- **Retained Size**: Memory that would be freed if this object were garbage collected — the object itself plus everything it's the sole retainer of. The `Map`'s retained size includes all its entries, all the keys, all the values, and everything those values reference.

**Focus on Retained Size.** An object with a small shallow size but a huge retained size is the real problem — it's a small reference keeping a large tree of objects alive. This is exactly what happens with closures (small closure object retaining a large scope) and caches (small Map object retaining millions of entries).

</details>

<details>
<summary>17. Show how to generate and read a flame graph for a Node.js application that has high CPU usage — walk through the profiling commands (clinic flame, 0x, or --prof), explain how to interpret the resulting flame graph to identify whether the bottleneck is JSON.parse/stringify on large payloads, a regex with catastrophic backtracking, synchronous file I/O, or heavy computation in the event loop, and demonstrate the difference between what an on-CPU flame graph shows vs what you'd see if the issue were slow I/O (off-CPU).</summary>

**Generating the flame graph:**

Use any of the tools from question 8 (`clinic flame`, `0x`, or `node --prof`). For production profiling without restarting the process, connect Chrome DevTools (as described in question 16) and use the CPU Profiler tab to record a profile, then view it as a flame chart. Send traffic with a load testing tool like `autocannon` while profiling.

**Reading the flame graph — identifying specific bottlenecks:**

The conceptual model was covered in question 8. Here's how each bottleneck looks in practice:

**JSON.parse/stringify on large payloads:**
- You'll see a wide bar labeled `JSON.stringify` or `JSON.parse` (V8 internal functions). Follow the stack downward — the caller tells you which endpoint or handler is serializing large objects.
- The bar might appear under `ServerResponse.end` (Express serializing the response body) or under a specific handler that manually stringifies.
- Fix: paginate responses, use streaming JSON (`JSON.stringify` alternatives like `fast-json-stringify` with a schema), or offload to a worker thread.

**Catastrophic regex backtracking:**
- You'll see a wide bar in V8's regex execution internals (`RegExpExecInternal`, `RegExpImpl::IrregexpExec`). The stack below shows which function called the regex.
- This is distinctive because the bar is often disproportionately wide relative to the simplicity of the calling code — a single `string.match()` consuming 60% of CPU.
- Fix: audit the regex pattern for nested quantifiers. Test with `safe-regex` or `rxxr2`. Replace with `re2` for guaranteed linear-time execution.

**Synchronous file I/O:**
- Look for `fs.readFileSync`, `fs.writeFileSync`, or `fs.statSync` in the flame graph. These block the event loop until the OS completes the I/O.
- Often hidden in dependencies — a logging library writing synchronously, a config parser reading files on every request, or `require()` calls inside request handlers (which are synchronous).
- Fix: switch to async equivalents (`fs.promises.readFile`), or move the synchronous operation to startup time.

**On-CPU vs Off-CPU — what you'd see:**

**On-CPU flame graph (event loop blocked — high CPU):**
The flame graph is full and wide. You see application code, V8 internals, JSON functions, regex functions — the CPU is busy. The wide bars at the top tell you exactly what's burning CPU. The event loop can't process other requests because the single thread is occupied.

**Off-CPU scenario (slow I/O — low CPU, high latency):**
The on-CPU flame graph looks normal — narrow, nothing dominating. CPU usage is low. But requests are slow. This means the event loop is idle, waiting for I/O (database, network, file system). An on-CPU flame graph won't help here because the CPU isn't the bottleneck.

To diagnose off-CPU issues, use:
- `clinic doctor` — it distinguishes CPU-bound from I/O-bound bottlenecks and suggests the right tool
- Distributed tracing — shows which downstream call is slow
- `clinic bubbleprof` — visualizes async operations and shows where the event loop is spending time waiting

</details>

<details>
<summary>18. Given a PostgreSQL database where users report slow page loads, walk through the exact steps to identify the problematic queries using pg_stat_statements (which columns matter, how to sort for impact), then take the worst query and run EXPLAIN ANALYZE — show how to read the output to identify sequential scans, bad join strategies, or row estimate mismatches, and demonstrate how to diagnose lock contention by querying pg_locks and pg_stat_activity to find blocking transactions.</summary>

This question covers the same ground as question 9 at a practical walkthrough level. Building on the conceptual foundation from question 9, here's the exact step-by-step debugging session.

**Step 1: Identify the problematic queries with pg_stat_statements.**

```sql
-- Top queries by total execution time (highest cumulative impact)
SELECT
  queryid,
  LEFT(query, 100) AS query_preview,
  calls,
  round(total_exec_time::numeric, 2) AS total_ms,
  round(mean_exec_time::numeric, 2) AS avg_ms,
  round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct_total,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

The key columns and what they reveal are covered in question 9. Sort by `total_exec_time` (cumulative impact), then look at `mean_exec_time`, `calls`, and `shared_blks_read` to understand the pattern.

**Step 2: Take the worst query and run EXPLAIN ANALYZE.**

Suppose the worst query is:
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'DE'
  AND o.status = 'pending'
  AND o.created_at > now() - interval '7 days'
ORDER BY o.created_at DESC
LIMIT 50;
```

**Reading the output — what to look for:**

```
Limit  (cost=1523.45..1523.57 rows=50 width=52) (actual time=3214.112..3214.145 rows=50 loops=1)
  -> Sort  (cost=1523.45..1526.78 rows=1332 width=52) (actual time=3214.110..3214.130 rows=50 loops=1)
        Sort Key: o.created_at DESC
        Sort Method: top-N heapsort  Memory: 32kB
        -> Nested Loop  (cost=0.87..1489.32 rows=1332 width=52) (actual time=0.045..3210.234 rows=8543 loops=1)
              -> Seq Scan on customers c  (cost=0.00..1142.00 rows=4521 width=16) (actual time=0.020..890.456 rows=45210 loops=1)
                    Filter: (country = 'DE')
                    Rows Removed by Filter: 154790
              -> Index Scan using orders_customer_id_idx on orders o  (cost=0.87..0.93 rows=1 width=40) (actual time=0.048..0.051 rows=0 loops=45210)
                    Index Cond: (customer_id = c.id)
                    Filter: ((status = 'pending') AND (created_at > ...))
                    Buffers: shared hit=12 read=135024
```

**Problems identified:**

1. **Seq Scan on customers** — scanning 200K rows to find 45K German customers. Fix: `CREATE INDEX idx_customers_country ON customers(country);`

2. **Row estimate mismatch** — planner estimated `rows=1332` for the join but got `rows=8543`. The planner underestimated, so it chose a Nested Loop (good for few rows) instead of a Hash Join. Fix: `ANALYZE customers; ANALYZE orders;` to update statistics.

3. **Nested Loop with 45,210 iterations** — each loop does an index scan on orders. That's 45K index lookups. With the `Buffers: shared read=135024`, most of those are hitting disk. A Hash Join would be far cheaper.

4. **High actual time vs estimated cost** — `actual time=3214ms` for what the planner estimated as cost 1523. The planner's cost units aren't milliseconds, but the large discrepancy with row estimate mismatches confirms stale statistics.

**Step 3: Diagnose lock contention if queries are blocking.**

The query from question 9 identifies blocking/blocked pairs. Here's a more actionable version:

```sql
-- Active locks with blocking chain
SELECT
  blocked_locks.pid AS blocked_pid,
  blocked_activity.usename AS blocked_user,
  LEFT(blocked_activity.query, 80) AS blocked_query,
  now() - blocked_activity.query_start AS blocked_duration,
  blocking_locks.pid AS blocking_pid,
  blocking_activity.usename AS blocking_user,
  LEFT(blocking_activity.query, 80) AS blocking_query,
  blocking_activity.state AS blocking_state,
  now() - blocking_activity.state_change AS blocking_idle_for
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
  AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
  AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
  AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
  AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
  AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

**Key diagnostic signals:**
- `blocking_state = 'idle in transaction'` — the blocking transaction finished its query but hasn't committed. This is the most common cause: a transaction that holds locks while doing non-database work (API calls, computation). Fix: restructure the code to commit before doing external work, or set `idle_in_transaction_session_timeout`.
- `blocking_idle_for` > 30s — a transaction holding locks for that long is likely problematic regardless of what it's doing.
- If the blocked query is a simple `UPDATE` and the blocking query is an `ALTER TABLE` or `CREATE INDEX` — a DDL operation is blocking DML. Fix: use `CREATE INDEX CONCURRENTLY`.

</details>

## Practical — Infrastructure & System Debugging

<details>
<summary>19. A pod is stuck in CrashLoopBackOff — walk through the exact kubectl commands to diagnose the root cause step by step: checking pod status and restart count, reading events, pulling logs from the previous crashed container (--previous flag), interpreting exit codes (1 vs 137 vs 143), and when you'd use ephemeral debug containers to inspect a crashing container that exits too fast to exec into. What are the three most common causes and the fix for each?</summary>

**Step 1: Check pod status and restart count.**

```bash
kubectl get pods -n production -l app=order-service
# NAME                             READY   STATUS             RESTARTS      AGE
# order-service-7f8b9c6d4-x2k9p   0/1     CrashLoopBackOff   7 (42s ago)   12m
```

`RESTARTS: 7` with increasing backoff delay (10s, 20s, 40s...) confirms CrashLoopBackOff — Kubernetes keeps restarting the container, but it keeps crashing.

**Step 2: Get detailed pod information and events.**

```bash
kubectl describe pod order-service-7f8b9c6d4-x2k9p -n production
```

Look at:
- **Last State**: Shows the exit code of the previous crash. `Terminated: ExitCode: 1` vs `ExitCode: 137` tells you very different things.
- **Events** section at the bottom: Shows the sequence of pulls, starts, and kills. Look for `Failed`, `BackOff`, `Unhealthy` events.
- **Ready/Liveness probe** results: If the container is being killed by a failing liveness probe, the events will show `Unhealthy` followed by `Killing`.

**Step 3: Check logs from the crashed container.**

```bash
# Current (likely empty if the container just restarted)
kubectl logs order-service-7f8b9c6d4-x2k9p -n production

# Previous crashed container — this is the critical one
kubectl logs order-service-7f8b9c6d4-x2k9p -n production --previous
```

The `--previous` flag shows logs from the last terminated container instance. This is where the crash reason usually lives — an unhandled exception, a missing environment variable, a failed database connection.

**Step 4: Interpret exit codes.**

| Exit Code | Meaning | Common Cause |
|---|---|---|
| **1** | Application error | Unhandled exception, missing env var, failed startup check |
| **137** | SIGKILL (128 + 9) | OOMKilled (cgroup limit exceeded) or liveness probe failure (Kubernetes killed it) |
| **143** | SIGTERM (128 + 15) | Graceful shutdown requested — normal during rolling updates, abnormal during CrashLoopBackOff |
| **126** | Command not found / not executable | Bad Dockerfile entrypoint, missing binary |
| **127** | File not found | Wrong command in the container spec |

For exit code 137 specifically, check whether it's OOM or liveness probe:
```bash
# Check for OOMKilled specifically
kubectl get pod order-service-7f8b9c6d4-x2k9p -n production -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# "OOMKilled" = memory limit exceeded
# "Error" with exit code 137 = killed by liveness probe
```

**Step 5: Ephemeral debug containers — when the container exits too fast.**

If the container crashes immediately (within seconds), `kubectl exec` fails because there's nothing to exec into. Ephemeral containers attach a debug container to the pod's namespace:

```bash
# Attach a debug container with shell tools
kubectl debug -it order-service-7f8b9c6d4-x2k9p -n production \
  --image=busybox:latest \
  --target=order-service

# Now you can inspect:
# - The filesystem (check if config files exist)
# - Environment variables (env | grep DATABASE)
# - Network (nslookup database-host, wget -qO- http://config-service/health)
# - Process namespace (ps aux — see the crashing process)
```

**Three most common causes and fixes:**

**1. Application error on startup (exit code 1):**
- Missing or invalid environment variable (e.g., `DATABASE_URL` not set in the ConfigMap/Secret)
- Database or dependency unreachable at startup (no retry logic)
- Syntax error or missing module in the deployed code

Fix: Check `--previous` logs for the error message. Verify ConfigMaps/Secrets are mounted correctly (`kubectl get configmap`, `kubectl describe pod` — look at the Volumes section). Add startup retry logic for dependencies.

**2. OOMKilled (exit code 137, reason: OOMKilled):**
- Memory limit set too low for the application's normal usage
- Memory leak (covered in questions 7 and 20)
- Large request payload causing a memory spike

Fix: Check current memory usage vs limits. If the limit is too low, increase it. If memory grows over time, investigate the leak. See question 20 for the full OOMKilled debugging workflow.

**3. Failing liveness probe (exit code 137, killed by kubelet):**
- Liveness probe `initialDelaySeconds` too short — the app hasn't finished starting before the probe kills it
- Liveness probe endpoint hitting a dependency that's slow (database health check in the liveness probe)
- Liveness probe timeout too aggressive

Fix: Increase `initialDelaySeconds` and `timeoutSeconds`. Use a startup probe (separate from liveness) for slow-starting apps. Keep liveness probes simple — check that the process is alive, not that every dependency is healthy (that's the readiness probe's job).

```yaml
# Common fix: add a startup probe and relax liveness timing
startupProbe:
  httpGet:
    path: /health
    port: 3000
  failureThreshold: 30   # 30 * 10s = 5 minutes to start
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 0  # startup probe handles the delay
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
```

</details>

<details>
<summary>20. A container keeps getting OOMKilled — walk through how you detect this (kubectl describe, events, exit code 137), explain how Kubernetes cgroup memory limits work, how to determine whether the memory limit is too low or the application has a genuine memory leak, how native memory (not tracked by the Node.js heap) and sidecar containers contribute to the total cgroup usage, and what the systematic fix looks like for each scenario.</summary>

**Detecting OOMKilled:**

```bash
# Quick check
kubectl get pod my-app-xyz -n production -o jsonpath='{.status.containerStatuses[*].lastState}'
# Look for: "reason":"OOMKilled"

# Detailed view
kubectl describe pod my-app-xyz -n production
# In the "Last State" section:
#   Terminated:
#     Reason:    OOMKilled
#     Exit Code: 137

# Events will show:
# Warning  OOMKilling  kubelet  Container my-app exceeded its memory limit
```

**How Kubernetes cgroup memory limits work:**

When you set `resources.limits.memory: 512Mi`, Kubernetes configures a cgroup memory limit for the container. The cgroup tracks **all memory** used by all processes in the container — not just the Node.js heap. When total RSS (Resident Set Size) exceeds the cgroup limit, the Linux kernel's OOM killer terminates the process immediately with SIGKILL (exit code 137). There's no graceful shutdown — the process is killed instantly.

Key detail: the cgroup limit applies to the **entire container**, not to a specific process. This matters when sidecars are involved.

**Determining: limit too low vs genuine memory leak.**

**Scenario A — Limit too low (steady-state memory exceeds limit):**

```bash
# Check what the limit is
kubectl get pod my-app-xyz -o jsonpath='{.spec.containers[*].resources.limits.memory}'
# 256Mi

# Check actual usage via metrics
kubectl top pod my-app-xyz -n production
# NAME          CPU    MEMORY
# my-app-xyz    45m    243Mi   <-- close to 256Mi limit = too tight
```

If the pod OOMs shortly after startup or under normal load, and `kubectl top` shows memory consistently near the limit, the limit is simply too low. The application's normal working set exceeds the allocation.

**Scenario B — Memory leak (memory grows over time until OOM):**

Look for the pattern: pod starts fine, runs for hours/days, then OOMs. After restart, it runs fine again for the same duration. Memory grows monotonically over time. This is a leak (covered in detail in questions 7 and 16).

To distinguish: graph memory over time (Prometheus/Grafana). A flat line near the limit = too low. A rising line that eventually hits the limit = leak.

**Native memory — the hidden contributor:**

Node.js `process.memoryUsage().heapUsed` only tracks V8 heap objects. But the cgroup counts everything:

| Memory Type | Tracked by V8 heap? | Examples |
|---|---|---|
| JS objects, strings, closures | Yes | Your application data |
| Buffers (`Buffer.alloc`) | No — `external` in `process.memoryUsage()` | HTTP request/response bodies, file reads |
| Native addons (C++) | No | `sharp` for images, `bcrypt`, `better-sqlite3` |
| Thread stacks (worker_threads) | No | Each worker thread allocates ~4MB for its stack |
| Shared libraries | No | `libuv`, OpenSSL, native modules |
| V8 overhead | Partially | Code compilation cache, internal structures |

A Node.js app with `heapUsed: 150MB` might have an RSS of 400MB due to Buffers, native addons, and V8 overhead. If your cgroup limit is 512Mi and you allocate based on heap size alone, you'll OOM.

**Sidecar containers:**

In multi-container pods (app + sidecar proxy like Envoy, or a log forwarder), each container has its own memory limit. But if you're using a pod-level limit or if sidecars don't have explicit limits, they share the node's resources. An Envoy sidecar can use 50-100MB. If your limit calculation assumed the app is alone, the sidecar's memory pushes the total over.

```yaml
# Always set limits on sidecars explicitly
containers:
  - name: app
    resources:
      limits:
        memory: 512Mi
  - name: envoy-proxy
    resources:
      limits:
        memory: 128Mi
```

**Systematic fix for each scenario:**

**Limit too low:**
1. Measure actual steady-state memory under production load (not local/staging — they differ)
2. Set the limit to 1.5-2x the steady-state usage to allow for spikes (GC cycles, burst traffic)
3. Set `--max-old-space-size` in Node.js to about 75% of the container memory limit, leaving room for native memory:
   ```yaml
   env:
     - name: NODE_OPTIONS
       value: "--max-old-space-size=384"  # for a 512Mi limit
   ```

**Memory leak:**
1. Use the heap snapshot approach from question 16 to identify the leaking code
2. Fix the leak (LRU cache, remove event listeners, fix closure scope)
3. As a temporary mitigation: set memory-based autoscaling or a periodic restart (not ideal, but buys time)

**Native memory growth:**
1. Track `process.memoryUsage().rss` alongside `heapUsed`. If RSS grows but heap doesn't, the leak is in native code.
2. Common culprits: Buffer accumulation (streams not being consumed), native addon leaks, too many worker threads
3. For Buffer leaks: ensure streams are properly piped and consumed. Unconsumed readable streams retain their internal buffers.

</details>

## Practical — Incident Scenarios

<details>
<summary>21. You need to debug an issue that only occurs in production under real traffic — walk through the concrete techniques you'd use without disrupting users: enabling debug-level logging for a specific user or request path using feature flags, setting up traffic mirroring to replay production traffic against a debug instance, using canary deployments to test a fix on a small percentage of traffic, and what guardrails prevent debug tooling itself from causing a new incident.</summary>

Building on the strategies introduced in question 3, here's the concrete implementation of each technique.

**Technique 1: Targeted debug logging via feature flags.**

Instead of redeploying with more logging (risky, slow), toggle debug-level logging for specific requests in real time:

```typescript
// Middleware that checks a feature flag for targeted debug logging
import { getFeatureFlag } from "./feature-flags";

function dynamicLogLevel(req: Request, res: Response, next: NextFunction) {
  const debugTargets = getFeatureFlag("debug-logging-targets");
  // debugTargets = { userIds: ["user-123"], paths: ["/api/checkout"], tenants: ["acme"] }

  const shouldDebug =
    debugTargets.userIds?.includes(req.user?.id) ||
    debugTargets.paths?.some((p: string) => req.path.startsWith(p)) ||
    debugTargets.tenants?.includes(req.headers["x-tenant-id"]);

  if (shouldDebug) {
    req.logLevel = "debug";
    // Log request details that you'd normally skip
    logger.debug({
      headers: redact(req.headers),
      body: redact(req.body),
      query: req.query,
      userId: req.user?.id,
    }, "Debug-targeted request received");
  }

  next();
}
```

This produces detailed logs for just the affected user/path while keeping log volume manageable for everyone else. Toggle it off via the feature flag when done — no deploy needed.

**Technique 2: Traffic mirroring to a debug instance.**

Deploy a separate instance of the service with debug flags, profilers, or verbose logging. Mirror a copy of production traffic to it without affecting the real response path:

```yaml
# Istio traffic mirroring configuration
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: production
          weight: 100
      mirror:
        host: order-service
        subset: debug
      mirrorPercentage:
        value: 5.0  # mirror 5% of traffic
```

The debug instance receives real request shapes, real data patterns, and real concurrency — but its responses are discarded. You can attach CPU profilers, enable verbose logging, or run with `--inspect` without any user impact.

**Critical guardrail**: The debug instance must NOT write to production databases, send real emails/notifications, charge payment methods, or call external APIs with side effects. Point it at read replicas or mock external services.

**Technique 3: Canary deployment to test a fix.**

Once you have a hypothesis and a fix, deploy it to a small percentage of traffic and compare metrics:

```yaml
# Argo Rollouts canary strategy
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 2          # 2% of traffic gets the fix
        - pause: { duration: 10m }
        - analysis:
            templates:
              - templateName: error-rate-check
        - setWeight: 10         # promote to 10% if analysis passes
        - pause: { duration: 15m }
        - analysis:
            templates:
              - templateName: error-rate-check
        - setWeight: 50
        - pause: { duration: 15m }
        - setWeight: 100        # full rollout
```

The analysis template compares the canary's error rate and latency against the baseline. If the canary is worse, it auto-rolls back. If equal or better, it promotes.

**Guardrails to prevent debug tooling from causing new incidents:**

- **Time-boxed debug flags**: Feature flags for debug logging should auto-expire (e.g., max 1 hour). Forgotten debug flags left on will flood logs and increase costs.
- **Resource limits on debug instances**: Set CPU/memory limits so a debug instance with profiling enabled can't starve the cluster.
- **Read-only database access**: Debug instances connect with read-only credentials. No accidental writes.
- **Separate log stream**: Route debug logs to a separate index/stream so they don't pollute production log queries or trigger alerts.
- **Alert on debug flag activation**: Notify the team when debug tooling is enabled so it doesn't get forgotten.
- **No debug endpoints in production without authentication**: Heap snapshot endpoints, profiler triggers, and debug log toggles must be behind strong auth (not just obscure URLs).

</details>

<details>
<summary>22. You're the on-call engineer and a SEV1 alert fires at 2 AM — your main API is returning errors for all users. Walk through the exact steps for the first 15 minutes: how you assess impact (dashboards, error rates, affected users), how you communicate status (incident channel, status page, stakeholders), how you decide between rolling back the last deploy vs scaling up vs toggling a feature flag, how you assign roles (incident commander, communicator), and what you do if the first mitigation attempt doesn't work.</summary>

This builds on the incident response structure from question 12 with a concrete walkthrough.

**Minute 0-2: Alert acknowledgment and initial assessment.**

```
Alert: [SEV1] API error rate > 50% for 5 minutes. All endpoints affected.
```

1. Acknowledge the alert (PagerDuty/OpsGenie) to stop re-escalation.
2. Open the primary dashboard — check:
   - **Error rate**: Is it 100% (total outage) or partial (50% — maybe one pod/region)?
   - **Error type**: 500s (server error) vs 502/503 (infrastructure/upstream) vs 429 (overload)?
   - **Scope**: All endpoints or specific ones? All regions or one?
   - **Timeline**: When did it start? Does it correlate with a deploy?

**Minute 2-4: Declare the incident and open communication.**

```
#incident-2024-03-15-api-outage

@here SEV1 declared — API returning 500s for all users.
Impact: ~100% of API requests failing since 01:55 UTC.
Investigating. Next update in 10 minutes.
```

- Update the status page: "Investigating — We are aware of issues with API availability and are actively investigating."
- If it's 2 AM and you're the only one on, you're IC + investigator. Page the secondary on-call for help.

**Minute 4-8: Correlate with recent changes.**

Check the deployment timeline:
```bash
# What deployed recently?
kubectl rollout history deployment/api-service -n production
# Or check your CI/CD tool's deployment history
```

Check for:
- Deploy in the last 1-2 hours? Strong candidate — investigate the diff.
- Config change (ConfigMap, Secret, feature flag)?
- Infrastructure event (node scaling, certificate rotation, cloud provider incident)?
- Database migration?

**Minute 8-12: Apply first mitigation.**

The decision tree:

**Recent deploy exists?** -> Roll back first, ask questions later.
```bash
kubectl rollout undo deployment/api-service -n production
# Watch: kubectl get pods -w -n production
```

**No recent deploy, errors suggest overload (connection pool exhaustion, timeouts)?** -> Scale up.
```bash
kubectl scale deployment/api-service -n production --replicas=10
```

**Errors tied to a specific feature (new checkout flow, new payment integration)?** -> Toggle the feature flag off.

**Errors suggest database issues (connection refused, slow queries)?** -> Check database health. Is it a RDS failover? Connection limit hit? A runaway query holding locks?

**Minute 12-15: Assess mitigation result.**

After the rollback/scale/toggle, watch the dashboard:
- Error rate dropping? The mitigation worked. Post an update: "Mitigated. Error rate returning to normal. Monitoring."
- Error rate unchanged? The first hypothesis was wrong. Move to the next:

```
#incident-2024-03-15-api-outage

Update: Rolled back last deploy — no improvement. Error rate still at 95%.
Investigating infrastructure. Checking database and downstream services.
Paging @database-team for support.
```

**If the first mitigation doesn't work:**

1. Don't panic — escalate. Page the team lead, the relevant domain expert, or the platform team.
2. Widen the investigation — check all dependencies in parallel (the fanning-out model from question 2): database, cache, message queue, DNS, external APIs.
3. Assign roles properly now that more people are online: one person coordinates (IC), one person communicates (posts updates), others debug specific subsystems.
4. Update the status page: "Identified — The issue is related to [X]. We are working on a fix. ETA: unknown."
5. Consider more drastic mitigations: failover to a different region, enable a static maintenance page, disable non-essential features to reduce load.

**After the crisis passes:** Don't go back to sleep and forget. Log the timeline while it's fresh. Schedule the postmortem for the next business day.

</details>

<details>
<summary>23. Write a blameless postmortem for an incident where a database migration caused 45 minutes of downtime — show the structure: incident summary, timeline with timestamps, root cause analysis that goes beyond the surface ("bad migration" → why wasn't it caught in review, why wasn't it tested against production-size data, why didn't the rollback work), contributing factors, impact assessment, and action items with specific owners and deadlines. What distinguishes a postmortem that actually prevents recurrence from one that just checks a process box?</summary>

## Postmortem: Orders API Outage — 2024-03-15

### Incident Summary

On March 15, 2024, a database migration adding a NOT NULL column with a DEFAULT value to the `orders` table caused a 45-minute full outage of the Orders API. The migration acquired an ACCESS EXCLUSIVE lock on a table with 52 million rows, blocking all reads and writes. The outage affected 100% of users from 14:02 to 14:47 UTC.

**Severity**: SEV1
**Duration**: 45 minutes
**Impact**: All order-related operations (create, read, update) failed. Approximately 12,000 API requests returned 500 errors. Estimated revenue impact: $34,000 in failed checkout attempts.

### Timeline

| Time (UTC) | Event |
|---|---|
| 13:55 | Engineer deploys migration `20240315_add_priority_column` via CI pipeline |
| 13:56 | Migration begins: `ALTER TABLE orders ADD COLUMN priority VARCHAR(10) NOT NULL DEFAULT 'normal'` |
| 14:02 | Error rate alerts fire — Orders API returning 500s (database connection timeouts) |
| 14:05 | On-call acknowledges alert, opens incident channel |
| 14:08 | On-call checks recent deploys, identifies the migration as the likely cause |
| 14:12 | On-call attempts rollback via `migrate:rollback` — fails because the migration's `down()` also needs the ACCESS EXCLUSIVE lock, which is queued behind the pending `up()` |
| 14:18 | On-call attempts to cancel the migration by killing the database session (`pg_terminate_backend`) |
| 14:20 | Migration session terminated, but lock still held by a pending autovacuum triggered by the partial ALTER. On-call doesn't realize this immediately |
| 14:28 | Database team paged, identifies the lingering autovacuum process |
| 14:35 | Autovacuum process terminated, locks released |
| 14:38 | Orders API begins recovering, error rate drops |
| 14:47 | Error rate returns to baseline. Incident resolved |
| 15:00 | Status page updated to "Resolved" |

### Root Cause

The migration `ALTER TABLE orders ADD COLUMN priority VARCHAR(10) NOT NULL DEFAULT 'normal'` acquired an ACCESS EXCLUSIVE lock on the `orders` table (52M rows). The database was running **PostgreSQL 10**, where adding a column with a DEFAULT rewrites the entire table — every row must be physically updated with the new default value, and the ACCESS EXCLUSIVE lock is held for the entire rewrite duration. (In PostgreSQL 11+, this operation would be near-instant because the default is stored in the catalog and the NOT NULL constraint is satisfied by the catalog-stored default, with no table rewrite needed.)

On a 52M-row table running PostgreSQL 10, this rewrite took longer than the API's database connection timeout (30s), causing all queries to the `orders` table to queue behind the lock and eventually time out.

### Contributing Factors

1. **No migration review checklist.** The PR review process has no specific checks for locking operations on large tables. The reviewer approved the migration without considering the table size or lock implications.

2. **Migration not tested against production-sized data.** Staging has 15,000 rows in the `orders` table. The migration completed in <1 second there. The 52M-row production table behaves fundamentally differently.

3. **No rollback plan tested.** The migration's `down()` method (`ALTER TABLE orders DROP COLUMN priority`) also requires an ACCESS EXCLUSIVE lock. In an outage caused by a lock, rolling back with another lock-requiring operation doesn't work.

4. **No statement timeout set for migrations.** The migration ran without a `statement_timeout`, so PostgreSQL let it hold the lock indefinitely. A 30-second statement timeout would have auto-cancelled the migration before it caused a prolonged outage.

5. **Slow detection.** The error rate alert had a 5-minute evaluation window. The outage started at 14:02 but wasn't alerted until 14:02 (the alert fired at the edge of its window). Faster alerting would have reduced detection time.

### Action Items

| # | Action | Owner | Deadline | Status |
|---|---|---|---|---|
| 1 | Create a migration review checklist: flag `ALTER TABLE` on tables > 1M rows, require `CONCURRENTLY` or multi-step approach | @alice | Mar 22 | Open |
| 2 | Add CI check that runs migrations against a production-sized dataset (anonymized snapshot) | @bob | Apr 5 | Open |
| 3 | Set `statement_timeout = 30s` for all migration connections | @carol | Mar 20 | Open |
| 4 | Document the safe migration pattern for adding NOT NULL columns (add nullable -> backfill -> add constraint) | @alice | Mar 25 | Open |
| 5 | Reduce error rate alert evaluation window from 5 minutes to 1 minute | @dave | Mar 19 | Open |
| 6 | Create runbook for "migration caused lock" scenarios, including how to identify and kill blocking sessions | @carol | Mar 29 | Open |

### What Distinguishes This From a Checkbox Postmortem

A checkbox postmortem would say: "Root cause: bad migration. Action: be more careful with migrations." That prevents nothing — it relies on humans remembering to be careful, which they won't.

This postmortem goes beyond the surface at each layer:
- **Why wasn't it caught in review?** -> No checklist exists. Action: create the checklist.
- **Why wasn't it caught in staging?** -> Staging data is too small. Action: test against production-sized data.
- **Why didn't rollback work?** -> The rollback itself requires a lock. Action: document the correct rollback pattern.
- **Why did it run indefinitely?** -> No timeout set. Action: enforce statement timeouts.

Each action item is a systemic fix — a guardrail that prevents the class of incident, not just this specific one. And each has a specific owner and deadline, not "the team will look into it."

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you debugged a production issue that couldn't be reproduced in staging or locally — what made it production-specific, how did you approach the investigation without full local reproducibility, and what did you change about your debugging or deployment process afterward?</summary>

**What the interviewer is looking for:**
- Systematic debugging under uncertainty — not just "I got lucky and found it"
- Understanding of why production differs from other environments (concepts from question 3)
- Ability to investigate without full access (can't just attach a debugger to production)
- Growth mindset — what did you change to prevent similar blind spots?

**Suggested structure (STAR with emphasis on the debugging process):**

1. **Situation** (2-3 sentences): What was the symptom? How was it discovered? Why couldn't it be reproduced locally?
2. **Investigation approach** (the meat of the answer): Walk through your debugging methodology. What hypotheses did you form? What evidence did you gather? What tools did you use? What dead ends did you hit before finding the cause?
3. **Root cause**: What made it production-specific? (Load, data shape, config, race condition, etc.)
4. **Process improvement**: What did you change afterward to make this class of issue easier to debug or prevent?

**Example outline to personalize:**

*"We had intermittent 504 timeouts on our order processing endpoint — about 2% of requests, but only during peak hours. Locally and in staging, the endpoint responded in 200ms consistently."*

*"I started by looking at the slow requests in our traces. The latency was in a database query, but the same query ran fast in staging. The difference: production had 40M rows and staging had 50K. EXPLAIN ANALYZE showed the query planner chose a sequential scan in production because the statistics were stale after a large data import — the planner thought the table had 500K rows. Running ANALYZE fixed it immediately."*

*"Afterward, I set up automated ANALYZE runs after bulk imports and added a dashboard tracking pg_stat_user_tables.last_analyze to catch stale statistics before they cause issues."*

**Key points to hit:**
- Show that you formed hypotheses and tested them, not just randomly checked things
- Highlight the specific production-only factor (data volume, concurrency, config)
- Show the tools you used (traces, logs, database diagnostics)
- End with a systemic improvement, not just "I fixed the bug"

</details>

<details>
<summary>25. Describe a time you led or participated in an incident response and the subsequent postmortem — what was the incident, how was it triaged and mitigated, what did the postmortem reveal as root cause vs contributing factors, and did the action items actually get implemented?</summary>

**What the interviewer is looking for:**
- Can you function under pressure in a real incident (not just theoretical knowledge from question 12)?
- Do you understand the difference between mitigation and root-cause fixing?
- Can you distinguish root cause from contributing factors (question 13)?
- Do you follow through on postmortem action items, or are they shelf-ware?

**Suggested structure:**

1. **The incident** (brief): What broke, what was the user impact, what severity?
2. **Triage and response**: How did you assess impact? What role did you play? How were decisions made?
3. **Mitigation**: What stopped the bleeding? Why that approach?
4. **Postmortem findings**: Root cause vs contributing factors. Be specific about the layers of "why."
5. **Action items and follow-through**: What was decided? Did it actually get done? If not, why not?

**Example outline to personalize:**

*"We had a SEV2 incident where our notification service was sending duplicate emails — some customers received the same order confirmation 5-10 times. I was the on-call engineer."*

*"Triage: I checked the notification service dashboard and saw message processing rate was 3x normal. The message queue consumer was processing messages but failing to acknowledge them due to a timeout on the ACK call, causing redeliveries."*

*"Mitigation: I toggled a feature flag to pause email sending while we investigated. This stopped the duplicate emails within 2 minutes."*

*"Postmortem: Root cause was a deployment that increased the email template rendering time from 200ms to 2s (new template with heavy image processing), which caused the consumer to exceed its ACK timeout. Contributing factors: (1) no limit on redelivery count, (2) no monitoring on duplicate message detection, (3) the template change wasn't flagged as a performance-sensitive change in review."*

*"Action items: We added idempotency keys to prevent duplicate sends (implemented in the next sprint), set a max redelivery count of 3, and added a dashboard for duplicate detection. All three were completed within two weeks because the team lead tracked them in our sprint planning."*

**Key points to hit:**
- Show calm, systematic response — not panic
- Demonstrate the mitigation-first mindset (stop the bleeding before root-causing)
- Show depth in the postmortem analysis — multiple layers of "why"
- Be honest about whether action items got done. If some didn't, explain why (competing priorities, team turnover) — interviewers respect honesty over a perfect story

</details>

<details>
<summary>26. Tell me about a time you improved debugging or observability tooling for your team — what gap did you identify (missing traces, poor log structure, no correlation IDs), what did you implement, and how did it measurably improve incident response or debugging speed?</summary>

**What the interviewer is looking for:**
- Proactive identification of a gap (not just reacting to an incident)
- Technical depth in the implementation (not just "we added logging")
- Measurable improvement — can you quantify the before/after?
- Ability to drive a cross-cutting concern across a team (observability touches every service)

**Suggested structure:**

1. **The gap**: What was the debugging experience like before? What specific pain did the team feel? How did you identify it?
2. **What you implemented**: Technical details — what tools, what patterns, what integration work?
3. **Adoption challenges**: How did you get the team to use it? (This shows senior-level thinking about cross-cutting concerns)
4. **Measurable impact**: How did debugging speed or incident response change?

**Example outline to personalize:**

*"After a particularly painful incident where we spent 3 hours tracing a request across 5 services by manually correlating log timestamps, I identified that we had no correlation IDs and no structured logging — just plaintext console.log statements with inconsistent formats."*

*"I implemented three things: (1) a shared middleware library that generates/propagates correlation IDs using AsyncLocalStorage (similar to question 15), (2) migrated all services from console.log to pino with structured JSON output, and (3) set up a centralized log aggregation with CloudWatch Logs Insights so we could query across services."*

*"The hardest part was adoption. I started by integrating it into our two most critical services myself, then wrote a migration guide and paired with each team member to migrate their services. Having a working example made adoption much easier than a proposal doc would have."*

*"Before: debugging a cross-service issue took 2-4 hours of manually searching log files. After: we could trace any request end-to-end in under 5 minutes by filtering on correlation ID. Our mean-time-to-resolution for incidents dropped from ~3 hours to ~45 minutes over the next quarter."*

**Key points to hit:**
- Show the "before" pain clearly — interviewers want to see that you identified a real problem, not a theoretical one
- Be specific about the technical implementation, not vague
- Address adoption — building the tool is half the work; getting the team to use it is the other half
- Quantify the impact if possible (MTTR reduction, debugging time, number of incidents where it was the key tool)

</details>

<details>
<summary>27. Describe a time you diagnosed a distributed system failure where the root cause was far from the symptoms — how did you trace from the user-visible impact back through multiple services to find the actual broken component, and what made the investigation difficult?</summary>

**What the interviewer is looking for:**
- Understanding of distributed system debugging (the core concept from question 1 — the broken service isn't the reporting service)
- The "working backward" debugging methodology (question 2) applied in practice
- Ability to maintain a clear mental model of a complex system under pressure
- What made it hard — this reveals depth of experience with distributed systems

**Suggested structure:**

1. **The symptom**: What did users/monitoring see? Which service reported the error?
2. **The misleading initial investigation**: Where did you look first? Why was it the wrong place?
3. **The tracing process**: How did you work backward through the dependency chain? What tools did you use? What was each hop in the investigation?
4. **The actual root cause**: What was broken, and why was it so far from the symptom?
5. **What made it difficult**: The specific property of the distributed system that made this hard (indirect dependency, async propagation, misleading metrics, etc.)

**Example outline to personalize:**

*"Users reported that their dashboard was showing stale data — orders placed hours ago weren't appearing. The dashboard service was healthy, returning 200s with data from its cache. No errors anywhere."*

*"I first checked the dashboard service and its database — everything looked normal. The data just wasn't there. I traced backward: the dashboard reads from a read-optimized projection that's built from events published by the order service. The order service was publishing events successfully. The event consumer that builds the projection was running, processing messages, and reporting no errors."*

*"The breakthrough came when I compared message counts: the order service published 50,000 events in the last hour, but the consumer only processed 35,000. The 15,000 missing events were going to a dead letter queue — silently. The consumer was failing to deserialize them because a schema change in the order service added a new field that the consumer's strict validation rejected. The consumer caught the validation error, sent the message to the DLQ, and logged it at WARN level — which nobody was monitoring."*

*"What made it difficult: no single component was in an error state. The order service was healthy. The queue was healthy. The consumer was healthy (processing most messages fine). The dashboard was healthy (serving what it had). The failure was in the gap between components — a silent data loss that only became visible as a business-level symptom (stale data) rather than a technical error."*

**Key points to hit:**
- Emphasize the misleading nature of the symptoms — you looked in the obvious place first and it was wrong
- Show each hop in the investigation chain clearly
- Highlight the specific distributed-systems property that made it hard (silent failures, as covered in question 10, or misleading error locations from question 1)
- End with what you changed to make this type of failure visible earlier (DLQ monitoring, message count reconciliation, schema compatibility testing)

</details>
