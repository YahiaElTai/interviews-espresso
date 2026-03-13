# Observability & Monitoring

> **26 questions** — 13 theory, 10 practical, 3 experience

- Three pillars of observability (logs, metrics, traces) — what each answers, why you need all three, how distributed systems made traditional monitoring insufficient
- Structured logging (JSON format) — log levels, contextual fields, redaction of sensitive data, log injection prevention, child loggers, pino as a Node.js example
- Request context propagation — correlation IDs, generation and propagation across HTTP headers and gRPC metadata, cross-service request tracing for debugging
- Metrics instrumentation — what to measure, counter vs gauge vs histogram selection, label design principles
- RED method (Rate, Errors, Duration) and USE method (Utilization, Saturation, Errors)
- Distributed tracing concepts — spans, traces, context propagation (W3C Trace Context), auto vs manual instrumentation, head vs tail sampling, when traces reveal problems logs and metrics cannot
- OpenTelemetry as the observability standard — SDK and Collector architecture, vendor-neutral instrumentation for metrics/traces/logs, why it replaced fragmented vendor-specific SDKs
- SLIs, SLOs, SLAs, and error budgets — defining indicators, setting targets, burn-rate alerting
- Alerting strategies — actionable alerts, symptom vs cause, severity tiers, alert fatigue prevention
- Health check endpoints (/health, /ready) — K8s probe integration, dependency checks
- Debugging with observability — correlating metrics, traces, and logs to diagnose latency spikes, silent downstream failures, and intermittent errors

---

## Foundational

<details>
<summary>1. What are the three pillars of observability (logs, metrics, traces), what specific questions does each one answer that the others cannot, and why did distributed systems make traditional monitoring (dashboards + server logs) insufficient — what breaks when you only have one or two of the three?</summary>

**The three pillars and what each uniquely answers:**

- **Logs** answer "what exactly happened?" — discrete events with full context (error messages, stack traces, request payloads). They give you the narrative detail for a specific occurrence.
- **Metrics** answer "how is the system behaving over time?" — aggregated numerical measurements (request rate, error percentage, CPU usage). They tell you trends, let you set thresholds, and power dashboards and alerts.
- **Traces** answer "where did time go across services?" — they follow a single request as it flows through multiple services, showing the call graph, timing of each hop, and which service introduced latency or failure.

**Why distributed systems broke traditional monitoring:**

In a monolith, a single server's logs and a few dashboards told the whole story. In a distributed system with 10+ services, a user request touches multiple processes across multiple machines. Traditional monitoring breaks because:

- **Server logs are fragmented**: the request starts in service A, calls B and C, which call D. No single log file has the full picture.
- **Dashboards show symptoms, not causes**: you see elevated error rates but can't tell which downstream dependency is responsible.
- **No causal chain**: without traces, you can't distinguish "service B is slow because service C is slow" from "service B is slow on its own."

**What breaks with only one or two pillars:**

- **Metrics only**: you know latency spiked at 2:15pm but can't tell which requests or why. You detect the problem but can't diagnose it.
- **Logs only**: you can search individual events but can't see trends, set meaningful alerts, or understand how a single request flowed across services. Finding the right log entries in millions of lines without a correlation ID is searching for a needle in a haystack.
- **Metrics + logs, no traces**: you know something is slow (metrics) and can find individual error logs, but you can't reconstruct the request flow to see that the latency comes from a specific downstream call buried three services deep.
- **Metrics + traces, no logs**: you can see the slow span but can't access the detailed error message, stack trace, or business context that explains why it failed.

</details>

<details>
<summary>2. What are the RED method (Rate, Errors, Duration) and USE method (Utilization, Saturation, Errors) — what does each measure, which types of components does each apply to (request-driven services vs infrastructure resources), and how do you decide which method to use when instrumenting a new system?</summary>

**RED method** (by Tom Wilkie) — for **request-driven services** (APIs, microservices, web servers):

- **Rate**: requests per second the service is handling
- **Errors**: number or percentage of those requests that are failing
- **Duration**: distribution of how long requests take (typically measured as histograms for p50/p95/p99)

RED directly measures user experience. If rate drops, errors spike, or duration increases, users are impacted.

**USE method** (by Brendan Gregg) — for **infrastructure resources** (CPU, memory, disk, network, queues):

- **Utilization**: percentage of the resource's capacity being used (e.g., 80% CPU)
- **Saturation**: work queued because the resource is full (e.g., run queue length, swap usage)
- **Errors**: hardware/software errors on the resource (e.g., disk I/O errors, network packet drops)

USE helps you find resource bottlenecks. A resource at high utilization with growing saturation is about to become a problem.

**How to decide which to use:**

| Component type | Method | Example |
|---|---|---|
| Your API endpoints | RED | HTTP request rate, 5xx rate, p99 latency |
| Message consumer | RED | Messages processed/sec, failure rate, processing time |
| Database server (as resource) | USE | Connection pool utilization, queue depth, disk errors |
| CPU/memory/network | USE | CPU %, memory saturation, NIC errors |

In practice, most services need both: RED for the service's external behavior (what users see) and USE for the underlying resources (where bottlenecks hide). Start with RED for every service — it directly maps to user impact. Add USE metrics for resources you've identified as likely bottlenecks or where capacity planning matters.

</details>

<details>
<summary>3. What are SLIs, SLOs, SLAs, and error budgets — how do they relate to each other, why does defining them change how a team operates (feature velocity vs reliability tradeoffs), and what goes wrong when teams set SLOs too aggressively or too loosely?</summary>

**The hierarchy:**

- **SLI (Service Level Indicator)**: a measurable metric that reflects user experience. Examples: "proportion of requests completing in under 300ms," "proportion of requests returning non-5xx responses."
- **SLO (Service Level Objective)**: a target value for an SLI. Example: "99.9% of requests will succeed" or "p99 latency under 500ms." This is an internal engineering goal.
- **SLA (Service Level Agreement)**: a contractual promise to customers with consequences (refunds, credits) if violated. SLAs are always looser than SLOs — you set your internal bar higher so you never breach the contract.
- **Error budget**: the allowed amount of unreliability. If your SLO is 99.9% availability, your error budget is 0.1% — roughly 43 minutes of downtime per month or ~4300 failed requests per million.

**Why defining them changes how a team operates:**

Without SLOs, reliability decisions are driven by gut feeling and fear. With SLOs, you get an objective framework:

- **Error budget remaining?** Ship features, take risks, deploy faster.
- **Error budget nearly exhausted?** Freeze feature work, focus on reliability — fix flaky tests, add retries, address tech debt.
- **Error budget completely burned?** Hard stop on deployments until the budget recovers.

This replaces the eternal "should we ship fast or be careful?" debate with data. It also gives engineering teams cover to push back on feature requests: "we can't ship this right now because our error budget is spent."

**What goes wrong with bad SLO targets:**

- **Too aggressive** (e.g., 99.99% for a non-critical service): the error budget is tiny (~4 minutes/month), so nearly any deployment or hiccup exhausts it. The team is permanently in "reliability mode," unable to ship features, and the SLO becomes something people ignore because it's unachievable.
- **Too loose** (e.g., 95% for a user-facing API): the error budget is so large that real user-impacting problems don't trigger any response. Users complain long before you hit your target, making the SLO meaningless as a signal.

The right SLO is the one that aligns with actual user expectations — tight enough that burning through it means users are genuinely unhappy, loose enough that normal operations and deployments don't constantly exhaust it.

</details>

## Structured Logging — Concepts

<details>
<summary>4. Why does structured logging (JSON format) matter over plain-text logs — what does it enable for log aggregation, querying, and alerting that unstructured logs make difficult or impossible, and what are the tradeoffs (readability in development, log size, serialization cost)?</summary>

**What structured logging enables:**

Plain-text logs like `"2024-01-15 User 123 failed to authenticate from 10.0.0.1"` are human-readable but machine-hostile. To extract the user ID or IP, you need fragile regex patterns that break when the message format changes.

Structured (JSON) logs make every field queryable:

```json
{"level":"warn","timestamp":"2024-01-15T10:30:00Z","msg":"authentication failed","userId":"123","ip":"10.0.0.1","reason":"invalid_token"}
```

This enables:

- **Precise querying**: `userId:123 AND level:error` in your log aggregator (ELK, Datadog, CloudWatch) — no regex needed
- **Aggregation**: count errors by `reason` field to find the most common failure mode
- **Alerting**: trigger an alert when `level:error AND service:auth-service` exceeds a threshold — reliable because you're matching on structured fields, not string patterns
- **Correlation**: include `traceId` and `correlationId` fields to link logs to traces and cross-service requests
- **Dashboards**: build visualizations from log fields (error rate by endpoint, slow queries by table)

**Tradeoffs:**

- **Development readability**: JSON is harder to scan in a terminal. Mitigate with pretty-printing in development (`pino-pretty`, `bunyan` CLI) while keeping JSON in production.
- **Log size**: JSON adds field names and syntax overhead. A structured log line is typically 1.5-2x larger than its plain-text equivalent. In practice, the cost is negligible compared to the querying benefits, and compression in transport (gzip) shrinks the difference.
- **Serialization cost**: JSON.stringify on every log line has CPU overhead. Libraries like pino minimize this by using fast serialization and avoiding synchronous I/O, but it's still more work than concatenating strings. At extreme log volumes (100k+ lines/second), this matters — which is why pino exists as a fast alternative to winston.

</details>

<details>
<summary>5. Why do distributed systems need correlation IDs, how are they generated and propagated across service boundaries (HTTP headers, gRPC metadata), and how do you use them to reconstruct a request's journey across multiple services when debugging a production issue?</summary>

**Why they're needed:**

In a distributed system, a single user request might touch 5-10 services. Each service writes its own logs. Without a shared identifier, there's no way to connect "this error in service D" to "this request that started in service A." A correlation ID is a unique string (typically a UUID) that follows the request across every service boundary, appearing in every log line, so you can query `correlationId:abc-123` and get the complete story.

**Generation and propagation:**

The first service to receive the request (usually the API gateway or edge service) generates the correlation ID and attaches it to a well-known header:

- **HTTP**: `X-Correlation-ID` or `X-Request-ID` header. Each downstream HTTP call includes this header.
- **gRPC**: metadata key `x-correlation-id` attached to the call context. Interceptors extract and propagate it.
- **Message queues**: the correlation ID goes into message headers/attributes (e.g., Kafka record headers, SQS message attributes).

If the incoming request already has a correlation ID (from an upstream service), you preserve it rather than generating a new one.

Within a service, the ID is stored in async context (Node.js `AsyncLocalStorage`) so every log line automatically includes it without manual passing.

**Using it for debugging:**

When a user reports "my request failed at 2:15pm," you:

1. Find the failing request in the edge service logs (by user ID, endpoint, timestamp)
2. Extract the `correlationId` from that log entry
3. Search all services' logs for that correlation ID: `correlationId:"abc-123"`
4. See the full timeline — the request entered the API gateway, was forwarded to the order service, which called the payment service, which timed out calling the external payment provider

Without this, you'd be manually correlating timestamps across services — slow, error-prone, and often impossible when services are in different time zones or have clock skew.

</details>

## Metrics — Concepts

<details>
<summary>6. How do you choose between counter, gauge, and histogram metric types — what does each represent, what goes wrong when you use the wrong type (e.g., a gauge for a cumulative value or a counter for a point-in-time measurement), and when would you reach for a histogram over a simple counter?</summary>

**The three types:**

- **Counter**: a value that only goes up (or resets to zero on restart). Examples: total requests served, total errors, total bytes sent. You use `rate()` to derive per-second rates from counters.
- **Gauge**: a value that goes up and down — a point-in-time measurement. Examples: current memory usage, active connections, queue depth, temperature.
- **Histogram**: records the distribution of values across configurable buckets. Examples: request duration, response size. Lets you compute percentiles (p50, p95, p99) and averages.

**What goes wrong with the wrong type:**

- **Gauge for a cumulative value** (e.g., tracking "total requests" as a gauge): if two requests arrive between scrapes, the gauge shows the final value and you lose the intermediate count. You can't derive accurate rates. After a restart, the gauge shows 0 (correct for a gauge), but for a cumulative metric you've lost history.
- **Counter for a point-in-time measurement** (e.g., tracking "current queue depth" as a counter): counters only go up. When the queue shrinks, your counter still shows the old high value. `rate()` on it would show the rate of queue additions, not the actual queue depth. Completely misleading.

**When to reach for a histogram over a counter:**

Use a histogram when you care about the **distribution**, not just the total. "We processed 1000 requests" (counter) tells you throughput. "p99 of those requests took 2.3s while p50 was 50ms" (histogram) tells you about tail latency affecting your slowest users.

The rule: if the question is "how many?" use a counter. If the question is "how long/big, and for whom?" use a histogram.

**Histogram tradeoff**: histograms are more expensive — each observation updates multiple bucket counters. With 10 buckets and 5 label dimensions, a single histogram metric creates dozens of time series. Use them for measurements that matter (request latency, queue wait time), not for everything.

</details>

<details>
<summary>7. What are the principles for designing metric labels — why does high cardinality matter, what happens to your metrics backend when labels like user ID or request path are unbounded, and how do you decide which dimensions are worth labeling vs which should be logged instead?</summary>

**Why high cardinality matters:**

Every unique combination of label values creates a separate time series. A metric with labels `{method, status, endpoint}` where method has 4 values, status has 5, and endpoint has 20 creates 4 x 5 x 20 = 400 time series. That's manageable.

Now add `userId` with 1 million unique values: suddenly you have 400 million time series. Your metrics backend (Prometheus, Datadog, etc.) must store, index, and query all of them. The result: memory exhaustion, slow queries, skyrocketing costs, and potentially crashing your monitoring infrastructure — the exact tool you need during incidents.

**Label design principles:**

1. **Labels must have bounded cardinality** — the set of possible values should be finite and small. Good: `method` (GET/POST/PUT/DELETE), `status_code_class` (2xx/3xx/4xx/5xx), `region` (us-east/eu-west). Bad: `userId`, `requestId`, `email`.

2. **Normalize high-cardinality paths** — don't use `/users/12345` as a label value. Instead, use the route template: `/users/:id`. Most HTTP frameworks expose this.

3. **Every label should be useful for aggregation** — if you'd never write `sum by (label_name)` or `group by label_name`, the label isn't earning its storage cost.

4. **Prefer fewer labels with clear meaning** — 3-4 well-chosen labels beat 10 that make queries complex and storage expensive.

**Labels vs logs — the decision boundary:**

- **Label it** if you need to aggregate, alert, or visualize by that dimension. "Error rate by endpoint" needs `endpoint` as a label.
- **Log it** if you only need the value when investigating a specific event. User IDs, request payloads, trace IDs — these belong in structured log fields where you search for specific values, not in metrics where every unique value costs storage.

The rule of thumb: if it has more than ~100 unique values, it's probably a log field, not a metric label.

</details>

<details>
<summary>8. How do you decide what to measure in a production service — what signals actually matter for understanding system health vs vanity metrics that look good on dashboards but don't drive action, and how do you balance comprehensive instrumentation against metric explosion and storage costs?</summary>

**Start with what drives action:**

A useful metric is one where a change in its value triggers a human action — investigating, scaling, rolling back, or fixing something. If no one would do anything different based on the metric, it's a vanity metric.

**The practical framework — work inward from user impact:**

1. **RED metrics for every service** (as covered in question 2): request rate, error rate, duration. These directly measure user experience and are the foundation of your SLOs.

2. **Dependency health**: latency and error rate for every outbound call (database queries, HTTP calls to other services, cache operations). When your service is slow, these tell you whether it's your code or a dependency.

3. **Resource saturation signals**: connection pool usage, event loop lag (Node.js-specific), memory heap usage, queue depth. These are leading indicators — they spike before user-facing symptoms appear.

4. **Business-critical operations**: payment processing success rate, order completion rate, authentication failures. These may not be "system" metrics, but they're the metrics the business cares about most.

**What to skip or defer:**

- Internal function timing that doesn't map to user impact
- Metrics that duplicate information already available (e.g., CPU metrics your cloud provider already collects and dashboards)
- Extremely fine-grained breakdowns you've never actually queried

**Balancing instrumentation vs cost:**

- **Use histograms sparingly** — they're the most expensive metric type (as discussed in question 6). Reserve them for latency and size distributions that matter.
- **Control label cardinality** (as covered in question 7) — this is the primary cost driver.
- **Review metrics quarterly**: query your metrics backend for time series that nobody has queried or alerted on in 90 days. Delete them.
- **Tiered retention**: keep high-resolution data (15s intervals) for 2 weeks, downsample to 1-minute resolution for long-term storage.

The goal isn't "measure everything" — it's "measure what you need to detect problems fast and diagnose them faster."

</details>

## Distributed Tracing — Concepts

<details>
<summary>9. Why does distributed tracing exist as a separate pillar from logs and metrics — what is a span, how does context propagation work (W3C Trace Context standard), and what kinds of production problems can you only diagnose with traces (that logs and metrics alone can't answer)?</summary>

**Why tracing is its own pillar:**

Logs give you events. Metrics give you aggregates. Neither shows you **the causal chain of a single request across services**. When a request touches services A -> B -> C -> D, you need to see the call graph, the timing of each hop, and where time was spent. That's what tracing provides — a request-scoped view of the distributed system.

**What is a span:**

A span represents a single unit of work — an HTTP handler, a database query, a function call. Each span has:

- A **trace ID** (shared across all spans in the same request)
- A **span ID** (unique to this span)
- A **parent span ID** (linking it to the caller)
- Start time, duration, status, and attributes (key-value metadata)

A trace is a tree of spans. The root span is the entry point (e.g., the API gateway receiving the request), and child spans branch out for each downstream call.

**W3C Trace Context propagation:**

Services propagate trace context via the `traceparent` HTTP header:

```
traceparent: 00-<trace-id>-<parent-span-id>-<trace-flags>
```

Example: `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`

When service B receives a request from service A, it extracts `traceparent`, creates a new child span with the same trace ID but a new span ID, and includes the updated `traceparent` in any outgoing calls. This is standardized so any W3C-compliant tracing library (OpenTelemetry, Zipkin, etc.) can participate in the same trace regardless of language or vendor.

**Problems only traces can diagnose:**

- **Fan-out latency**: a service calls 5 downstream services in parallel. Metrics show the service is slow. Logs show each downstream responded. Only the trace shows that one of the 5 took 3 seconds while the others took 50ms — and that's the bottleneck.
- **Retry storms**: service A retries failed calls to B, which retries to C. Metrics show elevated traffic. Only traces show the cascading retry pattern.
- **N+1 query patterns**: a service makes 100 individual database calls instead of 1 batch query. Metrics show high DB query count, but only the trace visualization makes the pattern obvious by showing 100 sequential child spans.
- **Cross-service dependency mapping**: traces reveal the actual runtime call graph — which services talk to which, and how often. Metrics and logs can't answer "what's the critical path for this request type?"

</details>

<details>
<summary>10. Why can't you trace 100% of requests in production, what are the tradeoffs between head-based sampling and tail-based sampling, and how do you choose a sampling strategy that captures enough data for debugging without overwhelming your tracing backend with cost and volume?</summary>

**Why 100% tracing is impractical:**

At scale, tracing every request creates enormous data volume. A service handling 10,000 requests/second, where each request generates 15 spans across services, produces 150,000 spans/second. Each span carries attributes, events, and links. The storage, network, and processing costs grow linearly with traffic. Tracing backends (Jaeger, Tempo, Datadog) charge by volume, and ingesting everything can cost more than the infrastructure being monitored.

Beyond cost, 100% tracing also adds overhead to each request — context propagation, span creation, and export all consume CPU and memory.

**Head-based sampling:**

The decision to sample or drop is made at the **start** of the trace (at the edge/entry service). A simple approach: sample 10% of requests randomly.

- **Pro**: simple to implement, predictable cost (you know exactly what percentage you're keeping), low overhead because dropped traces never create spans in downstream services
- **Con**: the sampling decision happens before you know whether the request is interesting. A rare error that happens in 0.1% of requests may never be captured at a 10% sample rate. You're equally likely to sample boring health checks as critical failures.

**Tail-based sampling:**

The decision is made **after** the trace completes, based on what actually happened. The collector buffers all spans temporarily, then decides which traces to keep based on criteria like: errors occurred, latency exceeded a threshold, specific attributes matched.

- **Pro**: captures all interesting traces — every error, every slow request, every trace touching a specific service. Much higher signal-to-noise ratio.
- **Con**: significantly more complex. Requires a stateful collector that buffers spans from all services, waits for the trace to "complete," and then makes the decision. This collector needs enough memory to buffer in-flight traces and enough smarts to know when a trace is done. It's also a single point of failure — if the collector drops spans before the decision is made, you lose data.

**Choosing a strategy:**

| Scenario | Strategy |
|---|---|
| Low traffic (<1000 req/s) | 100% sampling may be feasible |
| Medium traffic, cost-sensitive | Head-based at 10-20% + always-sample errors at the application level |
| High traffic, need reliability debugging | Tail-based sampling on errors, high latency, and specific attributes |
| Very high traffic | Head-based for baseline + tail-based for errors/slow requests (hybrid) |

A practical starting point: head-based sampling at 5-10%, combined with an application-level rule that force-samples any request that results in an error (by setting the trace flag). This gives you baseline traffic for understanding normal behavior plus guaranteed capture of failures.

</details>

## Alerting & SLOs — Concepts

<details>
<summary>11. What makes an alert actionable vs noisy — why should alerts be based on symptoms (user-facing impact) rather than causes (CPU spike), how do severity tiers work in practice, and what patterns lead to alert fatigue where the on-call engineer starts ignoring pages?</summary>

**Actionable vs noisy:**

An actionable alert tells the on-call engineer: something is broken that affects users, and you need to do something about it right now. A noisy alert fires frequently but either resolves itself, doesn't affect users, or doesn't have a clear action.

**Why symptoms over causes:**

A CPU spike to 90% might mean nothing — the garbage collector ran, a batch job completed, traffic is temporarily high but the service is handling it fine. Alerting on this wakes someone up for a non-issue. But "error rate exceeded 1% for 5 minutes" or "p99 latency above 2s" is a symptom that directly tells you users are impacted, regardless of what the underlying cause is.

Cause-based alerts have two problems: (1) not every cause produces user impact (false positives), and (2) not every user impact has the cause you're monitoring (false negatives). Symptom-based alerts catch both: any cause that hurts users triggers the alert, and only actual user impact triggers it.

**Severity tiers in practice:**

| Tier | Meaning | Response | Example |
|---|---|---|---|
| P1 / Critical | Active user impact, revenue loss | Page on-call immediately, respond in minutes | Error rate > 5%, payment processing down |
| P2 / Warning | Degraded performance or risk of escalation | Page during business hours, respond in hours | p99 latency > 1s, error budget burn rate elevated |
| P3 / Info | Something unusual, no immediate impact | Create a ticket, investigate next business day | Disk usage > 70%, elevated retry rate |
| Dashboard-only | Interesting but not actionable | No notification, visible on dashboards | Individual host CPU, cache hit ratio |

**Patterns that cause alert fatigue:**

- **Flapping alerts**: thresholds too close to normal operating values, so the alert fires and resolves repeatedly. Fix with hysteresis (different thresholds for firing vs resolving) or "for" duration requirements.
- **Duplicate alerts**: the same incident triggers 5 alerts from 5 services. Fix by alerting on the symptom at the edge, not on every downstream service.
- **Cause-based alerts at low thresholds**: "CPU > 60%" on every host. These fire constantly and teach engineers to ignore pages.
- **No severity tiers**: treating everything as a page. When everything is critical, nothing is.
- **Stale alerts**: alerts configured months ago for conditions that no longer matter, never cleaned up.

The test: if an on-call engineer ignores an alert and nothing bad happens, that alert should be deleted or downgraded.

</details>

## Error Tracking & Debugging — Concepts

<details>
<summary>12. How do you correlate metrics, traces, and logs together to debug complex production issues — what links them (trace IDs, timestamps, shared labels), why is this correlation the highest-value capability of an observability stack, and what breaks when one pillar is missing or disconnected from the others?</summary>

**What links the three pillars:**

- **Trace ID**: the primary correlation key. When a trace ID appears in metrics (as an exemplar), logs (as a structured field), and the tracing backend, you can jump from any pillar to the others for the same request.
- **Timestamps**: even without trace IDs, aligned timestamps let you correlate a latency spike in metrics with log entries from the same time window.
- **Shared labels/attributes**: `service`, `endpoint`, `environment`, `region` — consistent labels across all three pillars let you filter to the same scope. If metrics use `service=order-api` but logs use `service=orders`, correlation breaks.

**The debugging flow — why correlation is the highest-value capability:**

1. **Metrics detect**: a dashboard or alert shows error rate spiked at 2:15pm on `POST /orders`
2. **Traces diagnose**: you find traces for failed requests at that time. The trace shows the `payment-service` span returning a 500 after 5 seconds. You now know WHERE the problem is.
3. **Logs explain**: you click the trace ID, jump to logs, and see: `"connection pool exhausted, all 10 connections in use, waited 5000ms"`. You now know WHY.

Without correlation, you'd do each step manually — searching logs by timestamp, guessing which service is at fault, hoping the log format includes enough context. That turns a 5-minute diagnosis into a 45-minute one during an incident when minutes matter.

**What breaks when a pillar is disconnected:**

- **Logs without trace IDs**: you can find errors but can't link them to the trace showing the full request path. Debugging multi-service issues requires manual timestamp correlation across log streams.
- **Metrics without exemplars**: you see elevated p99 latency but can't jump to a specific trace that was slow. You're stuck guessing or sampling random traces.
- **Traces without log correlation**: you see a span failed but can't access the detailed error message. You know the database call took 5 seconds but not whether it was a lock contention, a missing index, or a connection timeout.
- **Inconsistent labels**: metrics show errors on `service=auth` but logs and traces use `service=authentication-service`. Dashboards can't link them, and you can't build cross-pillar views.

</details>

## OpenTelemetry — Concepts

<details>
<summary>13. Why did OpenTelemetry emerge as the observability standard — what problems did fragmented vendor-specific SDKs (Jaeger client, Prometheus client, vendor agents) create, how does the OTel architecture (SDK + Collector) solve vendor lock-in, and what does "vendor-neutral instrumentation" actually mean in practice for a team choosing an observability backend?</summary>

**The problem with fragmented SDKs:**

Before OpenTelemetry, instrumenting a service meant choosing vendor-specific libraries:

- Tracing: Jaeger client, Zipkin client, or Datadog's `dd-trace`
- Metrics: Prometheus client, StatsD client, or a vendor agent
- Logs: each logging library with its own format

This created several pain points:

- **Vendor lock-in**: switching from Jaeger to Datadog meant rewriting instrumentation across every service. The cost of switching grew linearly with codebase size.
- **Inconsistent instrumentation**: different teams chose different libraries, so traces from one service couldn't connect to traces from another using a different vendor.
- **Duplicate work**: library authors had to write instrumentation plugins for every vendor. The Express instrumentation for Jaeger was separate from the one for Zipkin.
- **No unified context propagation**: trace context from a Jaeger-instrumented service might not propagate correctly to a Zipkin-instrumented service.

**How OTel's architecture solves this:**

OpenTelemetry separates instrumentation from export into two layers:

1. **SDK (in-process)**: your application code uses OTel's API to create spans, record metrics, and emit logs. This API is vendor-neutral — it doesn't know or care where the data goes. Auto-instrumentation libraries (HTTP, database drivers, etc.) are written once against this API and work for every backend.

2. **Collector (out-of-process)**: a standalone process that receives telemetry data from SDKs, processes it (batching, sampling, enrichment), and exports it to one or more backends (Jaeger, Prometheus, Datadog, Grafana Cloud, etc.). The collector is where vendor-specific configuration lives.

**What "vendor-neutral instrumentation" means in practice:**

Your application code is instrumented once with OTel. When management decides to switch from self-hosted Jaeger to Datadog:

- **Application code**: no changes. Zero.
- **Collector config**: change the exporter from `jaeger` to `datadog`. One config file change.
- **Timeline**: hours, not weeks.

You can even export to multiple backends simultaneously (e.g., Prometheus for metrics + Jaeger for traces + CloudWatch for logs) by configuring multiple exporters in the collector pipeline. This lets teams evaluate new backends without migrating all at once.

</details>

## Practical — Instrumentation & Configuration

<details>
<summary>14. Configure structured logging with pino in a Node.js service — show the setup with appropriate log levels, sensitive field redaction (passwords, tokens), log injection prevention (how attackers exploit unsanitized user input in log fields to forge entries or break log parsing), child loggers for request-scoped context, and explain why each configuration choice matters for production use</summary>

```typescript
import pino from 'pino';

// Base logger configuration
const logger = pino({
  // Use 'info' in production — debug/trace generate massive log volume
  // and slow down the service. Use LOG_LEVEL env var for temporary debugging.
  level: process.env.LOG_LEVEL || 'info',

  // Redact sensitive fields anywhere in the log object.
  // Pino uses fast-redact under the hood — paths are compiled once at startup,
  // so redaction adds negligible per-log overhead.
  redact: {
    paths: [
      'password',
      'token',
      'authorization',
      'req.headers.authorization',
      'req.headers.cookie',
      'creditCard',
      'body.password',
      'body.secret',
      '*.password',      // wildcard: any top-level key containing 'password'
    ],
    censor: '[REDACTED]',
  },

  // Customize the timestamp format for ISO 8601 (default is epoch ms).
  // ISO is human-readable in logs while still machine-parseable.
  timestamp: pino.stdTimeFunctions.isoTime,

  // Format the level as a string ('info') instead of a number (30).
  // Easier to query in log aggregators: level:"error" vs level:50.
  formatters: {
    level(label) {
      return { level: label };
    },
  },
});

export default logger;
```

**Child loggers for request-scoped context:**

```typescript
import { randomUUID } from 'crypto';
import { Request, Response, NextFunction } from 'express';

function requestLoggerMiddleware(req: Request, res: Response, next: NextFunction) {
  // Child logger inherits all parent config (redaction, level, formatters)
  // and adds request-scoped fields to every log line automatically.
  const correlationId = req.headers['x-correlation-id'] as string || randomUUID();

  req.log = logger.child({
    correlationId,
    method: req.method,
    path: req.url,
  });

  req.log.info('request started');
  next();
}

// In a route handler, every log line automatically includes correlationId, method, path:
// req.log.info({ orderId: '456' }, 'order created');
// Output: {"level":"info","time":"...","correlationId":"abc-123","method":"POST","path":"/orders","orderId":"456","msg":"order created"}
```

**Log injection prevention:**

Log injection happens when user-supplied input is written directly into log fields without sanitization. In plain-text logs, an attacker could inject newlines to forge log entries:

```
// User submits username: "admin\n{"level":"info","msg":"user admin logged in successfully"}"
// In plain-text logging, this creates a fake log entry that looks legitimate
```

Pino's structured JSON logging **mitigates this by default** because:

1. JSON serialization escapes special characters — newlines become `\n` in the JSON string, so they can't break the log line structure.
2. Each log entry is a single JSON object on one line, so injected content stays inside the string field.

However, you should still sanitize user input before logging to avoid polluting logs with malicious data:

```typescript
// Don't log raw user input as the message — use it as a field
req.log.info({ username: userInput }, 'login attempt');  // Safe: userInput is a JSON field value
// NOT: req.log.info(`login attempt by ${userInput}`);   // Risky: userInput is in the message string
```

**Development vs production:**

In development, pipe through `pino-pretty` for readable output:

```bash
# Development
node app.js | npx pino-pretty

# Production — raw JSON to stdout, collected by your log shipper
node app.js
```

</details>

<details>
<summary>15. Instrument a Node.js service with OpenTelemetry — show the setup for auto-instrumentation (HTTP, database calls), add a manual span for a custom business operation, configure the collector pipeline (exporter, processor, batching), and explain what each piece does and what you lose without it</summary>

**Step 1: SDK setup with auto-instrumentation (`instrumentation.ts`)**

This file must be loaded before your application code — it monkey-patches HTTP, database drivers, etc. at import time.

```typescript
// instrumentation.ts — loaded via: node --require ./instrumentation.ts app.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-grpc';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  // Resource identifies this service in the tracing backend
  resource: new Resource({
    [ATTR_SERVICE_NAME]: 'order-service',
    [ATTR_SERVICE_VERSION]: '1.2.0',
  }),

  // Exports spans to the OTel Collector via gRPC
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4317',
  }),

  // Exports metrics periodically (default: every 60s)
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter(),
    exportIntervalMillis: 30000, // 30s for faster feedback
  }),

  // Auto-instruments HTTP, Express, PostgreSQL, Redis, gRPC, etc.
  // Each instrumentation patches the library to create spans automatically.
  instrumentations: [
    getNodeAutoInstrumentations({
      // Disable fs instrumentation — too noisy, creates spans for every file read
      '@opentelemetry/instrumentation-fs': { enabled: false },
    }),
  ],
});

sdk.start();

// Graceful shutdown: flush pending spans before process exits
process.on('SIGTERM', () => sdk.shutdown());
```

**What auto-instrumentation gives you without writing any code:**

- HTTP incoming/outgoing request spans (method, URL, status code)
- Database query spans (PostgreSQL, MySQL, Redis) with query text
- gRPC call spans
- Express route spans with route template

**Step 2: Manual span for a custom business operation**

```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('order-service');

async function processOrder(order: Order): Promise<OrderResult> {
  // Manual span wraps business logic that auto-instrumentation can't see
  return tracer.startActiveSpan('process-order', async (span) => {
    try {
      span.setAttribute('order.id', order.id);
      span.setAttribute('order.item_count', order.items.length);
      span.setAttribute('order.total', order.total);

      const result = await validateInventory(order);
      span.addEvent('inventory-validated');

      await chargePayment(order);
      span.addEvent('payment-charged');

      return result;
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: (error as Error).message });
      span.recordException(error as Error);
      throw error;
    } finally {
      span.end(); // Always end the span
    }
  });
}
```

**Step 3: Collector pipeline configuration (`otel-collector-config.yaml`)**

```yaml
# The Collector receives, processes, and exports telemetry data
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317  # SDKs send data here
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Batching: buffers spans and sends them in bulk.
  # Without batching, every span triggers a network call — massive overhead.
  batch:
    timeout: 5s          # Flush every 5 seconds
    send_batch_size: 512 # Or when 512 spans accumulate

  # Memory limiter: prevents the collector from using too much memory
  # under load, dropping data instead of crashing.
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  prometheus:
    endpoint: 0.0.0.0:8889  # Prometheus scrapes metrics from here

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]  # memory_limiter first to protect the collector
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

**What you lose without each piece:**

- **Without auto-instrumentation**: you manually create spans for every HTTP call and DB query, or you have no visibility into library-level operations. Most teams would skip it and have blind spots.
- **Without the collector**: SDKs export directly to backends, coupling your app to a specific vendor. You also lose batching, retry, and the ability to process/filter telemetry before it reaches the backend.
- **Without batching**: every span is exported individually, creating thousands of small network calls per second. This adds latency to your application and can overwhelm the backend.
- **Without memory limiting**: a traffic spike causes the collector to buffer unbounded spans, OOM, and crash — losing all buffered data during the incident when you need it most.

</details>

<details>
<summary>16. Implement correlation ID middleware for a Node.js HTTP service — show how to generate an ID for incoming requests (or extract one from upstream headers), propagate it through async context so all log lines include it, pass it to downstream HTTP/gRPC calls, and explain what breaks in cross-service debugging without this</summary>

**AsyncLocalStorage-based correlation ID propagation:**

```typescript
// correlation-context.ts
import { AsyncLocalStorage } from 'async_hooks';
import { randomUUID } from 'crypto';

interface RequestContext {
  correlationId: string;
}

// AsyncLocalStorage propagates context through the entire async call chain
// without manually passing it as a function parameter.
export const requestContext = new AsyncLocalStorage<RequestContext>();

export function getCorrelationId(): string {
  return requestContext.getStore()?.correlationId || 'no-context';
}
```

**Express middleware — extract or generate the correlation ID:**

```typescript
// correlation-middleware.ts
import { Request, Response, NextFunction } from 'express';
import { randomUUID } from 'crypto';
import { requestContext } from './correlation-context';

export function correlationMiddleware(req: Request, res: Response, next: NextFunction) {
  // Prefer upstream correlation ID to maintain the chain across services.
  // Generate a new one only at the entry point (API gateway, first service).
  const correlationId = (req.headers['x-correlation-id'] as string) || randomUUID();

  // Include it in the response so clients/upstream can log it too
  res.setHeader('x-correlation-id', correlationId);

  // Run all downstream handlers inside this async context
  requestContext.run({ correlationId }, () => next());
}
```

**Logger integration — auto-include correlation ID in every log line:**

```typescript
// logger.ts
import pino from 'pino';
import { getCorrelationId } from './correlation-context';

const baseLogger = pino({
  level: process.env.LOG_LEVEL || 'info',
  // mixin runs on every log call and merges returned fields into the log entry
  mixin() {
    return { correlationId: getCorrelationId() };
  },
});

export default baseLogger;

// Usage in any file — correlationId is automatically included:
// logger.info({ orderId: '123' }, 'order processed');
// Output: {"level":"info","correlationId":"abc-123","orderId":"123","msg":"order processed"}
```

**Propagating to downstream HTTP calls:**

```typescript
// http-client.ts
import { getCorrelationId } from './correlation-context';

async function callDownstreamService(path: string, body: unknown): Promise<Response> {
  return fetch(`http://payment-service:3000${path}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-correlation-id': getCorrelationId(), // Forward the correlation ID
    },
    body: JSON.stringify(body),
  });
}
```

**Wiring it up:**

```typescript
import express from 'express';
import { correlationMiddleware } from './correlation-middleware';

const app = express();
app.use(correlationMiddleware); // Must be early in the middleware chain
```

**What breaks without this:**

As covered conceptually in question 5, without correlation IDs you can't link logs across services for the same request. In practice: a user reports a failed checkout, you find the error in the order service logs, but you can't determine which payment service log entry corresponds to that specific request. With 10,000 requests per minute, manual timestamp correlation is unreliable — especially when services run in different containers with slight clock skew. The correlation ID turns a multi-hour investigation into a single log query.

</details>

<details>
<summary>17. Instrument a Node.js API with custom metrics — create a request duration histogram, an active connections gauge, and a requests-processed counter with appropriate labels. Show the code, explain the label choices, and demonstrate what high-cardinality labels (e.g., user ID as a label) would do to your metrics backend</summary>

```typescript
import { metrics } from '@opentelemetry/api';

const meter = metrics.getMeter('order-service');

// HISTOGRAM: request duration distribution
// Use histogram when you need percentiles (p50, p95, p99), not just averages.
const requestDuration = meter.createHistogram('http_request_duration_ms', {
  description: 'HTTP request duration in milliseconds',
  unit: 'ms',
});

// GAUGE: current active connections (value goes up and down)
const activeConnections = meter.createUpDownCounter('http_active_connections', {
  description: 'Number of currently active HTTP connections',
});

// COUNTER: total requests processed (only goes up)
const requestsTotal = meter.createCounter('http_requests_total', {
  description: 'Total number of HTTP requests processed',
});

// Express middleware to record all three metrics
function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const start = performance.now();
  activeConnections.add(1);

  res.on('finish', () => {
    const duration = performance.now() - start;

    // Labels chosen for bounded cardinality:
    const labels = {
      method: req.method,                      // ~5 values (GET, POST, PUT, DELETE, PATCH)
      route: req.route?.path || 'unknown',     // Route template, not actual path
      status_code: String(res.statusCode),     // ~20 values (200, 201, 400, 404, 500...)
    };

    requestDuration.record(duration, labels);
    requestsTotal.add(1, labels);
    activeConnections.add(-1);
  });

  next();
}
```

**Why these label choices matter:**

- `method`: bounded (GET/POST/PUT/DELETE/PATCH) — ~5 values
- `route`: uses the Express route template (`/users/:id`) not the actual URL (`/users/12345`). Route templates are bounded; actual URLs are unbounded.
- `status_code`: bounded to HTTP status codes — around 20 in practice

With 5 methods x 20 routes x 20 status codes = 2,000 time series per metric. Manageable.

**What high-cardinality labels do — the disaster scenario:**

```typescript
// DO NOT DO THIS
const badLabels = {
  method: req.method,
  route: req.route?.path || 'unknown',
  status_code: String(res.statusCode),
  user_id: req.user?.id,  // 1 million unique users
};
```

With `user_id` added: 5 x 20 x 20 x 1,000,000 = **2 billion time series**. Each time series in Prometheus consumes ~1-2 KB of memory. That's terabytes of memory just for one metric.

Real-world consequences:

- **Prometheus/Mimir**: OOM kills, or the cardinality limit rejects new series and you lose data
- **Datadog/Grafana Cloud**: per-time-series billing means costs explode from hundreds to tens of thousands of dollars per month
- **Query performance**: a simple `rate(http_requests_total[5m])` query must scan billions of series — dashboards time out

The fix: user ID belongs in structured logs (as covered in question 7) where you search for specific values, not in metrics where every unique value creates storage overhead.

</details>

<details>
<summary>18. Implement /health and /ready endpoints for a Node.js service that checks downstream dependencies (database, cache, external API) — show the code, explain the difference between liveness and readiness semantics, show the Kubernetes probe YAML that integrates with these endpoints, and explain what happens when readiness checks are too aggressive or too lenient</summary>

**Liveness vs readiness semantics:**

- **Liveness** (`/health`): "Is the process alive and not deadlocked?" If this fails, Kubernetes kills and restarts the pod. Should be cheap and fast — no dependency checks. A deadlocked process can't respond to HTTP, so just returning 200 is enough.
- **Readiness** (`/ready`): "Can this pod handle traffic right now?" If this fails, Kubernetes removes the pod from the Service's endpoint list — no traffic is routed to it, but the pod is NOT restarted. This is where you check dependencies.

**Implementation:**

```typescript
import express from 'express';
import { Pool } from 'pg';
import Redis from 'ioredis';

const app = express();
const db = new Pool({ connectionString: process.env.DATABASE_URL });
const redis = new Redis(process.env.REDIS_URL);

// Liveness: is the process running and responsive?
// No dependency checks — if the DB is down, restarting this pod won't fix it.
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Readiness: can this pod serve traffic?
// Check every dependency the service needs to function correctly.
app.get('/ready', async (req, res) => {
  const checks: Record<string, 'ok' | 'fail'> = {};

  // Database check — use a lightweight query, not a full table scan
  try {
    await db.query('SELECT 1');
    checks.database = 'ok';
  } catch {
    checks.database = 'fail';
  }

  // Redis check
  try {
    await redis.ping();
    checks.redis = 'ok';
  } catch {
    checks.redis = 'fail';
  }

  const allHealthy = Object.values(checks).every((s) => s === 'ok');

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'ready' : 'not_ready',
    checks,
  });
});
```

**Kubernetes probe configuration:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 3000
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5    # Wait for app to start
            periodSeconds: 10          # Check every 10s
            failureThreshold: 3        # Kill after 3 consecutive failures (30s)
            timeoutSeconds: 2          # Fail if response takes >2s
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 10    # Wait for DB connections to establish
            periodSeconds: 5           # Check every 5s
            failureThreshold: 3        # Remove from service after 3 failures (15s)
            successThreshold: 1        # One success to re-add to service
            timeoutSeconds: 3          # Dependency checks may take slightly longer
```

**What happens when readiness checks are misconfigured:**

- **Too aggressive** (e.g., `failureThreshold: 1`, `periodSeconds: 2`): a single slow database response removes the pod from the service. Under load, the database might be briefly slow — aggressive readiness checks cause cascading removal of pods, reducing capacity and making the problem worse. The remaining pods get even more traffic, their dependency checks also start failing, and the whole deployment goes offline.

- **Too lenient** (e.g., `failureThreshold: 10`, `periodSeconds: 30`): a pod with a broken database connection keeps receiving traffic for 5 minutes before Kubernetes removes it. Users experience errors the entire time. The readiness check exists to prevent this, but with lenient settings it's too slow to help.

- **Dependency checks in liveness** (common mistake): if you check the database in the liveness probe and the database goes down, Kubernetes restarts ALL pods. Restarting pods doesn't fix a down database — it makes things worse by adding restart storms on top of the outage. Dependency checks belong in readiness only.

</details>

## Practical — Production Operations

<details>
<summary>19. Define concrete SLIs for a REST API service (availability, latency, correctness), set SLO targets with specific numbers, calculate the monthly error budget, and show how burn-rate alert thresholds are derived — walk through the math and explain how the team should respond when the error budget is nearly exhausted</summary>

**Step 1: Define SLIs (what to measure)**

| SLI | Definition | How to measure |
|---|---|---|
| Availability | Proportion of requests that return non-5xx responses | `count(status < 500) / count(all requests)` |
| Latency | Proportion of requests completing within the threshold | `count(duration < 300ms) / count(all requests)` |
| Correctness | Proportion of responses with valid data | `count(responses passing validation) / count(all responses)` — requires application-level checks |

**Step 2: Set SLO targets**

- **Availability SLO**: 99.9% of requests succeed (non-5xx)
- **Latency SLO**: 95% of requests complete in under 300ms
- **Correctness SLO**: 99.95% of responses return valid data

**Step 3: Calculate error budgets**

Assume 10 million requests per month (a moderate-traffic API):

| SLO | Error budget (%) | Error budget (requests) | Error budget (time equivalent) |
|---|---|---|---|
| 99.9% availability | 0.1% | 10,000 failed requests | ~43 minutes of total downtime |
| 95% latency | 5% | 500,000 slow requests | N/A (latency, not downtime) |
| 99.95% correctness | 0.05% | 5,000 incorrect responses | N/A |

**Step 4: Derive burn-rate alerts**

The burn rate is how fast you're consuming your error budget relative to the SLO window. A burn rate of 1 means you'll exactly exhaust the budget by month end. A burn rate of 14.4 means you'll exhaust it in 1/14.4 of a month — roughly 2 days.

For the availability SLO (99.9%, 10,000 error budget):

```
Burn rate = (actual error rate) / (allowed error rate)
         = (actual error rate) / 0.1%
```

**Multi-window burn-rate alerting** (Google SRE approach):

| Severity | Burn rate | Long window | Short window | Time to exhaust budget | Action |
|---|---|---|---|---|---|
| P1 (page) | 14.4x | 1 hour | 5 min | ~2 days | Page on-call immediately |
| P2 (page) | 6x | 6 hours | 30 min | ~5 days | Page during business hours |
| P3 (ticket) | 1x | 3 days | 6 hours | ~30 days | Create ticket, investigate |

The two-window approach prevents false alarms: the long window confirms a sustained trend, the short window confirms it's still happening right now. Without the short window, you'd page for a spike that already resolved.

**Example Prometheus alert rule:**

```yaml
# P1: burning 14.4x budget over a 1-hour window, confirmed in 5-min window
- alert: HighErrorBudgetBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h]))
      / sum(rate(http_requests_total[1h]))
    ) > (14.4 * 0.001)
    AND
    (
      sum(rate(http_requests_total{status=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m]))
    ) > (14.4 * 0.001)
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning at 14.4x — budget exhausted in ~2 days"
```

**Step 5: Team response when budget is nearly exhausted**

- **>50% budget remaining**: normal operations, ship features
- **25-50% remaining**: heightened awareness — review recent deployments, check for regressions, no risky changes
- **<25% remaining**: reliability sprint — freeze non-critical features, prioritize the top error source, add missing instrumentation to prevent recurrence
- **Budget exhausted**: deployment freeze until budget recovers in the next window. Only reliability fixes and rollbacks allowed. Post-incident review to understand what consumed the budget.

</details>

<details>
<summary>20. Design an alerting strategy for a microservices application — show example alert rules that follow symptom-based alerting principles, define severity tiers (page vs ticket vs dashboard), set up routing and escalation, and explain how you'd evaluate whether the alerting is working well after a month in production</summary>

**Principle: alert on symptoms at the edge, not causes inside each service.**

In a microservices app with an API gateway, 5 backend services, a database, and a cache, don't alert on every service's CPU or memory. Alert on what users experience: error rates, latency, and availability at the entry point.

**Example alert rules (Prometheus/Alertmanager syntax):**

```yaml
groups:
  - name: symptom-alerts
    rules:
      # P1: User-facing error rate exceeds SLO burn rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5..", service="api-gateway"}[5m]))
          / sum(rate(http_requests_total{service="api-gateway"}[5m]))
          > 0.01
        for: 3m  # Must persist for 3 minutes to avoid transient spikes
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "API error rate above 1% for 3+ minutes"
          runbook: "https://wiki.internal/runbooks/high-error-rate"

      # P2: Latency degradation
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, sum(rate(http_request_duration_ms_bucket{service="api-gateway"}[5m])) by (le))
          > 2000
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "p99 latency above 2s for 5+ minutes"

      # P3: Leading indicator — connection pool nearing saturation
      - alert: DatabasePoolSaturation
        expr: |
          pg_pool_active_connections / pg_pool_max_connections > 0.85
        for: 10m
        labels:
          severity: ticket
          team: backend
        annotations:
          summary: "DB connection pool at >85% capacity for 10+ minutes"
```

**Severity tiers and routing:**

| Severity | Response time | Notification | Example |
|---|---|---|---|
| `critical` (P1) | Immediate (minutes) | PagerDuty page to on-call | Error rate > 1%, payment failures, data loss |
| `warning` (P2) | Hours (business hours) | Slack alert channel + PagerDuty (low urgency) | Latency degraded, error budget burning fast |
| `ticket` (P3) | Days | Auto-create Jira ticket + Slack | Pool saturation, disk usage > 70%, cert expiring in 14 days |
| `info` | None | Dashboard only | Cache hit ratio changed, deployment completed |

**Alertmanager routing and escalation:**

```yaml
route:
  receiver: default-slack
  group_by: ['alertname', 'service']
  group_wait: 30s       # Wait before sending first notification (batches related alerts)
  group_interval: 5m    # Wait before sending updates for same group
  repeat_interval: 4h   # Re-notify if alert is still firing

  routes:
    - match:
        severity: critical
      receiver: pagerduty-oncall
      continue: true  # Also send to Slack
    - match:
        severity: critical
      receiver: slack-critical
    - match:
        severity: warning
      receiver: slack-alerts
    - match:
        severity: ticket
      receiver: jira-ticket

receivers:
  - name: pagerduty-oncall
    pagerduty_configs:
      - service_key: '<key>'
        escalation_policy: 'backend-oncall'
  - name: slack-critical
    slack_configs:
      - channel: '#incidents'
  - name: slack-alerts
    slack_configs:
      - channel: '#alerts'
  - name: jira-ticket
    webhook_configs:
      - url: 'http://jira-webhook-bridge/create-ticket'
```

**Evaluating alerting quality after a month:**

Track these metrics:

1. **Alert-to-incident ratio**: what percentage of pages corresponded to real user impact? Target: >80%. Below that, alerts are too noisy.
2. **Incidents without alerts**: how many user-reported issues had no corresponding alert? Each one is a gap in coverage.
3. **Time to acknowledge**: how quickly are pages acknowledged? Increasing times signal fatigue.
4. **Flapping alerts**: alerts that fired and resolved more than 3 times in a day. Each one needs a threshold adjustment or a `for` duration increase.
5. **Actions per alert**: for each alert type, did the on-call engineer take action or just acknowledge and ignore? Ignored alerts should be downgraded or deleted.

Review these monthly with the team. Delete or downgrade alerts nobody acts on. Add alerts for incidents that had no coverage. The goal is: every page means real user impact, and every real incident triggers a page.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>21. An API endpoint's p99 latency spiked from 200ms to 2s starting 30 minutes ago — walk through step by step how you use metrics to identify which endpoint and when, traces to find the slow span, and logs to find the root cause. Show the specific queries and reasoning at each step</summary>

**Step 1: Metrics — narrow down WHAT and WHEN**

Start with a broad query to confirm the spike and identify which endpoint:

```promql
# p99 latency per endpoint over the last hour
histogram_quantile(0.99,
  sum(rate(http_request_duration_ms_bucket{service="order-service"}[5m])) by (le, route)
)
```

This shows `POST /orders` spiked from 200ms to 2s starting at 14:45. Other endpoints are normal, so it's isolated to one route.

Check if traffic volume changed (ruling out load-related cause):

```promql
# Request rate per endpoint
sum(rate(http_requests_total{service="order-service", route="/orders", method="POST"}[5m]))
```

Traffic is flat — the spike isn't caused by increased load. Something in the request processing path got slower.

Check downstream dependency latency:

```promql
# Outbound call duration by target service
histogram_quantile(0.99,
  sum(rate(outbound_request_duration_ms_bucket{service="order-service"}[5m])) by (le, target)
)
```

This reveals calls to `inventory-service` went from 50ms to 1.8s at the same time. The slow span is likely the inventory call.

**Step 2: Traces — find WHERE in the request path the time is spent**

In Jaeger/Tempo, query for slow traces:

- Service: `order-service`
- Operation: `POST /orders`
- Min duration: `1500ms`
- Time range: last 45 minutes

Open a slow trace. The waterfall shows:

```
POST /orders (2100ms total)
├── validate-input (2ms)
├── inventory-service.checkStock (1850ms)  ← the bottleneck
│   └── GET inventory-service/stock (1850ms)
├── payment-service.charge (180ms)
└── database INSERT (45ms)
```

The `inventory-service.checkStock` span is consuming 88% of the total request time. Click into that span — the span attributes show `db.statement: SELECT * FROM inventory WHERE sku IN (...)` and `db.duration: 1800ms`.

So the inventory service itself is slow because of a database query.

**Step 3: Logs — find WHY**

Take the trace ID from the slow trace and search logs:

```
# In your log aggregator (Loki, Elasticsearch, CloudWatch)
{service="inventory-service"} |= "trace_id=abc123def456"
```

The logs for this trace show:

```json
{"level":"warn","msg":"slow query detected","query":"SELECT * FROM inventory WHERE sku IN (...)","duration_ms":1800,"rows_scanned":2400000,"trace_id":"abc123def456"}
```

2.4 million rows scanned. Check what changed — search for recent deployments or schema changes:

```
{service="inventory-service"} |= "migration" OR "deploy" | json | time > 14:30
```

Found: a migration at 14:42 dropped an index on the `sku` column as part of a "cleanup." The query went from an index scan to a full table scan.

**Resolution**: recreate the index. The fix is a one-line migration. Latency returns to normal within minutes of the index being rebuilt.

**The key insight**: metrics told us WHAT was slow (the endpoint) and pointed to WHERE (inventory service). Traces showed us exactly WHICH operation in the call chain was the bottleneck. Logs explained WHY (missing index after migration). Each pillar answered a different question — as covered in question 12, this correlation is what makes observability powerful.

</details>

<details>
<summary>22. A downstream service is silently failing — it returns 200 OK but the response data is wrong, causing subtle bugs for users. Your error rates look normal and no alerts fired. Walk through how you'd detect and diagnose this class of problem — show the specific metric queries, trace filters, or log searches you'd use at each step, and what instrumentation you'd add to prevent it from being silent next time</summary>

This is the hardest class of production issues — **correctness failures**. Standard RED metrics (rate, errors, duration) only catch explicit errors (5xx) and latency problems. When a service returns 200 with wrong data, all your dashboards are green.

**How you typically discover it**: a user report, a QA bug, a downstream system processing incorrect data, or a business metric anomaly (e.g., order values don't match expected totals).

**Step 1: Reproduce and characterize the problem**

Start with what you know from the user report. Say users report seeing stale inventory counts — items show "in stock" but are actually sold out.

Check if it's widespread or isolated:

```promql
# If you have a correctness metric (you probably don't yet — that's the problem)
# Check business metrics instead:
sum(rate(orders_failed_insufficient_stock_total[1h]))
```

If this counter spiked, it confirms the inventory data being returned is stale/wrong. The correlation with time helps narrow the cause.

**Step 2: Trace a known-bad request**

If you have a specific failing request (from a user report with an order ID), search for its trace:

```
# In your log aggregator
{service="order-service"} |= "order_id=ORD-789"
```

Extract the trace ID and open it in your tracing UI. The trace shows:

```
POST /orders (250ms) — 200 OK
├── inventory-service.checkStock (30ms) — 200 OK, response: { available: true }
├── payment-service.charge (150ms) — 200 OK
└── database INSERT (40ms)
```

Everything looks normal — all spans returned 200. But you know the inventory data was wrong. Now inspect the inventory service span's response body (if you're logging response payloads) or check the inventory service's own traces.

**Step 3: Investigate the inventory service**

```
# Search inventory service logs around the same timestamp
{service="inventory-service"} |= "sku=WIDGET-42" | json
```

You might find: the inventory service is reading from a cache, and the cache entry is stale. Or it's reading from a read replica with replication lag. Or a recent deployment changed the response schema and the consuming service is parsing old-format responses.

**Step 4: Confirm the root cause**

```promql
# Check cache hit/miss ratios and cache age
inventory_cache_hit_ratio
inventory_cache_entry_age_seconds{quantile="0.99"}
```

If the p99 cache age is 3600 seconds (1 hour) when it should be 60 seconds, the TTL was misconfigured.

**What instrumentation to add to prevent silent correctness failures:**

1. **Response validation metrics**: in the consuming service, validate responses from downstream services and record correctness:

```typescript
const correctnessCounter = meter.createCounter('downstream_response_correctness_total');

const response = await inventoryService.checkStock(sku);

// Business logic validation — does this response make sense?
if (response.available && response.quantity <= 0) {
  correctnessCounter.add(1, { service: 'inventory', result: 'invalid' });
  logger.warn({ sku, response }, 'inventory response inconsistency');
} else {
  correctnessCounter.add(1, { service: 'inventory', result: 'valid' });
}
```

2. **Synthetic checks / contract tests**: periodically call the inventory service with known SKUs and verify the response matches the database. Alert when they diverge.

3. **Cache staleness metrics**: track the age of cached entries and alert when cache entries exceed the expected TTL.

4. **Cross-system reconciliation**: periodically compare data across systems (e.g., inventory counts in cache vs database) and alert on drift. This catches an entire class of correctness issues that per-request validation might miss.

5. **Add a correctness SLI** (as defined in question 19): `count(valid responses) / count(total responses)` with an SLO target, so correctness failures consume error budget and trigger burn-rate alerts.

</details>

<details>
<summary>23. Intermittent 5xx errors are affecting approximately 2% of requests to a specific endpoint, but the error only happens under certain conditions you haven't identified yet — walk through how you use distributed traces and correlation IDs to find the common pattern across failing requests and isolate the root cause</summary>

**Step 1: Confirm the scope and gather failing traces**

Start by confirming the 2% error rate and collecting a sample of failing requests:

```promql
# Error rate for the specific endpoint
sum(rate(http_requests_total{route="/orders", status=~"5.."}[5m]))
/ sum(rate(http_requests_total{route="/orders"}[5m]))
```

Confirmed: ~2% error rate, started gradually (not a sudden spike, so likely not a deployment).

In the tracing UI, query for failed traces:
- Service: `order-service`
- Operation: `POST /orders`
- Status: `ERROR`
- Time range: last 2 hours

Collect 20-30 failing traces. Also collect 20-30 successful traces from the same time period for comparison.

**Step 2: Compare failing vs successful traces — find the common pattern**

Look at the span attributes across failing traces and ask: what do the failing requests have in common that successful ones don't?

Check these dimensions:
- **Which downstream span fails?** If all failing traces have an error on the `payment-service` span, the issue is in payment processing.
- **Request attributes**: do failing requests share a specific `region`, `customer_tier`, `payment_method`, or `product_category`?
- **Timing patterns**: do failures cluster at specific seconds (e.g., every :00 and :30, suggesting a cron job competing for resources)?
- **Infrastructure**: do failures come from a specific pod/host? Check the `host.name` or `k8s.pod.name` span attributes.

Example finding: all 25 failing traces have `payment_method: "wire_transfer"`, while all successful traces use `credit_card` or `paypal`. Now you have the condition.

**Step 3: Use correlation IDs to get the full picture from logs**

Take the trace IDs from several failing requests and pull the complete log chain:

```
# Search across all services for failing trace IDs
{trace_id=~"abc123|def456|ghi789"} | json
```

The logs reveal:

```json
{"level":"error","service":"payment-service","msg":"payment provider timeout","provider":"wire-transfer-gateway","timeout_ms":3000,"trace_id":"abc123"}
{"level":"error","service":"payment-service","msg":"payment provider timeout","provider":"wire-transfer-gateway","timeout_ms":3000,"trace_id":"def456"}
```

Every failing request times out on the wire transfer payment provider. Successful requests use different providers. The wire transfer gateway is responding slowly or is partially down.

**Step 4: Verify the hypothesis**

```promql
# Outbound call latency to the wire transfer gateway
histogram_quantile(0.99,
  sum(rate(outbound_request_duration_ms_bucket{
    service="payment-service",
    target="wire-transfer-gateway"
  }[5m])) by (le)
)
```

The p99 latency to the wire transfer gateway is 3200ms (above the 3000ms timeout), confirming that ~2% of requests use wire transfer, and those requests are timing out because the external provider is degraded.

**Step 5: Fix and prevent**

- **Immediate**: increase the timeout or add a circuit breaker for the wire transfer gateway to fail fast and return a meaningful error instead of a 500
- **Short-term**: add specific monitoring for per-provider payment latency and error rates
- **Long-term**: ensure span attributes include `payment_method` so you can filter traces by this dimension without manually comparing 30 traces

The key technique here is **comparative analysis**: collect a sample of failing AND succeeding requests, then diff the attributes to find what's unique to failures. This works for any intermittent issue — the pattern is always there, you just need enough samples to surface it.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you set up or significantly improved observability for a production system — what was the state before, what did you implement, and how did it change the team's ability to detect and respond to issues?</summary>

**What the interviewer is looking for:**

- You understand observability as a system capability, not just "adding logging"
- You can articulate the gap between "we had logs" and "we could diagnose production issues"
- You made deliberate choices about what to instrument and why
- You can quantify the improvement (faster incident response, fewer user-reported issues, reduced MTTR)

**Suggested structure (STAR format):**

1. **Situation**: describe the system and its scale. What observability existed before? (Often: unstructured logs, no tracing, basic uptime monitoring, alerts based on host metrics)
2. **Task**: what triggered the improvement? (A bad incident, scaling pains, inability to debug cross-service issues, compliance requirements)
3. **Action**: what did you implement and why? Walk through the technical choices:
   - Structured logging with correlation IDs
   - Distributed tracing with OpenTelemetry
   - RED/USE metrics dashboards
   - SLO-based alerting
   - Health check endpoints for Kubernetes
4. **Result**: how did the team's capability change? Use concrete metrics.

**Example outline to personalize:**

"We had a microservices architecture with 8 services handling order processing. Observability was basic — each service logged to stdout in plain text, we had Prometheus scraping default Node.js metrics, and alerting was CPU/memory threshold-based.

After a 3-hour incident where we couldn't trace a data inconsistency across services (we eventually found it by correlating timestamps in 4 different log streams manually), I proposed and led an observability improvement initiative.

I implemented: (1) structured JSON logging with pino, including correlation IDs propagated via AsyncLocalStorage across all services, (2) OpenTelemetry auto-instrumentation with a shared collector pipeline, (3) RED method dashboards per service, and (4) replaced CPU/memory alerts with SLO-based burn-rate alerts.

The result: our median time to diagnose incidents dropped from 45 minutes to under 10 minutes. The next cross-service issue — a similar data inconsistency — was traced in 8 minutes by following the trace ID. We also caught 3 latency regressions via SLO alerts before any users reported them."

**Key points to hit:**

- Emphasize the "before and after" — the interviewer wants to see the delta you created
- Show that you made tradeoffs (e.g., chose head sampling at 10% to control costs, skipped custom metrics for low-traffic services)
- Mention any pushback and how you handled it (cost concerns, team adoption, learning curve)
- If you haven't done a full observability overhaul, you can talk about a smaller improvement — adding structured logging, implementing correlation IDs, or creating a specific dashboard that changed how the team debugged issues

</details>

<details>
<summary>25. Describe a time you used observability tooling (metrics, traces, logs) to diagnose a complex production incident that would have been very difficult to solve without it — what were the symptoms, how did you correlate signals, and what was the root cause?</summary>

**What the interviewer is looking for:**

- You can walk through a structured debugging process under pressure
- You used multiple signals (not just "I grepped the logs") and correlated them
- The problem was genuinely complex — not a simple crash with a stack trace
- You can explain why the tooling mattered (what would the alternative have been without it?)

**Suggested structure:**

1. **Symptoms**: what was observed? (Alert fired, user reports, dashboard anomaly)
2. **Initial hypothesis and why it was wrong**: show that the problem wasn't obvious
3. **The debugging path**: step through metrics → traces → logs (or whatever order you actually used). For each step, explain what you learned and what it ruled out.
4. **Root cause**: what was actually wrong?
5. **Why observability was critical**: what would have happened without tracing/correlation? How long would it have taken?

**Example outline to personalize:**

"Our order processing pipeline had intermittent failures — about 1 in 50 orders would silently fail to send a confirmation email, but the order itself was created successfully. No errors in the order service logs, no 5xx responses, and the error rate dashboards were green.

We only caught it because a customer complained they never received a confirmation. I started by checking the email service logs for the customer's order ID, but found nothing — the email service never received the request.

I pulled the trace for that order's correlation ID and found the full request chain: order-service → event-bus → email-service. The trace showed the event was published to the message queue, but the email-service span was missing. The event was published but never consumed.

Checking the message queue metrics, I found a consumer lag spike at the same timestamps. The email service consumer was falling behind. Logs from the email service showed: the consumer was crashing on malformed events from a different producer (a recently deployed notification service), and the crash caused the consumer to restart and skip the next few messages due to an auto-commit offset configuration.

Without distributed tracing, I would have been stuck checking each service independently. The trace showed me the gap in the chain — the missing span — which pointed directly at the message queue as the boundary where things broke. That narrowed 8 services to 1 specific interaction in under 10 minutes."

**Key points to hit:**

- The problem should be non-trivial — something that crossed service boundaries or had a non-obvious cause
- Show the progression of your investigation, not just the answer
- Mention what the alternative would have been ("without the trace, we would have had to check each service's logs independently, correlating by timestamp across 5 log streams")
- End with any preventive measures you added afterward (dead letter queue, consumer health checks, better alerting)

</details>

<details>
<summary>26. Tell me about a time you dealt with alert fatigue or redesigned an alerting strategy — what was wrong with the existing alerts, what changes did you make, and how did you measure whether the new approach was better?</summary>

**What the interviewer is looking for:**

- You understand that alerting is a system that requires maintenance, not a "set and forget" configuration
- You can identify the symptoms of bad alerting (fatigue, ignored pages, missing coverage)
- You took a systematic approach to fixing it, not just deleting alerts randomly
- You measured the outcome — this is the hardest part and separates strong answers from generic ones

**Suggested structure:**

1. **The problem**: describe the state of alerting and its impact on the team
2. **Root cause analysis**: why was it bad? (Not just "too many alerts" — what specifically was wrong?)
3. **The redesign**: what framework or principles did you apply?
4. **Measurement**: how did you know the new approach was better?

**Example outline to personalize:**

"Our on-call rotation had a reputation problem — engineers dreaded it. Over a 4-week period, I tracked that the on-call engineer received an average of 47 pages per week. After reviewing the alerts, I found:

- 60% were cause-based (CPU > 70%, memory > 80%) that almost always resolved themselves within minutes
- 15% were flapping alerts with thresholds too close to normal operating ranges
- 10% were duplicate alerts — the same database slowdown triggered alerts in 4 services
- Only 15% were genuinely actionable and required human intervention

I proposed a redesign based on three principles from the concepts in question 11: (1) alert on symptoms, not causes, (2) add severity tiers with different notification channels, (3) every alert must have a runbook.

The changes: I deleted all cause-based host metric alerts (moved them to dashboards), consolidated the 4 duplicate database alerts into 1 symptom-based alert at the API gateway level, added `for` durations to prevent flapping, and introduced P1/P2/P3 tiers with PagerDuty pages only for P1.

I measured the outcome over the next month: pages per week dropped from 47 to 8, and the page-to-action ratio (pages that required real intervention) went from 15% to 75%. We also tracked incidents-without-alerts — we had 2 user-reported issues with no corresponding alert, so we added coverage for those scenarios. After 3 months, the on-call satisfaction survey score went from 2.1/5 to 4.0/5."

**Key points to hit:**

- Quantify the before and after — number of alerts, page-to-action ratio, on-call burden
- Show you used a principled approach (symptom-based alerting, severity tiers) not just intuition
- Mention that you kept iterating — alerting is never "done," you reviewed it monthly
- If you haven't done a full redesign, talk about a smaller improvement: fixing a flapping alert, adding a `for` duration, creating severity tiers for a single service, or deleting an alert that everyone ignored and measuring whether anything bad happened

</details>

