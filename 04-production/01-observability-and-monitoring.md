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

<!-- Answer will be added later -->

</details>

<details>
<summary>2. What are the RED method (Rate, Errors, Duration) and USE method (Utilization, Saturation, Errors) — what does each measure, which types of components does each apply to (request-driven services vs infrastructure resources), and how do you decide which method to use when instrumenting a new system?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What are SLIs, SLOs, SLAs, and error budgets — how do they relate to each other, why does defining them change how a team operates (feature velocity vs reliability tradeoffs), and what goes wrong when teams set SLOs too aggressively or too loosely?</summary>

<!-- Answer will be added later -->

</details>

## Structured Logging — Concepts

<details>
<summary>4. Why does structured logging (JSON format) matter over plain-text logs — what does it enable for log aggregation, querying, and alerting that unstructured logs make difficult or impossible, and what are the tradeoffs (readability in development, log size, serialization cost)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why do distributed systems need correlation IDs, how are they generated and propagated across service boundaries (HTTP headers, gRPC metadata), and how do you use them to reconstruct a request's journey across multiple services when debugging a production issue?</summary>

<!-- Answer will be added later -->

</details>

## Metrics — Concepts

<details>
<summary>6. How do you choose between counter, gauge, and histogram metric types — what does each represent, what goes wrong when you use the wrong type (e.g., a gauge for a cumulative value or a counter for a point-in-time measurement), and when would you reach for a histogram over a simple counter?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What are the principles for designing metric labels — why does high cardinality matter, what happens to your metrics backend when labels like user ID or request path are unbounded, and how do you decide which dimensions are worth labeling vs which should be logged instead?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do you decide what to measure in a production service — what signals actually matter for understanding system health vs vanity metrics that look good on dashboards but don't drive action, and how do you balance comprehensive instrumentation against metric explosion and storage costs?</summary>

<!-- Answer will be added later -->

</details>

## Distributed Tracing — Concepts

<details>
<summary>9. Why does distributed tracing exist as a separate pillar from logs and metrics — what is a span, how does context propagation work (W3C Trace Context standard), and what kinds of production problems can you only diagnose with traces (that logs and metrics alone can't answer)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why can't you trace 100% of requests in production, what are the tradeoffs between head-based sampling and tail-based sampling, and how do you choose a sampling strategy that captures enough data for debugging without overwhelming your tracing backend with cost and volume?</summary>

<!-- Answer will be added later -->

</details>

## Alerting & SLOs — Concepts

<details>
<summary>11. What makes an alert actionable vs noisy — why should alerts be based on symptoms (user-facing impact) rather than causes (CPU spike), how do severity tiers work in practice, and what patterns lead to alert fatigue where the on-call engineer starts ignoring pages?</summary>

<!-- Answer will be added later -->

</details>

## Error Tracking & Debugging — Concepts

<details>
<summary>12. How do you correlate metrics, traces, and logs together to debug complex production issues — what links them (trace IDs, timestamps, shared labels), why is this correlation the highest-value capability of an observability stack, and what breaks when one pillar is missing or disconnected from the others?</summary>

<!-- Answer will be added later -->

</details>

## OpenTelemetry — Concepts

<details>
<summary>13. Why did OpenTelemetry emerge as the observability standard — what problems did fragmented vendor-specific SDKs (Jaeger client, Prometheus client, vendor agents) create, how does the OTel architecture (SDK + Collector) solve vendor lock-in, and what does "vendor-neutral instrumentation" actually mean in practice for a team choosing an observability backend?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Instrumentation & Configuration

<details>
<summary>14. Configure structured logging with pino in a Node.js service — show the setup with appropriate log levels, sensitive field redaction (passwords, tokens), log injection prevention (how attackers exploit unsanitized user input in log fields to forge entries or break log parsing), child loggers for request-scoped context, and explain why each configuration choice matters for production use</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Instrument a Node.js service with OpenTelemetry — show the setup for auto-instrumentation (HTTP, database calls), add a manual span for a custom business operation, configure the collector pipeline (exporter, processor, batching), and explain what each piece does and what you lose without it</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Implement correlation ID middleware for a Node.js HTTP service — show how to generate an ID for incoming requests (or extract one from upstream headers), propagate it through async context so all log lines include it, pass it to downstream HTTP/gRPC calls, and explain what breaks in cross-service debugging without this</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. Instrument a Node.js API with custom metrics — create a request duration histogram, an active connections gauge, and a requests-processed counter with appropriate labels. Show the code, explain the label choices, and demonstrate what high-cardinality labels (e.g., user ID as a label) would do to your metrics backend</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Implement /health and /ready endpoints for a Node.js service that checks downstream dependencies (database, cache, external API) — show the code, explain the difference between liveness and readiness semantics, show the Kubernetes probe YAML that integrates with these endpoints, and explain what happens when readiness checks are too aggressive or too lenient</summary>

<!-- Answer will be added later -->

</details>

## Practical — Production Operations

<details>
<summary>19. Define concrete SLIs for a REST API service (availability, latency, correctness), set SLO targets with specific numbers, calculate the monthly error budget, and show how burn-rate alert thresholds are derived — walk through the math and explain how the team should respond when the error budget is nearly exhausted</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. Design an alerting strategy for a microservices application — show example alert rules that follow symptom-based alerting principles, define severity tiers (page vs ticket vs dashboard), set up routing and escalation, and explain how you'd evaluate whether the alerting is working well after a month in production</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>21. An API endpoint's p99 latency spiked from 200ms to 2s starting 30 minutes ago — walk through step by step how you use metrics to identify which endpoint and when, traces to find the slow span, and logs to find the root cause. Show the specific queries and reasoning at each step</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. A downstream service is silently failing — it returns 200 OK but the response data is wrong, causing subtle bugs for users. Your error rates look normal and no alerts fired. Walk through how you'd detect and diagnose this class of problem — show the specific metric queries, trace filters, or log searches you'd use at each step, and what instrumentation you'd add to prevent it from being silent next time</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Intermittent 5xx errors are affecting approximately 2% of requests to a specific endpoint, but the error only happens under certain conditions you haven't identified yet — walk through how you use distributed traces and correlation IDs to find the common pattern across failing requests and isolate the root cause</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you set up or significantly improved observability for a production system — what was the state before, what did you implement, and how did it change the team's ability to detect and respond to issues?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Describe a time you used observability tooling (metrics, traces, logs) to diagnose a complex production incident that would have been very difficult to solve without it — what were the symptoms, how did you correlate signals, and what was the root cause?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>26. Tell me about a time you dealt with alert fatigue or redesigned an alerting strategy — what was wrong with the existing alerts, what changes did you make, and how did you measure whether the new approach was better?</summary>

<!-- Answer framework will be added later -->

</details>

