# Prometheus, Grafana & Loki

> **27 questions** — 13 theory, 14 practical

- PLG stack (Prometheus, Loki, Grafana) — why metrics, logs, and visualization are separate tools
- Prometheus architecture — pull model, service discovery, scrape config, storage retention
- Metric types — counter, gauge, histogram, summary; when to use each, common misuse pitfalls, histogram bucket design, cross-instance aggregation
- Label cardinality — why it's the biggest operational risk, detection, prevention, real examples
- PromQL — rate() vs irate(), aggregation operators (sum, avg, max by), vector matching (on, ignoring, group_left)
- PromQL for RED method — request rate, error percentage, p95/p99 latency with histogram_quantile()
- Alerting rules — high error rate, high latency, predict_linear for disk, for duration, reducing false positives
- Alertmanager — routing tree, grouping, throttling, inhibition, silencing, notification channels
- prom-client for Node.js — /metrics endpoint, default metrics, custom histograms/counters, label cardinality risks
- Grafana dashboard design — USE/RED structure, drill-down hierarchy, panel types (time series, stat, heatmap), template variables for multi-service views, antipatterns
- Loki architecture and tradeoffs — labels-only indexing vs full-text (ELK/Splunk), why it's cheaper at scale, comparison with ELK/Datadog/CloudWatch on cost, query capability, and operational complexity
- LogQL — label filtering, regex, field extraction (pattern/json parsers), metric queries (rate of errors)
- Debugging monitoring infrastructure — missing metrics, broken alerts, slow dashboards, cardinality explosions, Loki query failures

---

## Foundational

<details>
<summary>1. Why does the PLG stack (Prometheus, Loki, Grafana) separate metrics collection, log aggregation, and visualization into three distinct tools instead of using a single unified platform — what problem does each tool solve, how do they connect to form a complete observability stack, and what are the tradeoffs of this composable approach vs an all-in-one solution like Datadog or New Relic?</summary>

<!-- Answer will be added later -->

</details>

## Prometheus — Concepts

<details>
<summary>2. Why does Prometheus use a pull-based model where it scrapes targets on a schedule instead of having applications push metrics — what advantages does this give for reliability and service discovery, how does the scrape lifecycle work (discovery, scrape, ingestion, storage), and what are Prometheus's storage retention characteristics and limitations that affect how you architect a monitoring stack?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What are the four Prometheus metric types (counter, gauge, histogram, summary), why does each exist for a different kind of measurement, and what are the common misuse pitfalls -- for example, what breaks when you use a gauge where you should use a counter, or when you call rate() on a gauge?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. How should you design histogram bucket boundaries for latency metrics — why do the default buckets often fail in production, how does bucket choice affect query accuracy and storage cost, and what challenges arise when you need to aggregate histograms across multiple instances (why can't you just average percentiles)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Why is label cardinality the single biggest operational risk in Prometheus — what happens to Prometheus when cardinality explodes, how do you detect it before it causes an outage, what are real-world examples of innocent-looking labels that destroy performance (user IDs, request paths, pod names), and what prevention strategies should every team adopt?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How does PromQL handle rate calculations and aggregations -- explain rate() vs irate() (when each is appropriate and why rate() is almost always what you want), and how do aggregation operators like sum/avg/max work with the "by" and "without" clauses to slice metrics across dimensions?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. How does PromQL vector matching work when you need to combine metrics from different sources — explain the 'on' and 'ignoring' keywords, what 'group_left' and 'group_right' do for many-to-one joins, and what are the common mistakes that cause 'many-to-many matching not allowed' errors?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How do you design alerting rules that are actually useful in production — why does the "for" duration clause exist and how does it reduce false positives, how does predict_linear() work for proactive alerts like disk space, and what patterns distinguish good alerts (actionable, low noise) from bad ones (alert fatigue, flapping, too sensitive)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. How does Alertmanager's routing and notification pipeline work — explain the routing tree (how alerts are matched to receivers), what grouping does and why it prevents notification storms, how throttling (group_wait, group_interval, repeat_interval) controls notification frequency, and when you'd use inhibition rules vs silences?</summary>

<!-- Answer will be added later -->

</details>

## Grafana — Concepts

<details>
<summary>10. What makes a well-designed Grafana dashboard vs a wall of random graphs -- how do the USE method (utilization, saturation, errors) and RED method (rate, errors, duration) provide structure for organizing panels, what drill-down hierarchy should dashboards follow (overview to detail), and which panel types (time series, stat, heatmap, table) suit which kinds of data?</summary>

<!-- Answer will be added later -->

</details>

## Loki — Concepts

<details>
<summary>11. How does Loki's architecture fundamentally differ from Elasticsearch/ELK by using labels-only indexing instead of full-text indexing — what does "index labels, not log content" mean in practice, how does this architectural choice make Loki dramatically cheaper to operate at scale, and what query patterns does it make harder compared to full-text search?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. How does LogQL work for querying logs in Loki — explain label-based filtering, regex line filters, structured field extraction using pattern and JSON parsers, and how metric queries (like counting error rates from logs) bridge the gap between logs and metrics?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Configuration & Instrumentation

<details>
<summary>13. Set up a Prometheus scrape configuration for a Kubernetes environment — show the YAML for scraping application pods using service discovery (kubernetes_sd_configs), explain how relabel_configs work to filter targets and rewrite labels, configure an appropriate scrape interval and storage retention, and explain what breaks if you set the scrape interval too low or retention too high for your storage.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Instrument a Node.js application using prom-client — show the TypeScript code to expose a /metrics endpoint, enable default metrics (event loop lag, heap size, GC), create custom histograms for HTTP request duration and custom counters for business events, and explain the label cardinality risks that catch most teams off guard (like using request path or user ID as labels).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Write the PromQL queries that implement the RED method for a typical HTTP service — show queries for request rate, error percentage (as a ratio of total requests), and p95/p99 latency using histogram_quantile(). Explain why histogram_quantile() requires rate() inside it, what the "le" label means, and what happens to accuracy when bucket boundaries don't align with actual latency distribution.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Write alerting rules for three production scenarios: high error rate (>5% of requests returning 5xx), high p99 latency (>2s for 5 minutes), and disk filling up within 4 hours using predict_linear() — show the YAML for each rule, explain how the "for" duration prevents flapping, and what labels and annotations you'd add to make the alert actionable when it fires.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Alerting, Dashboards & Logs

<details>
<summary>17. Configure Alertmanager's routing tree for a multi-team setup — show the YAML where critical alerts go to PagerDuty, warning alerts go to Slack grouped by service, and a catch-all default route. Configure group_by, group_wait, group_interval, and repeat_interval with appropriate values, add an inhibition rule where a cluster-down alert suppresses individual service alerts, and explain the routing match logic.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Build a Grafana dashboard for a production service using the RED method — describe the panel layout and drill-down hierarchy (overview row with stat panels for current rate/error%/p99, then time series panels for trends, then heatmap for latency distribution), show how to configure template variables so the same dashboard works across all services and environments, and explain what dashboard antipatterns to avoid (too many panels, no drill-down, missing units/thresholds).</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>19. A service's metrics suddenly disappear from Prometheus — walk through the exact steps to diagnose why: checking the Prometheus targets page, verifying the scrape config and service discovery, testing the /metrics endpoint directly, inspecting relabel_configs for dropped targets, and checking for scrape errors. What are the most common causes of missing metrics and how do you fix each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. An alert that should be firing isn't, or an alert is firing constantly when it shouldn't — walk through the debugging process: testing the PromQL expression in the Prometheus UI, checking the "for" duration and whether the condition was sustained, inspecting Alertmanager's routing to verify the alert reaches the right receiver, checking for silences or inhibition rules suppressing it, and verifying notification channel configuration. Show the specific pages and API endpoints you'd check.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. Grafana dashboards are loading slowly or timing out — diagnose whether the problem is in Grafana, the data source (Prometheus/Loki), or the queries themselves. Walk through checking query execution time in Prometheus, identifying queries that scan too much data (missing label filters, long time ranges without recording rules), using the Grafana query inspector, and explain when the fix is query optimization vs recording rules vs adjusting dashboard time range defaults.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. Prometheus is consuming excessive memory and disk, and you suspect a cardinality explosion — walk through how to detect which metrics and labels are causing the problem (using TSDB status page, promtool, or PromQL queries like count of time series per metric), how to identify the offending labels, and how to mitigate the issue both immediately (dropping labels via relabel_configs, deleting series) and long-term (metric naming conventions, cardinality limits, CI checks).</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Loki queries are returning no results or timing out — walk through the debugging process: verifying logs are being ingested (checking Promtail/Alloy pipeline, Loki /ready and /metrics endpoints), testing label selectors vs line filters, diagnosing query timeouts caused by scanning too many streams (wrong label strategy forcing full scans), and explain how to restructure queries or labels to fix performance.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you designed or significantly overhauled a monitoring and alerting setup — what was the existing state, what tools did you choose and why, how did you decide what to monitor and alert on, and what impact did it have on incident response?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>25. Describe a time you dealt with a cardinality explosion or a scaling issue in your metrics infrastructure — how did you discover it, what was the root cause, how did you fix it, and what guardrails did you put in place to prevent it from happening again?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>26. Tell me about a time you used metrics, logs, and dashboards together to debug a production incident — walk through the investigation flow, which signals led you to the root cause, and how the observability setup helped (or failed) you.</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>27. Describe a time you tackled alert fatigue on your team — what was causing it, how did you measure the problem, what changes did you make to alerting rules or routing, and how did you know the improvements worked?</summary>

<!-- Answer framework will be added later -->

</details>
