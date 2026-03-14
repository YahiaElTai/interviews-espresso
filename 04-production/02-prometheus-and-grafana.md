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

Each tool solves one problem well:

- **Prometheus** collects and stores numeric time-series metrics. It answers "how many," "how fast," "how much" -- quantitative signals about system behavior over time.
- **Loki** aggregates and indexes logs. It answers "what happened" -- the qualitative, event-level detail you need when metrics tell you something is wrong but not why.
- **Grafana** visualizes data from both (and many other sources). It provides dashboards, exploration, and correlation across metrics and logs in one UI.

**How they connect:** Prometheus scrapes `/metrics` endpoints and stores time-series data. Loki receives log streams from agents (Promtail, Grafana Alloy). Grafana queries both as data sources, and you can jump from a metric spike in a Prometheus panel directly to the corresponding logs in Loki using shared labels (service name, pod, namespace). Grafana also handles alerting evaluation and can route alerts through Alertmanager.

**Tradeoffs vs all-in-one (Datadog, New Relic):**

| | PLG (composable) | All-in-one (Datadog) |
|---|---|---|
| **Cost** | Free/open-source, pay only for infrastructure | Per-host/per-GB pricing that scales expensively |
| **Flexibility** | Swap any component (e.g., Thanos for long-term storage, Mimir for multi-tenant Prometheus) | Locked into vendor's capabilities and pricing |
| **Operational burden** | You run, upgrade, and scale each component yourself | Fully managed -- vendor handles reliability |
| **Correlation** | Manual setup of shared labels, data source linking | Built-in correlation between metrics, logs, traces |
| **Time to value** | Days/weeks to set up properly | Hours -- install agent, get dashboards |

**When to choose PLG:** Your team has 2+ engineers who can own the infra, and you need to customize or scale beyond vendor pricing. **When to choose all-in-one:** Small team with no dedicated platform/SRE resources.

</details>

## Prometheus — Concepts

<details>
<summary>2. Why does Prometheus use a pull-based model where it scrapes targets on a schedule instead of having applications push metrics — what advantages does this give for reliability and service discovery, how does the scrape lifecycle work (discovery, scrape, ingestion, storage), and what are Prometheus's storage retention characteristics and limitations that affect how you architect a monitoring stack?</summary>

**Why pull, not push:**

- **Prometheus controls the pace.** It decides when and how often to scrape, so a burst of new instances can't overwhelm the metrics system with pushes.
- **Target health is free.** If a scrape fails, Prometheus knows the target is down -- you get `up == 0` without any extra health-check logic. With push, a silent target is ambiguous (is it healthy and idle, or dead?).
- **Service discovery integration.** Prometheus discovers targets dynamically (Kubernetes API, Consul, DNS, EC2) and starts scraping them automatically. No application-side configuration telling it where to push.
- **Simpler applications.** Apps just expose an HTTP endpoint with current metric values. No SDK required to push on a schedule, no retry logic, no batching.

**The scrape lifecycle:**

1. **Discovery** -- Service discovery providers (e.g., `kubernetes_sd_configs`) produce a list of targets with metadata labels (pod name, namespace, annotations).
2. **Relabeling** -- `relabel_configs` filter, rename, or drop targets before scraping. This is where you use annotations like `prometheus.io/scrape: "true"` to opt in.
3. **Scrape** -- Prometheus sends an HTTP GET to each target's metrics endpoint (default `/metrics`) at the configured `scrape_interval` (typically 15-30s). The response is plaintext in the Prometheus exposition format.
4. **Ingestion** -- Scraped samples get timestamps and are written to the local TSDB (time-series database). Each unique combination of metric name + labels = one time series.
5. **Storage** -- TSDB stores data in 2-hour blocks on disk, compacted over time. Retention is configured by time (`--storage.tsdb.retention.time`, default 15 days) or size (`--storage.tsdb.retention.size`).

**Storage limitations that shape architecture:**

- **Local-only by default.** Prometheus stores on a single node's disk. Lose the node, lose the data. This is why production stacks add Thanos or Mimir for durable long-term storage.
- **No built-in clustering.** You can't horizontally scale a single Prometheus instance. For large environments, you shard by team/service and federate, or use Thanos/Mimir.
- **Retention vs. disk.** Each active time series consumes ~1-2 bytes per sample. At 15s scrape interval, 10K series = ~50GB/month. Long retention on high-cardinality data fills disks fast.
- **No downsampling.** Prometheus keeps full-resolution data for the retention period. Thanos adds downsampling (5m, 1h) for efficient long-range queries.

</details>

<details>
<summary>3. What are the four Prometheus metric types (counter, gauge, histogram, summary), why does each exist for a different kind of measurement, and what are the common misuse pitfalls -- for example, what breaks when you use a gauge where you should use a counter, or when you call rate() on a gauge?</summary>

**Counter** -- A monotonically increasing value that only goes up (or resets to zero on process restart). Use for things you count: total requests, total errors, total bytes sent.

```
http_requests_total{method="GET", status="200"} 14832
```

Key property: `rate()` and `increase()` work correctly on counters because they detect resets. When the process restarts and the counter goes from 14832 to 0, `rate()` handles this gracefully instead of computing a massive negative spike.

**Gauge** -- A value that can go up or down. Use for current state: temperature, queue depth, active connections, memory usage.

```
node_memory_available_bytes 4294967296
```

Key property: You read the current value directly. `avg_over_time()`, `max_over_time()`, `min_over_time()` are the typical functions.

**Histogram** -- Counts observations (like request latency) into configurable buckets, plus a sum and count. Produces multiple time series with the `le` (less than or equal) label.

```
http_request_duration_seconds_bucket{le="0.1"}  24054
http_request_duration_seconds_bucket{le="0.5"}  33421
http_request_duration_seconds_bucket{le="1"}    34065
http_request_duration_seconds_bucket{le="+Inf"} 34079
http_request_duration_seconds_sum               8547.2
http_request_duration_seconds_count             34079
```

Key property: Buckets are cumulative (each includes all smaller buckets). This enables `histogram_quantile()` for percentile calculations and, critically, histograms are aggregatable across instances -- you can `sum` buckets from 10 pods and compute a meaningful cluster-wide p99.

**Summary** -- Similar to histogram but calculates quantiles client-side (in the application process). Produces pre-computed quantiles like `{quantile="0.99"}`.

Key property: Pre-computed quantiles **cannot be aggregated** across instances. If 10 pods each report their own p99, you cannot combine them into a cluster-wide p99 -- averaging percentiles is mathematically wrong. This is why histograms are almost always preferred over summaries in practice.

**Common misuse pitfalls:**

- **Gauge instead of counter for requests:** If you track "requests in the last minute" as a gauge, you lose the ability to use `rate()`. You also lose data between scrapes -- if 1000 requests happen between two 15s scrapes, a gauge snapshot might show only the current moment's value. A counter captures every single request.
- **`rate()` on a gauge:** `rate()` computes per-second increase assuming monotonic growth with resets. On a gauge that naturally decreases (like active connections dropping), `rate()` produces nonsensical results -- it may interpret a decrease as a counter reset and compute a huge spike.
- **Counter for something that decreases:** Queue depth as a counter makes no sense -- it goes up and down. Use a gauge.
- **Summary when you need cross-instance aggregation:** If you have multiple replicas and need cluster-wide percentiles, summaries force you into the "averaging percentiles" trap. Use histograms.

</details>

<details>
<summary>4. How should you design histogram bucket boundaries for latency metrics — why do the default buckets often fail in production, how does bucket choice affect query accuracy and storage cost, and what challenges arise when you need to aggregate histograms across multiple instances (why can't you just average percentiles)?</summary>

**Why default buckets fail:**

The default Prometheus histogram buckets are: `[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]` seconds. These are generic and often misaligned with actual latency distributions:

- **Fast APIs (sub-10ms):** Most requests fall into the first 1-2 buckets, giving you almost no resolution. The p50 and p99 both land in `le="0.01"`, so you can't distinguish them.
- **Slow APIs (multi-second):** If your API routinely takes 3-8 seconds, there's only one bucket between 2.5 and 5 -- no resolution at all in the range that matters.

**How to design good buckets:**

1. **Measure first.** Look at actual latency distribution before choosing. Run in production with default buckets temporarily, then refine.
2. **Concentrate buckets around your SLO.** If your SLO is p99 < 500ms, you need fine granularity around 200-800ms. Example: `[0.01, 0.025, 0.05, 0.1, 0.2, 0.3, 0.5, 0.75, 1, 2, 5]`.
3. **Use exponential spacing** where you don't have specific targets: each bucket roughly 2-3x the previous.
4. **Keep bucket count reasonable.** Each bucket creates a separate time series per label combination. 10-15 buckets is typical; 30+ starts adding significant cardinality.

**Storage cost impact:**

Every histogram metric produces `N_buckets + 2` time series (one per bucket, plus `_sum` and `_count`) for each unique label combination. With 15 buckets and 4 label dimensions with modest cardinality, you can easily generate thousands of time series from a single histogram. Doubling your buckets doubles that cost.

**Cross-instance aggregation -- why averaging percentiles is wrong:**

Suppose you have 2 instances. Instance A has p99 = 100ms (handles mostly fast requests). Instance B has p99 = 2000ms (handles slow batch requests). The "average p99" would be 1050ms, but the real cluster-wide p99 could be anything -- it depends on the combined distribution.

Histograms solve this because buckets are additive. You can `sum` the bucket counts across all instances, then compute `histogram_quantile()` on the aggregated buckets to get a mathematically correct cluster-wide percentile:

```promql
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)
```

This works because you're reconstructing the combined distribution from the summed bucket counts, not averaging pre-computed quantiles. The accuracy depends on bucket granularity -- if the real p99 falls between two widely-spaced buckets, Prometheus linearly interpolates, which can be significantly off.

</details>

<details>
<summary>5. Why is label cardinality the single biggest operational risk in Prometheus — what happens to Prometheus when cardinality explodes, how do you detect it before it causes an outage, what are real-world examples of innocent-looking labels that destroy performance (user IDs, request paths, pod names), and what prevention strategies should every team adopt?</summary>

**Why cardinality is the #1 risk:**

Every unique combination of metric name + label key-value pairs = one time series. Prometheus keeps all active time series in memory for fast ingestion and querying. Cardinality is multiplicative -- if you have labels `method` (5 values), `status` (5 values), and `path` (1000 values), that's 25,000 series from one metric. Add a `user_id` label with 100K users and you're at 2.5 billion series.

**What happens when cardinality explodes:**

1. **Memory exhaustion.** Each active series consumes ~1-3KB in memory. 10M series = ~10-30GB RAM. Prometheus OOMs.
2. **Ingestion slowdown.** The TSDB index grows, writes take longer, scrapes start timing out.
3. **Query failures.** Queries that touch high-cardinality metrics scan millions of series, timing out or consuming all available CPU.
4. **Compaction failures.** Background compaction can't keep up with the volume, causing disk usage to spiral.
5. **Cascading failure.** Prometheus falls behind on scrapes, `up` metrics go stale, monitoring of everything goes blind.

**Real-world examples of innocent labels that destroy performance:**

- **`path="/api/users/abc123"`** -- Unbounded. Every unique URL path creates new series. REST APIs with IDs in paths are the #1 offender.
- **`user_id="..."`** -- Millions of users = millions of series per metric.
- **`request_id` or `trace_id`** -- Unique per request. This creates a new series for every single request and will kill Prometheus in minutes.
- **`pod` or `container_id`** in high-churn environments -- Rolling deployments create new pod names constantly, and old series stay in memory until they go stale (default 5 minutes, but the TSDB keeps them until the next block cut).
- **`error_message`** -- Stack traces or dynamic error messages as label values. Hundreds of unique strings.

**Detection before outage:**

- **TSDB status page** (`/api/v1/status/tsdb`) -- shows top metrics by series count and top label pairs by cardinality.
- **PromQL queries:**
  ```promql
  # Total active series
  prometheus_tsdb_head_series

  # Series count per metric (find the expensive ones)
  count by (__name__) ({__name__=~".+"})

  # Rate of new series creation (sudden spike = cardinality bomb)
  rate(prometheus_tsdb_head_series_created_total[5m])
  ```
- **Alert on it:**
  ```yaml
  - alert: HighCardinality
    expr: prometheus_tsdb_head_series > 2000000
    for: 10m
    annotations:
      summary: "Prometheus has {{ $value }} active series"
  ```

**Prevention strategies:**

1. **Normalize paths.** Strip IDs: `/api/users/:id` not `/api/users/abc123`. Do this in instrumentation code, not relabel_configs.
2. **Allowlist labels.** Only permit known, bounded label values. Reject or bucket unknown values into "other."
3. **Review in PR.** Any new metric or label should be reviewed for cardinality impact before merging.
4. **Use `metric_relabel_configs`** to drop dangerous labels at scrape time as a safety net, and use a metrics proxy (like Grafana Agent or prom-client middleware) that enforces per-metric cardinality limits. Prometheus itself has no built-in per-metric cardinality cap, which is exactly why external enforcement matters.
5. **Separate high-cardinality data.** If you truly need per-user metrics, send those to a different system (like ClickHouse or a dedicated high-cardinality TSDB) -- not Prometheus.

</details>

<details>
<summary>6. How does PromQL handle rate calculations and aggregations -- explain rate() vs irate() (when each is appropriate and why rate() is almost always what you want), and how do aggregation operators like sum/avg/max work with the "by" and "without" clauses to slice metrics across dimensions?</summary>

**`rate()` vs `irate()`:**

Both compute per-second rates from counter metrics, but they use different data points:

- **`rate(metric[5m])`** -- Uses the first and last data points in the 5-minute window to compute the average per-second rate. Smooths out spikes. If there are counter resets in the window, it detects and corrects for them.
- **`irate(metric[5m])`** -- Uses only the last two data points in the window to compute an instantaneous rate. Shows sudden spikes and drops. The window (`[5m]`) only determines how far back to look for those two points -- it doesn't average over 5 minutes.

**Why `rate()` is almost always what you want:**

- **Alerting.** Alerts need stable signals. `irate()` is noisy -- a single slow scrape or one burst of traffic causes massive spikes that trigger false alerts.
- **Dashboards.** `rate()` gives you the trend you're actually interested in. `irate()` shows a jagged line that's hard to interpret.
- **Aggregation.** `rate()` produces smooth, representative values that aggregate well across instances. `irate()` values across instances are correlated to different scrape moments, making sums less meaningful.

**When `irate()` is useful:** Real-time debugging dashboards where you need to see the exact spike, not the smoothed average. Auto-refresh dashboards with short time ranges. Never for alerting.

**Aggregation operators:**

`sum`, `avg`, `max`, `min`, `count`, `stddev`, `topk`, `bottomk` all aggregate across series that share the same label set.

**`by` -- keep only these labels** (group by them, drop everything else):

```promql
# Total request rate per service (drop instance, pod, method, etc.)
sum(rate(http_requests_total[5m])) by (service)
```

**`without` -- drop these labels** (keep everything else):

```promql
# Total request rate, removing instance-level detail
sum(rate(http_requests_total[5m])) without (instance, pod)
```

`by` and `without` are inverses. Use `by` when you want a specific grouping. Use `without` when you want to remove a few dimensions but keep the rest.

**Common patterns:**

```promql
# Error percentage by service
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)

# Max memory across all pods of a service
max(container_memory_usage_bytes) by (service)

# Average latency per endpoint
rate(http_request_duration_seconds_sum[5m])
/
rate(http_request_duration_seconds_count[5m])
```

**Gotcha:** If you aggregate with `by` and forget a label, you silently merge series that should stay separate. For example, `sum(rate(http_requests_total[5m])) by (service)` merges all HTTP methods and status codes -- fine for total throughput, wrong for error rate calculation.

</details>

<details>
<summary>7. How does PromQL vector matching work when you need to combine metrics from different sources — explain the 'on' and 'ignoring' keywords, what 'group_left' and 'group_right' do for many-to-one joins, and what are the common mistakes that cause 'many-to-many matching not allowed' errors?</summary>

**The problem vector matching solves:**

When you write a binary operation like `metric_a / metric_b`, PromQL needs to match series from the left side with series from the right side. By default, it matches on ALL labels -- both sides must have exactly the same label set (after dropping `__name__`). In practice, metrics from different sources rarely have identical labels, so you need to control how matching works.

**`on` -- match only on these labels:**

```promql
# Divide request errors by total requests, matching only on "service" and "method"
http_errors_total / on(service, method) http_requests_total
```

Both sides might have extra labels (like `instance`), but matching only considers `service` and `method`.

**`ignoring` -- match on all labels except these:**

```promql
# Same result if the only differing label is "status"
http_errors_total / ignoring(status) http_requests_total
```

`on` and `ignoring` are inverses, like `by` and `without` for aggregation.

**`group_left` and `group_right` -- many-to-one joins:**

By default, PromQL expects one-to-one matching: each series on the left matches exactly one series on the right. When one side has more series (e.g., multiple status codes on the left, one total on the right), you get a "many-to-many matching not allowed" error.

`group_left` says "the left side has multiple series per match group" (many-to-one where the "one" is on the right):

```promql
# Error rate by status code. Left has {service, status}, right has {service}
rate(http_requests_total{status=~"5.."}[5m])
/ on(service) group_left
sum(rate(http_requests_total[5m])) by (service)
```

Here, the left side has multiple series per service (one per status code), while the right side has one series per service. `group_left` tells PromQL this is intentional.

`group_right` is the inverse -- the right side has more series.

You can also copy labels from the "one" side into the result:

```promql
# Add "team" label from service_info metric to request rate
rate(http_requests_total[5m])
* on(service) group_left(team)
service_info
```

**Common mistakes causing "many-to-many matching not allowed":**

1. **Forgot to aggregate one side.** You're dividing errors by total, but both sides have `instance` and `status` labels, producing multiple matches. Fix: aggregate the denominator with `sum() by(...)`.
2. **Missing `group_left`/`group_right`.** The cardinalities genuinely differ, but you didn't declare the join type.
3. **Wrong `on()` labels.** You matched on labels that don't produce unique groups on either side. For example, matching `on(service)` when both sides have multiple series per service without `group_left`.
4. **Label collision.** Both sides have a label with the same name but different values, and you're not explicit about which to keep.

**Debugging approach:** When you get this error, check the cardinality of each side of the operation. Use `count by (label) (metric)` to see how many series exist per label value on each side, then decide which side is the "many" and add the appropriate `group_left` or `group_right`.

</details>

<details>
<summary>8. How do you design alerting rules that are actually useful in production — why does the "for" duration clause exist and how does it reduce false positives, how does predict_linear() work for proactive alerts like disk space, and what patterns distinguish good alerts (actionable, low noise) from bad ones (alert fatigue, flapping, too sensitive)?</summary>

**The `for` duration clause:**

```yaml
- alert: HighErrorRate
  expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
  for: 5m
```

The `for` clause means the expression must be continuously true for 5 minutes before the alert fires. Without it, a single bad scrape, a brief deployment spike, or a momentary network blip triggers the alert immediately.

How it works internally: When the expression first becomes true, the alert enters `pending` state. Prometheus re-evaluates on every evaluation interval (default 15s-1m). If the expression stays true for the entire `for` duration, the alert transitions to `firing` and is sent to Alertmanager. If it goes back to false at any point during the `for` window, the pending timer resets.

**`predict_linear()` for proactive alerts:**

```yaml
- alert: DiskWillFillIn4Hours
  expr: predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0
  for: 30m
```

`predict_linear(metric[window], seconds_ahead)` fits a simple linear regression over the data in the window and extrapolates. Here it looks at the last hour's disk space trend and predicts whether available bytes will hit zero within 4 hours. The `for: 30m` prevents alerting on temporary write bursts.

**Good alert patterns:**

- **Alert on symptoms, not causes.** Alert on "error rate > 5%" not "pod restarted." Users feel symptoms; causes are for investigation.
- **Every alert must be actionable.** If the on-call person can't do anything about it at 3 AM, it shouldn't page. Make it a warning to Slack instead.
- **Include runbook links.** Annotations should have `runbook_url` pointing to a doc that explains what to check and how to mitigate.
- **Use `for` duration generously.** 5-10 minutes for most alerts. Only use short/no `for` for truly critical "the system is completely down" alerts.
- **Alert on SLOs, not arbitrary thresholds.** "Error budget burn rate > 10x" is more meaningful than "errors > 100/s" because it's tied to business impact.

**Bad alert patterns:**

- **Flapping alerts.** The condition oscillates around the threshold. Fix: add hysteresis (alert at >5%, resolve at <3%), increase `for` duration, or smooth the metric with a longer `rate()` window.
- **Overly sensitive thresholds.** Alerting on CPU > 70% -- most services run fine at 70% CPU. Alert on saturation (request queuing, latency increase), not utilization.
- **Missing context.** An alert that says "HighLatency" with no information about which service, which endpoint, or what the current value is. Add labels and annotations.
- **Duplicate alerts.** Multiple alerts firing for the same underlying issue. Use Alertmanager's grouping and inhibition (covered in question 9) to solve this.
- **"Just in case" alerts.** Alerts added after an incident that are never tuned. They accumulate into noise. Review alerts quarterly -- if an alert hasn't fired in 3 months, question whether it's needed.

</details>

<details>
<summary>9. How does Alertmanager's routing and notification pipeline work — explain the routing tree (how alerts are matched to receivers), what grouping does and why it prevents notification storms, how throttling (group_wait, group_interval, repeat_interval) controls notification frequency, and when you'd use inhibition rules vs silences?</summary>

**The routing tree:**

Alertmanager receives firing alerts from Prometheus and routes them through a tree of matchers to determine which receiver (Slack, PagerDuty, email, etc.) handles each alert.

```yaml
route:
  receiver: default-slack          # catch-all
  group_by: [alertname, service]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: pagerduty-oncall
      group_wait: 10s              # pages go out faster
      repeat_interval: 1h
    - match:
        severity: warning
      receiver: team-slack
      routes:
        - match:
            team: payments
          receiver: payments-slack  # nested: warnings for payments team
```

Alerts are matched top-to-bottom. The first matching child route wins. If no child matches, the parent's receiver handles it. Routes can nest arbitrarily deep.

**Grouping (`group_by`):**

Without grouping, if 50 pods of the same service all fire `HighErrorRate` simultaneously, you get 50 separate notifications. Grouping batches alerts that share the same `group_by` labels into a single notification.

With `group_by: [alertname, service]`, all `HighErrorRate` alerts for `order-service` become one notification: "HighErrorRate firing for order-service (50 instances)."

**Throttling -- the three timing knobs:**

- **`group_wait: 30s`** -- After a new group appears, wait 30 seconds before sending the first notification. This batches alerts that fire in quick succession (e.g., all pods failing within seconds of each other) into one message instead of dribbling them out one by one.
- **`group_interval: 5m`** -- After sending a notification for a group, wait at least 5 minutes before sending another notification for the same group (even if new alerts join the group). Prevents rapid-fire updates.
- **`repeat_interval: 4h`** -- If the alert is still firing and nothing has changed, resend the notification every 4 hours as a reminder. Set this high enough to avoid noise, low enough that critical alerts don't get forgotten.

**Inhibition rules:**

Inhibition automatically suppresses certain alerts when other alerts are firing. It's declarative: "If alert X is firing, suppress alert Y."

```yaml
inhibit_rules:
  - source_match:
      alertname: ClusterDown
    target_match:
      severity: warning
    equal: [cluster]
```

This says: when `ClusterDown` is firing for cluster "prod-eu", suppress all `warning` severity alerts for the same cluster. This prevents a flood of downstream alerts when the root cause is a major infrastructure failure.

**Silences:**

Silences are manual, time-bounded mutes. You create them through the Alertmanager UI or API, specifying label matchers and a duration.

**When to use which:**

- **Inhibition** -- Automated, permanent rules for known dependency relationships. "If the database is down, don't alert on every service that can't connect to it."
- **Silences** -- Manual, temporary. Use during planned maintenance windows, known deployments, or while you're actively working on a known issue and don't want repeated pages.

</details>

## Grafana — Concepts

<details>
<summary>10. What makes a well-designed Grafana dashboard vs a wall of random graphs -- how do the USE method (utilization, saturation, errors) and RED method (rate, errors, duration) provide structure for organizing panels, what drill-down hierarchy should dashboards follow (overview to detail), and which panel types (time series, stat, heatmap, table) suit which kinds of data?</summary>

**USE and RED as organizational frameworks:**

These methods prevent the "wall of random graphs" by giving you a small, focused set of questions to answer per dashboard:

- **RED (for services/endpoints):** Rate (requests/sec), Errors (error rate or percentage), Duration (latency percentiles). This tells you "is this service healthy for its consumers?"
- **USE (for infrastructure/resources):** Utilization (how busy -- CPU %, memory used), Saturation (how overloaded -- queue depth, thread pool exhaustion), Errors (hardware/OS errors). This tells you "is this resource becoming a bottleneck?"

A service dashboard uses RED. An infrastructure dashboard uses USE. Don't mix them randomly.

**Drill-down hierarchy:**

Structure dashboards in layers, from broad to specific:

1. **Fleet overview** -- One dashboard showing all services at a glance. Stat panels with RED metrics per service, color-coded by health. This is the "war room" view during incidents.
2. **Service dashboard** -- One per service. Top row: stat panels showing current rate, error %, p99 latency (big numbers, color-coded with thresholds). Middle rows: time series panels for trends over time. Bottom rows: detailed breakdowns by endpoint, status code, pod.
3. **Deep-dive dashboard** -- Resource-level detail. Heatmaps for latency distribution, per-pod breakdowns, GC metrics, event loop lag. You navigate here from the service dashboard when investigating.

Link dashboards together using Grafana's dashboard links and data links so you can click from "service has high latency" to "which pods/endpoints are slow."

**Panel type selection:**

| Panel type | Best for | Example |
|---|---|---|
| **Stat** | Single current value, optionally with sparkline | Current request rate, error %, p99 |
| **Time series** | Trends over time, comparing multiple series | Request rate over 24h, latency percentiles over time |
| **Heatmap** | Distribution over time | Latency distribution (histogram buckets over time) |
| **Table** | Ranked lists, comparison across many items | Top 10 slowest endpoints, error counts by status code |
| **Gauge** | Current value against a known max | Disk usage %, CPU utilization |
| **Bar gauge** | Comparing values across categories | Memory usage per pod |

**Dashboard antipatterns to avoid:**

- **Too many panels.** If you have 40+ panels, nobody will look at most of them. Aim for 8-15 panels per dashboard. Use drill-down links for detail.
- **No template variables.** Hard-coded service names mean you need a separate dashboard per service. Use variables (`$service`, `$environment`, `$namespace`) so one dashboard works for all.
- **Missing units and thresholds.** A line at "0.5" means nothing without units. Configure panel units (requests/sec, bytes, seconds) and add threshold colors (green/yellow/red) so anyone can glance at it and know if it's healthy.
- **Default time range too wide.** A dashboard that defaults to "Last 7 days" loads slowly and obscures recent spikes. Default to 1-6 hours for operational dashboards.
- **No row organization.** Panels scattered randomly. Use collapsible rows with clear titles: "Overview", "Latency Detail", "Error Breakdown", "Resource Usage."

</details>

## Loki — Concepts

<details>
<summary>11. How does Loki's architecture fundamentally differ from Elasticsearch/ELK by using labels-only indexing instead of full-text indexing — what does "index labels, not log content" mean in practice, how does this architectural choice make Loki dramatically cheaper to operate at scale, and what query patterns does it make harder compared to full-text search?</summary>

**"Index labels, not log content" in practice:**

Elasticsearch tokenizes every log line, builds an inverted index over every word, and stores that index. If a log line contains `"user 12345 failed to authenticate from 10.0.0.1"`, Elasticsearch indexes "user", "12345", "failed", "authenticate", "10.0.0.1" -- all searchable instantly.

Loki does none of that. It only indexes the **labels** attached to the log stream -- typically `{service="auth", namespace="prod", pod="auth-7f8b9"}`. The actual log content is compressed and stored as chunks in object storage (S3, GCS) without any content index. When you query, Loki uses labels to find the right chunks, then brute-force scans the decompressed log lines for your search term.

**Why this makes Loki dramatically cheaper:**

| | Elasticsearch | Loki |
|---|---|---|
| **Index size** | Often larger than raw logs (inverted index + doc values) | Tiny -- only label combinations, not content |
| **Storage backend** | Requires fast local SSDs for index performance | Chunks in cheap object storage (S3 at ~$0.023/GB) |
| **Memory** | Needs significant RAM for index caching | Minimal -- no content index to cache |
| **CPU at ingest** | Tokenization, analysis, index building | Just compress and write chunks |
| **Operational complexity** | Cluster management, shard rebalancing, split brain | Stateless queriers + object storage |

At scale (TB/day of logs), Elasticsearch clusters require substantial infrastructure to maintain index performance. Loki's storage cost is dominated by object storage, which is an order of magnitude cheaper.

**What query patterns become harder:**

- **Ad-hoc full-text search across all logs.** Searching for a specific error message across all services requires Loki to decompress and scan every chunk. Elasticsearch returns this in seconds; Loki may take minutes without good label filters.
- **Searching without knowing which service.** If you don't know which label selectors to use, Loki scans broadly and slowly. Good label strategy is essential -- you need to narrow by `{service=..., namespace=...}` before doing line filters.
- **Complex text analytics.** Elasticsearch supports aggregations, fuzzy matching, and structured queries over log content. Loki's LogQL is simpler -- label filters + regex/pattern matching, no fuzzy search.
- **High-cardinality label queries.** Loki's index covers labels, but too many label values (like unique pod names in a high-churn environment) bloats the index and degrades performance -- the same cardinality problem as Prometheus.

**The tradeoff in one sentence:** Loki trades query flexibility for dramatically lower cost and operational simplicity, which is the right tradeoff when 90% of your log queries start with "show me logs for service X in the last hour."

</details>

<details>
<summary>12. How does LogQL work for querying logs in Loki — explain label-based filtering, regex line filters, structured field extraction using pattern and JSON parsers, and how metric queries (like counting error rates from logs) bridge the gap between logs and metrics?</summary>

LogQL queries have two stages: select log streams with label matchers, then filter/parse the content.

**Label-based filtering (stream selectors):**

```logql
{service="order-api", namespace="production"}
```

This is the most important performance decision. Labels determine which chunks Loki reads from storage. Always start with the narrowest label selector possible.

Operators: `=` (exact), `!=` (not equal), `=~` (regex match), `!~` (regex not match).

```logql
{service=~"order-.*", namespace!="staging"}
```

**Line filters:**

After selecting streams, filter log lines by content:

```logql
{service="order-api"} |= "error"          # contains "error" (fastest - simple substring)
{service="order-api"} != "healthcheck"     # does NOT contain
{service="order-api"} |~ "status=[45]\\d+" # regex match
{service="order-api"} !~ "DEBUG|TRACE"     # regex not match
```

Chain multiple filters -- each narrows further:

```logql
{service="order-api"} |= "error" != "healthcheck" |~ "user_id=\\d+"
```

**Structured field extraction:**

**JSON parser** -- extracts fields from JSON log lines into labels you can filter on:

```logql
{service="order-api"} | json | status >= 500 | line_format "{{.method}} {{.path}} {{.status}}"
```

If the log line is `{"method":"POST","path":"/orders","status":500,"duration":1.2}`, the `json` parser extracts `method`, `path`, `status`, `duration` as queryable labels.

**Pattern parser** -- extracts fields from unstructured logs using a template:

```logql
{service="nginx"} | pattern `<ip> - - [<timestamp>] "<method> <path> <_>" <status> <bytes>`
  | status >= 500
```

This parses nginx access logs and extracts `ip`, `method`, `path`, `status`, `bytes` as labels. `<_>` discards a field.

**Metric queries -- bridging logs and metrics:**

LogQL can compute metrics from log streams, turning logs into time-series data:

```logql
# Rate of error log lines per second over the last 5 minutes
rate({service="order-api"} |= "error" [5m])

# Count of 5xx status codes per minute, grouped by path
sum by (path) (
  count_over_time(
    {service="order-api"} | json | status >= 500 [1m]
  )
)

# Average request duration extracted from JSON logs
avg_over_time(
  {service="order-api"} | json | unwrap duration [5m]
) by (method)
```

Key metric functions:
- `rate()` -- log lines per second (like Prometheus `rate()`)
- `count_over_time()` -- total matching lines in window
- `sum_over_time()`, `avg_over_time()`, `max_over_time()` -- aggregate extracted numeric values (requires `unwrap` to convert a label to a numeric value)
- `bytes_rate()` -- bytes per second of log data

**Why this matters:** Metric queries let you create Grafana alerts and dashboards from log data without pre-instrumenting metrics in code. You can alert on "error rate in logs > 10/s" when there's no Prometheus counter for that specific error.

</details>

## Practical — Configuration & Instrumentation

<details>
<summary>13. Set up a Prometheus scrape configuration for a Kubernetes environment — show the YAML for scraping application pods using service discovery (kubernetes_sd_configs), explain how relabel_configs work to filter targets and rewrite labels, configure an appropriate scrape interval and storage retention, and explain what breaks if you set the scrape interval too low or retention too high for your storage.</summary>

**Complete scrape configuration:**

```yaml
global:
  scrape_interval: 15s     # how often to scrape targets
  evaluation_interval: 15s  # how often to evaluate alerting rules
  scrape_timeout: 10s       # must be less than scrape_interval

scrape_configs:
  # Scrape application pods that opt in via annotations
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod  # discover all pods in the cluster

    relabel_configs:
      # Only scrape pods with prometheus.io/scrape: "true" annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"

      # Use custom metrics path if annotated, otherwise /metrics
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # Use custom port if annotated
      - source_labels:
          [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      # Add useful labels from pod metadata
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: service
```

**How relabel_configs work:**

Relabeling runs before scraping and transforms the target's label set. Each rule has:
- `source_labels` -- input labels to read (concatenated with `;`)
- `regex` -- pattern to match against the concatenated source
- `action` -- what to do: `keep` (only scrape if regex matches), `drop` (skip if matches), `replace` (write to target_label), `labelmap` (copy matching labels), `labeldrop`/`labelkeep` (remove/keep labels by name)
- `target_label` -- where to write the result

The `__meta_kubernetes_*` labels are injected by service discovery and contain pod metadata (name, namespace, annotations, labels). They're available during relabeling but dropped before storage -- you must explicitly copy what you need into permanent labels.

**Storage retention configuration (CLI flags):**

```bash
prometheus \
  --storage.tsdb.retention.time=15d \     # keep data for 15 days
  --storage.tsdb.retention.size=50GB \    # OR cap storage at 50GB (whichever hits first)
  --storage.tsdb.path=/prometheus/data
```

**What breaks with bad settings:**

**Scrape interval too low (e.g., 1-2 seconds):**
- Prometheus generates 15x more samples per series compared to 15s interval, consuming proportionally more memory, CPU, and disk.
- Target applications spend more time serving `/metrics` requests, which can impact their own performance.
- Short-lived metrics (like histogram buckets) multiply the ingestion rate.
- `rate()` calculations don't actually improve much -- you're paying storage cost for negligible monitoring benefit.

**Retention too high for available disk:**
- TSDB fills the disk and Prometheus stops accepting new samples. Active monitoring goes blind.
- Compaction fails when disk is nearly full, which paradoxically makes disk usage worse (uncompacted blocks are larger).
- Queries over long time ranges become extremely slow without recording rules or downsampling.

**Practical recommendations:** 15s scrape interval is the sweet spot for most workloads. For retention beyond 15-30 days, use remote write to Thanos or Mimir instead of extending local retention.

</details>

<details>
<summary>14. Instrument a Node.js application using prom-client — show the TypeScript code to expose a /metrics endpoint, enable default metrics (event loop lag, heap size, GC), create custom histograms for HTTP request duration and custom counters for business events, and explain the label cardinality risks that catch most teams off guard (like using request path or user ID as labels).</summary>

```typescript
import express from "express";
import {
  collectDefaultMetrics,
  register,
  Counter,
  Histogram,
} from "prom-client";

// Default metrics: event loop lag, heap size, GC duration, active handles, etc.
// prefix scopes them to avoid collisions across services
collectDefaultMetrics({ prefix: "orderservice_" });

// Custom histogram for HTTP request duration
const httpRequestDuration = new Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration in seconds",
  labelNames: ["method", "route", "status_code"] as const,
  // Buckets tuned for a typical API (most responses 10-500ms)
  buckets: [0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2, 5],
});

// Custom counter for business events
const ordersCreated = new Counter({
  name: "orders_created_total",
  help: "Total orders successfully created",
  labelNames: ["payment_method"] as const,
});

const app = express();

// Metrics middleware -- track every request
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();

  res.on("finish", () => {
    end({
      method: req.method,
      route: req.route?.path ?? "unknown", // normalized route, NOT req.url
      status_code: res.statusCode.toString(),
    });
  });

  next();
});

// Business logic uses the counter
app.post("/orders", (req, res) => {
  // ... create order logic ...
  ordersCreated.inc({ payment_method: "credit_card" });
  res.status(201).json({ id: "order-123" });
});

// Metrics endpoint -- Prometheus scrapes this
app.get("/metrics", async (_req, res) => {
  res.set("Content-Type", register.contentType);
  res.end(await register.metrics());
});

app.listen(3000);
```

**Key implementation details:**

- `startTimer()` returns a function that, when called, observes the elapsed time. This is the cleanest pattern for measuring durations.
- `register.metrics()` is async -- it serializes all registered metrics into Prometheus exposition format.
- `collectDefaultMetrics()` automatically registers ~20 Node.js runtime metrics: `process_cpu_seconds_total`, `nodejs_eventloop_lag_seconds`, `nodejs_heap_size_total_bytes`, `nodejs_gc_duration_seconds`, `nodejs_active_handles_total`, etc.

**Label cardinality risks that catch teams off guard:**

1. **Using `req.url` instead of `req.route.path`.** `req.url` includes path parameters: `/api/users/abc123`, `/api/users/def456` -- unbounded cardinality. `req.route.path` gives you `/api/users/:id` -- bounded. This is the single most common mistake.

2. **User ID or tenant ID as a label.** Tempting for per-user monitoring, but 100K users x 4 metrics x 15 buckets = millions of series. Use logs for per-user debugging, not metrics.

3. **Uncontrolled `route` for 404s.** Bots and scanners hit random URLs like `/wp-admin`, `/.env`, `/phpmyadmin`. Each creates a new series. Fix: map unmatched routes to `"not_found"`.

4. **Error messages as labels.** Dynamic error strings create unbounded label values. Use error categories (`"validation"`, `"timeout"`, `"internal"`) not the raw message.

5. **Not handling the "unknown" case.** When `req.route?.path` is undefined (middleware hits before route matching), falling back to `req.path` reintroduces the problem. Always have a bounded fallback like `"unknown"`.

</details>

<details>
<summary>15. Write the PromQL queries that implement the RED method for a typical HTTP service — show queries for request rate, error percentage (as a ratio of total requests), and p95/p99 latency using histogram_quantile(). Explain why histogram_quantile() requires rate() inside it, what the "le" label means, and what happens to accuracy when bucket boundaries don't align with actual latency distribution.</summary>

**Rate -- requests per second:**

```promql
sum(rate(http_requests_total{service="order-api"}[5m]))
```

Break down by endpoint:

```promql
sum(rate(http_requests_total{service="order-api"}[5m])) by (method, route)
```

**Errors -- error percentage as a ratio of total requests:**

```promql
sum(rate(http_requests_total{service="order-api", status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="order-api"}[5m]))
```

Multiply by 100 for a percentage in Grafana, or leave as a ratio (0.05 = 5%) for alerting thresholds.

**Duration -- p95 and p99 latency:**

```promql
# p99 latency across all instances
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{service="order-api"}[5m])) by (le)
)

# p95 latency broken down by route
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{service="order-api"}[5m])) by (le, route)
)
```

**Why `histogram_quantile()` requires `rate()` inside:**

Histogram buckets are counters -- they monotonically increase over the lifetime of the process. Without `rate()`, you'd be computing percentiles over the entire history of the process, not over a recent time window. `rate()` converts the cumulative bucket counts into per-second rates for the specified window, giving you "what's the p99 latency for requests in the last 5 minutes" rather than "what's the p99 across all requests since the process started."

Additionally, `rate()` handles counter resets correctly. Without it, a process restart would cause `histogram_quantile()` to produce nonsensical results.

**What the `le` label means:**

`le` stands for "less than or equal to" -- it's the upper bound of each histogram bucket. A series `http_request_duration_seconds_bucket{le="0.5"}` counts all requests that completed in 0.5 seconds or less. Buckets are cumulative, so `le="1"` includes everything in `le="0.5"`.

The `le` label must be preserved in the `by` clause of `sum()` because `histogram_quantile()` needs it to reconstruct the distribution. Forgetting `by (le)` is a common mistake that produces an error.

**Accuracy when buckets don't align with actual distribution:**

`histogram_quantile()` uses linear interpolation within buckets. If your p99 falls between bucket boundaries `le="0.5"` and `le="1"`, Prometheus assumes values are evenly distributed in that range and interpolates linearly. This can be wildly inaccurate:

- If most requests in that range are at 0.55s, the interpolated p99 might report 0.95s -- a 72% overestimate.
- The wider the gap between buckets around your target percentile, the worse the accuracy.

This is why bucket design matters (covered in question 4). Concentrate buckets around your SLO target, not spread evenly across the full range.

</details>

<details>
<summary>16. Write alerting rules for three production scenarios: high error rate (>5% of requests returning 5xx), high p99 latency (>2s for 5 minutes), and disk filling up within 4 hours using predict_linear() — show the YAML for each rule, explain how the "for" duration prevents flapping, and what labels and annotations you'd add to make the alert actionable when it fires.</summary>

```yaml
groups:
  - name: service-health
    rules:
      # 1. High error rate -- >5% of requests returning 5xx
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
          ) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.service }} error rate is {{ $value | humanizePercentage }}"
          description: "More than 5% of requests are returning 5xx errors."
          runbook_url: "https://runbooks.internal/high-error-rate"
          dashboard: "https://grafana.internal/d/service-red?var-service={{ $labels.service }}"

      # 2. High p99 latency -- >2s sustained for 5 minutes
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }} p99 latency is {{ $value | humanizeDuration }}"
          description: "p99 latency has exceeded 2 seconds for 5 minutes."
          runbook_url: "https://runbooks.internal/high-latency"

  - name: infrastructure
    rules:
      # 3. Disk filling up within 4 hours
      - alert: DiskWillFillIn4Hours
        expr: |
          predict_linear(
            node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[1h],
            4 * 3600
          ) < 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Disk {{ $labels.mountpoint }} on {{ $labels.instance }} will fill in ~4h"
          description: "Based on the last hour's trend, available space will reach zero."
          current_available: "{{ $value | humanize1024 }}B remaining"
          runbook_url: "https://runbooks.internal/disk-full"
```

**How `for` prevents flapping in each case:**

- **HighErrorRate `for: 5m`:** A brief spike during a deployment (lasting 1-2 minutes) won't trigger the alert. The error rate must stay above 5% for a full 5 minutes. Combined with the `rate()[5m]` window, this means roughly 10 minutes of problematic behavior before a page fires.
- **HighP99Latency `for: 5m`:** A single slow request or GC pause that briefly pushes p99 above 2s won't page. The latency must be consistently high.
- **DiskWillFillIn4Hours `for: 30m`:** A temporary log burst that makes the trend look alarming for 10 minutes won't trigger. The prediction must hold for 30 minutes, meaning the disk usage trend is genuinely sustained.

**What makes these alerts actionable:**

- **`summary`** uses template variables (`$labels.service`, `$value`) so the notification tells you WHICH service and HOW BAD it is, not just "something is wrong."
- **`runbook_url`** links directly to the investigation steps. The on-call person doesn't need to remember what to do at 3 AM.
- **`dashboard`** links to the relevant Grafana dashboard with the service pre-selected.
- **`severity` labels** drive Alertmanager routing -- `critical` goes to PagerDuty, `warning` goes to Slack (as configured in question 17).
- **`description`** gives context on the threshold so the responder knows what "normal" should look like.

</details>

## Practical — Alerting, Dashboards & Logs

<details>
<summary>17. Configure Alertmanager's routing tree for a multi-team setup — show the YAML where critical alerts go to PagerDuty, warning alerts go to Slack grouped by service, and a catch-all default route. Configure group_by, group_wait, group_interval, and repeat_interval with appropriate values, add an inhibition rule where a cluster-down alert suppresses individual service alerts, and explain the routing match logic.</summary>

```yaml
global:
  resolve_timeout: 5m  # mark alert resolved if not received for 5 min
  slack_api_url: "https://hooks.slack.com/services/T00/B00/xxx"
  pagerduty_url: "https://events.pagerduty.com/v2/enqueue"

receivers:
  - name: "default-slack"
    slack_configs:
      - channel: "#alerts-general"
        send_resolved: true

  - name: "pagerduty-oncall"
    pagerduty_configs:
      - routing_key: "<pagerduty-integration-key>"
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'

  - name: "team-platform-slack"
    slack_configs:
      - channel: "#platform-alerts"
        send_resolved: true

  - name: "team-payments-slack"
    slack_configs:
      - channel: "#payments-alerts"
        send_resolved: true

route:
  receiver: default-slack         # catch-all: anything unmatched goes here
  group_by: [alertname, service]  # group alerts by name + service
  group_wait: 30s                 # wait 30s to batch initial alerts
  group_interval: 5m              # wait 5m before sending updates to a group
  repeat_interval: 4h             # re-notify every 4h if still firing

  routes:
    # Critical alerts → PagerDuty (faster timing)
    - match:
        severity: critical
      receiver: pagerduty-oncall
      group_wait: 10s             # pages go out fast
      repeat_interval: 1h         # re-page every hour if unacknowledged
      continue: false             # stop matching after this

    # Warning alerts → Slack, with team-specific subroutes
    - match:
        severity: warning
      receiver: default-slack     # default for warnings
      routes:
        - match:
            team: platform
          receiver: team-platform-slack
        - match:
            team: payments
          receiver: team-payments-slack

inhibit_rules:
  # Cluster-down suppresses all individual service alerts in the same cluster
  - source_matchers:
      - alertname = ClusterDown
    target_matchers:
      - severity =~ "warning|critical"
      - alertname != ClusterDown    # don't suppress itself
    equal: [cluster]                # must match on cluster label

  # Critical suppresses warning for the same alert on the same service
  - source_matchers:
      - severity = critical
    target_matchers:
      - severity = warning
    equal: [alertname, service]
```

**Routing match logic walkthrough:**

1. Alert `{alertname="HighErrorRate", severity="critical", service="order-api", team="payments"}` arrives.
2. Alertmanager starts at the root route and checks child routes top-to-bottom.
3. First child: `match: severity: critical` -- matches. Routes to `pagerduty-oncall`. `continue: false` (default) means stop -- no further routes are checked.

4. Alert `{alertname="HighLatency", severity="warning", service="checkout", team="payments"}` arrives.
5. First child: `severity: critical` -- no match. Move to next child.
6. Second child: `severity: warning` -- matches. Check its subroutes.
7. Subroute `team: platform` -- no match. Next subroute `team: payments` -- matches. Routes to `team-payments-slack`.

8. Alert `{alertname="Watchdog", severity="info"}` arrives.
9. No child routes match. Falls through to root receiver: `default-slack`.

**Inhibition logic:**

When `ClusterDown{cluster="prod-eu"}` fires, Alertmanager automatically drops notifications for any alert with `severity=~"warning|critical"` that has `cluster="prod-eu"`. This prevents the on-call from getting 50 individual service alerts when the real problem is the cluster itself. The `equal: [cluster]` ensures suppression only happens within the same cluster -- a cluster-down in prod-eu doesn't silence alerts in prod-us.

</details>

<details>
<summary>18. Build a Grafana dashboard for a production service using the RED method — describe the panel layout and drill-down hierarchy (overview row with stat panels for current rate/error%/p99, then time series panels for trends, then heatmap for latency distribution), show how to configure template variables so the same dashboard works across all services and environments, and explain what dashboard antipatterns to avoid (too many panels, no drill-down, missing units/thresholds).</summary>

**Template variables (dashboard settings):**

```
Variable: service
Type: Query
Query: label_values(http_requests_total, service)
Multi-value: enabled

Variable: namespace
Type: Query
Query: label_values(http_requests_total{service="$service"}, namespace)

Variable: route
Type: Query
Query: label_values(http_requests_total{service="$service"}, route)
Include All: enabled
```

These create dropdowns at the top of the dashboard. All panel queries use `{service="$service", namespace="$namespace"}` so one dashboard works for every service and environment.

**Panel layout -- three rows:**

**Row 1: Overview (stat panels -- the "at a glance" row)**

| Panel | Query | Unit | Thresholds |
|---|---|---|---|
| Request Rate | `sum(rate(http_requests_total{service="$service"}[5m]))` | req/s | -- |
| Error Rate | `sum(rate(http_requests_total{service="$service",status_code=~"5.."}[5m])) / sum(rate(http_requests_total{service="$service"}[5m]))` | percent (0-1) | green <1%, yellow <5%, red >=5% |
| p99 Latency | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service="$service"}[5m])) by (le))` | seconds | green <500ms, yellow <1s, red >=1s |

Stat panels show a single big number with color-coding. Anyone can glance at these and know if the service is healthy in 2 seconds.

**Row 2: Trends (time series panels)**

- **Request Rate over time** -- `sum(rate(http_requests_total{service="$service"}[5m])) by (route)` -- stacked area chart, one line per route. Shows traffic patterns and which endpoints are busiest.
- **Error Rate over time** -- same error ratio query, `by (route)` -- time series. Correlate error spikes with deployments or traffic changes.
- **Latency percentiles over time** -- three lines: p50, p95, p99 using `histogram_quantile()`. Shows whether latency degradation is gradual (capacity issue) or sudden (code regression).

**Row 3: Deep Dive (collapsible)**

- **Latency heatmap** -- `sum(rate(http_request_duration_seconds_bucket{service="$service"}[5m])) by (le)` with heatmap visualization. Shows the full latency distribution over time -- much more informative than percentile lines alone. A bimodal distribution (some fast, some slow) is invisible in p99 but obvious in a heatmap.
- **Status code breakdown** -- `sum(rate(http_requests_total{service="$service"}[5m])) by (status_code)` -- stacked bar or time series. Shows 200s vs 400s vs 500s over time.
- **Top endpoints by error count** -- table panel, sorted by error rate descending. Quick identification of which endpoint is broken.

**Dashboard antipatterns (expanding on question 10):**

- **No template variables** -- Hard-coding `service="order-api"` means maintaining N identical dashboards. Always parameterize.
- **Too many panels per row** -- More than 3-4 panels in a row makes each unreadably small. Use rows with 2-3 panels and make the deep-dive row collapsible.
- **Missing units** -- A value of `0.247` means nothing. Set panel units: seconds for latency, `reqps` for rate, `percentunit` (0-1) for ratios. Grafana formats them as "247ms" and "5.2%" automatically.
- **No link to logs** -- Add a data link on the error rate panel that opens Loki pre-filtered: `https://grafana.internal/explore?expr={service="${__field.labels.service}"}|="error"`. This makes dashboard-to-logs investigation seamless.
- **Default time range too wide** -- Set dashboard default to "Last 1 hour" with auto-refresh at 30s for operational use. A 7-day default loads slowly and hides recent issues.

</details>

## Practical — Debugging & Troubleshooting

<details>
<summary>19. A service's metrics suddenly disappear from Prometheus — walk through the exact steps to diagnose why: checking the Prometheus targets page, verifying the scrape config and service discovery, testing the /metrics endpoint directly, inspecting relabel_configs for dropped targets, and checking for scrape errors. What are the most common causes of missing metrics and how do you fix each?</summary>

**Step 1: Check the Prometheus Targets page**

Go to `http://prometheus:9090/targets`. Find your service's job. Three possible states:

- **Target is missing entirely** -- Prometheus doesn't know about it. Problem is in service discovery or relabel_configs.
- **Target shows `DOWN` (red)** -- Prometheus found it but can't scrape. The "Error" column tells you why (connection refused, timeout, 404, etc.).
- **Target shows `UP` (green)** -- Prometheus is scraping successfully, but the specific metric you're looking for isn't in the response. Problem is in the application or metric_relabel_configs.

**Step 2: If target is missing -- check service discovery**

```
http://prometheus:9090/service-discovery
```

This shows all discovered targets BEFORE relabeling. If your pod appears here with `__meta_kubernetes_*` labels but doesn't appear in `/targets`, your `relabel_configs` are dropping it.

Common causes:
- **Missing annotation.** Pod doesn't have `prometheus.io/scrape: "true"`. Fix: add the annotation to the deployment spec.
- **Wrong port annotation.** `prometheus.io/port` points to a port that doesn't exist or isn't the metrics port.
- **Relabel regex mismatch.** A `keep` rule's regex doesn't match the label value. Debug by temporarily removing relabel rules one at a time.

**Step 3: If target is DOWN -- test the endpoint directly**

```bash
# From inside the cluster
kubectl exec -it debug-pod -- curl -s http://order-api.prod.svc:3000/metrics

# Port-forward and test locally
kubectl port-forward svc/order-api 3000:3000
curl -s http://localhost:3000/metrics
```

Common causes:
- **App crashed or isn't ready.** Check pod status: `kubectl get pods -l app=order-api`.
- **Wrong port/path.** The scrape config targets port 80 but metrics are on port 3000, or the path is `/api/metrics` not `/metrics`.
- **Network policy blocking.** Prometheus pod can't reach the target. Check NetworkPolicy rules.
- **Scrape timeout.** The `/metrics` endpoint takes too long (e.g., a metric collection callback is slow). Check `scrape_duration_seconds` for that target.

**Step 4: If target is UP but specific metrics are missing**

The target is being scraped but individual metrics disappeared:

- **Application stopped registering the metric.** A deployment changed the code. Check the raw `/metrics` output for the metric name.
- **`metric_relabel_configs` dropping it.** A relabel rule with `action: drop` matches the metric. Check the Prometheus config.
- **Metric renamed.** A library upgrade changed the metric name (e.g., `node_cpu` became `node_cpu_seconds_total`).

**Step 5: Check for scrape errors**

```promql
# Scrape failures for the target
scrape_duration_seconds{job="order-api"}         # how long scrapes take
scrape_samples_scraped{job="order-api"}           # how many samples per scrape
scrape_samples_post_metric_relabeling{job="order-api"}  # samples after relabeling (if lower, relabeling is dropping)
up{job="order-api"}                               # 1 = success, 0 = failed
```

If `scrape_samples_scraped` is high but `scrape_samples_post_metric_relabeling` is low, your `metric_relabel_configs` are dropping metrics.

</details>

<details>
<summary>20. An alert that should be firing isn't, or an alert is firing constantly when it shouldn't — walk through the debugging process: testing the PromQL expression in the Prometheus UI, checking the "for" duration and whether the condition was sustained, inspecting Alertmanager's routing to verify the alert reaches the right receiver, checking for silences or inhibition rules suppressing it, and verifying notification channel configuration. Show the specific pages and API endpoints you'd check.</summary>

**Scenario A: Alert should be firing but isn't**

**Step 1: Test the PromQL expression directly**

Go to `http://prometheus:9090/graph` and paste the alert's `expr`. Does it return results?

- **No results** -- The expression itself doesn't match any data. Common causes:
  - Metric name changed (typo, library upgrade).
  - Label values don't match (`status_code` vs `status`, `"500"` vs `500`).
  - The `rate()` window is too narrow for the scrape interval (if scrape_interval is 30s, `rate()[1m]` needs at least 2 samples -- use `rate()[5m]`).
- **Returns results but below threshold** -- The condition isn't actually true. Adjust threshold or investigate why the value is different from what you expected.

**Step 2: Check the alert's state in Prometheus**

```
http://prometheus:9090/alerts
```

Find your alert. Three states:
- **Inactive** -- Expression evaluates to false. The condition isn't met.
- **Pending** -- Expression is true but `for` duration hasn't elapsed. If it keeps resetting from pending to inactive, the condition is intermittent (flapping). Check if the value oscillates around the threshold.
- **Firing** -- Prometheus considers it active. If you're not getting notifications, the problem is downstream in Alertmanager.

**Step 3: Check the alerting rules are loaded**

```
http://prometheus:9090/api/v1/rules
```

Verify your rule group and alert rule appear. If not, the rules file wasn't loaded (check Prometheus config `rule_files` and look for config reload errors in Prometheus logs).

**Step 4: Check Alertmanager received the alert**

```
http://alertmanager:9093/#/alerts
```

Or via API:

```bash
curl http://alertmanager:9093/api/v2/alerts
```

If the alert appears here but you didn't get a notification:
- **Check silences:** `http://alertmanager:9093/#/silences` -- someone may have silenced it during maintenance and forgot to remove it.
- **Check inhibition rules:** Another alert (like `ClusterDown`) might be suppressing it via inhibition. Check which inhibit rules exist in the Alertmanager config.
- **Check routing:** The alert might be routed to the wrong receiver. Use the Alertmanager routing tree visualizer or test with the API:

```bash
# Test which receiver an alert would route to
curl -X POST http://alertmanager:9093/api/v2/alerts/groups
```

**Step 5: Verify notification channel**

If Alertmanager has the alert and routing is correct, the notification channel itself may be broken:
- Slack webhook URL expired or channel was renamed.
- PagerDuty integration key is wrong.
- Check Alertmanager logs for delivery errors.

---

**Scenario B: Alert firing constantly when it shouldn't**

**Step 1: Run the expression and check the actual value**

The threshold may be wrong. If error rate is `0.03` (3%) and the threshold is `> 0.02`, it's firing correctly -- the threshold is too sensitive.

**Step 2: Check for noisy metrics**

A small denominator inflates ratios. If a service gets 2 requests/min and 1 fails, the error rate is 50%. Fix with a minimum traffic guard:

```promql
(
  sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
  /
  sum(rate(http_requests_total[5m])) by (service)
) > 0.05
and
sum(rate(http_requests_total[5m])) by (service) > 1  # at least 1 req/s
```

**Step 3: Check the `for` duration**

If the alert has no `for` or `for: 0s`, every momentary spike fires it. Add `for: 5m` to require sustained violation.

**Step 4: Check the rate window**

`rate()[1m]` is noisy. A 1-minute window on a 15s scrape interval means 4 samples -- a single slow request can spike the average. Use `rate()[5m]` for alerting to smooth out transients.

</details>

<details>
<summary>21. Grafana dashboards are loading slowly or timing out — diagnose whether the problem is in Grafana, the data source (Prometheus/Loki), or the queries themselves. Walk through checking query execution time in Prometheus, identifying queries that scan too much data (missing label filters, long time ranges without recording rules), using the Grafana query inspector, and explain when the fix is query optimization vs recording rules vs adjusting dashboard time range defaults.</summary>

**Step 1: Use Grafana's Query Inspector to isolate the problem**

Open the slow dashboard, click on a panel's title, and select "Inspect > Query". This shows:
- **Request time** -- how long Grafana waited for the data source to respond.
- **Data source response time** -- how long Prometheus/Loki took to execute the query.
- **Rows returned** -- how much data came back.

If the data source response time is high (seconds), the problem is in Prometheus/Loki or the query itself. If the data source responds fast but Grafana is slow, the problem is rendering (too many series, too many data points).

**Step 2: Test the query directly in Prometheus**

Copy the query from the inspector and run it in `http://prometheus:9090/graph`. Check the execution time shown in the UI. Also check:

```promql
# How many series does this query touch?
count(http_requests_total{service="order-api"})
```

If this returns thousands of series, the query is scanning too much data.

**Step 3: Identify common query problems**

- **Missing label filters.** A query like `rate(http_requests_total[5m])` without any label selectors scans EVERY series for that metric across all services. Always filter: `{service="$service"}`.
- **Long time ranges on raw data.** A 30-day view with `rate()[5m]` at 15s resolution = 172,800 data points per series. Prometheus must read and process all of them.
- **High-cardinality `by` clause.** `sum(rate(...)) by (path, method, status, instance)` can produce thousands of series. Grafana struggles to render that many lines.
- **Nested `histogram_quantile` over wide ranges.** Computing p99 across a 7-day range with fine-grained buckets is expensive. Prometheus must read every bucket series.
- **Regex label matchers.** `{service=~".*"}` or `{path=~".+"}` force full scans. Use exact matches where possible.

**Step 4: Choose the right fix**

**Query optimization** (quick wins):
- Add label selectors to narrow the scan.
- Reduce `by` dimensions -- do you really need per-instance breakdown?
- Use `topk(10, ...)` instead of showing all series.
- Shorten the default time range (1h instead of 24h for operational dashboards).

**Recording rules** (for queries that are inherently expensive):

When a query touches high-cardinality data and is used repeatedly (dashboards, alerts), pre-compute it as a recording rule:

```yaml
groups:
  - name: service-red-recordings
    interval: 30s
    rules:
      - record: service:http_request_rate:5m
        expr: sum(rate(http_requests_total[5m])) by (service)

      - record: service:http_error_ratio:5m
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)

      - record: service:http_request_duration_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          )
```

Dashboard queries then read the pre-computed `service:http_request_rate:5m` instead of computing `sum(rate(...))` on every dashboard load. The recording rule runs once every 30 seconds; the dashboard query reads a single low-cardinality series, making it near-instant regardless of time range.

**When recording rules are worth it:**
- The same expensive query appears in multiple dashboards or alerts.
- The dashboard is used frequently (on a TV, refreshing every 30s).
- Long time ranges are needed (7d, 30d views) where raw data scanning is prohibitive.

**Adjusting dashboard defaults** (least effort):
- Set default time range to 1-6 hours instead of 24h or 7d.
- Set max data points per panel (Grafana will downsample).
- Use auto-refresh interval of 30s-1m, not 5s.

</details>

<details>
<summary>22. Prometheus is consuming excessive memory and disk, and you suspect a cardinality explosion — walk through how to detect which metrics and labels are causing the problem (using TSDB status page, promtool, or PromQL queries like count of time series per metric), how to identify the offending labels, and how to mitigate the issue both immediately (dropping labels via relabel_configs, deleting series) and long-term (metric naming conventions, cardinality limits, CI checks).</summary>

**Step 1: Confirm cardinality is the problem**

```promql
# Total active time series -- is this abnormally high?
prometheus_tsdb_head_series

# Rate of new series creation -- a sudden spike means something just deployed a cardinality bomb
rate(prometheus_tsdb_head_series_created_total[5m])

# Memory used by head block (in-memory series)
prometheus_tsdb_head_chunks_created_total
process_resident_memory_bytes
```

If `prometheus_tsdb_head_series` is in the millions (or jumped dramatically recently), cardinality is almost certainly the cause.

**Step 2: Find the offending metrics and labels**

**TSDB status page** (most direct):

```
http://prometheus:9090/api/v1/status/tsdb
```

This returns:
- `seriesCountByMetricName` -- top 10 metrics by series count. This immediately tells you which metric is exploding.
- `labelValueCountByLabelName` -- which label names have the most unique values (e.g., `path` has 50,000 unique values).
- `memoryInBytesByLabelName` -- memory consumed per label.
- `seriesCountByLabelValuePair` -- top label key=value pairs (e.g., `job="order-api"` has 2M series).

**PromQL queries for deeper investigation:**

```promql
# Series count per metric name -- find the expensive ones
sort_desc(count by (__name__) ({__name__=~".+"}))

# For a specific metric, find which label has the most values
count(http_request_duration_seconds_bucket) by (path)

# How many series does a specific job contribute?
count({job="order-api"})
```

**promtool (offline analysis):**

```bash
# Analyze TSDB for cardinality issues
promtool tsdb analyze /prometheus/data

# Output shows top series by label, helping identify which label dimensions are unbounded
```

**Step 3: Immediate mitigation**

**Drop the offending label at scrape time** using `metric_relabel_configs`:

```yaml
scrape_configs:
  - job_name: "order-api"
    metric_relabel_configs:
      # Replace the high-cardinality "path" label with a constant, scoped to histogram metrics only
      - source_labels: [__name__, path]
        regex: "http_request_duration_seconds_.+;.+"
        target_label: path
        replacement: "aggregated"
        action: replace

      # Or drop entire metrics that are too expensive
      - source_labels: [__name__]
        regex: "expensive_metric_.*"
        action: drop
```

Reload Prometheus config (`kill -HUP` or `/-/reload` endpoint). New scrapes will stop creating series with the dropped label. Existing series remain in TSDB until they fall outside the retention window.

**Delete existing series** (to free resources immediately):

```bash
# Delete series matching a selector (requires --web.enable-admin-api)
curl -X POST 'http://prometheus:9090/api/v1/admin/tsdb/delete_series' \
  -d 'match[]={__name__="http_request_duration_seconds_bucket", path=~".+"}'

# Force a head block compaction to actually free disk
curl -X POST 'http://prometheus:9090/api/v1/admin/tsdb/clean_tombstones'
```

**Step 4: Long-term prevention**

1. **Fix the source.** The application code needs to normalize paths, remove unbounded labels, etc. (as covered in question 5). Relabel_configs are a band-aid, not a fix.

2. **Metric naming conventions.** Establish team standards: counters end in `_total`, histograms end in `_seconds` or `_bytes`, labels are always bounded with known value sets documented.

3. **CI/CD checks.** Add a pipeline step that:
   - Scans code for new metric registrations and flags unbounded labels.
   - Runs the app locally, scrapes `/metrics`, and counts unique series. Reject if above a threshold.

4. **Cardinality alerts** (as shown in question 5):
   ```yaml
   - alert: HighCardinalityMetric
     expr: count by (__name__) ({__name__=~".+"}) > 10000
     for: 15m
     annotations:
       summary: "Metric {{ $labels.__name__ }} has {{ $value }} series"
   ```

5. **Use a metrics proxy.** Tools like Grafana Agent or Prometheus relabel rules can enforce per-metric cardinality limits, automatically dropping series when a metric exceeds a configured maximum.

</details>

<details>
<summary>23. Loki queries are returning no results or timing out — walk through the debugging process: verifying logs are being ingested (checking Promtail/Alloy pipeline, Loki /ready and /metrics endpoints), testing label selectors vs line filters, diagnosing query timeouts caused by scanning too many streams (wrong label strategy forcing full scans), and explain how to restructure queries or labels to fix performance.</summary>

**Scenario A: No results**

**Step 1: Verify Loki is healthy**

```bash
# Is Loki up and ready?
curl http://loki:3100/ready      # should return "ready"
curl http://loki:3100/metrics    # check for errors in internal metrics
```

Check Loki's internal metrics for ingestion:
```promql
# Are logs being ingested at all?
rate(loki_distributor_lines_received_total[5m])

# Are there ingestion errors?
rate(loki_distributor_ingester_append_failures_total[5m])
```

If no logs are being ingested, the problem is upstream in the collection pipeline.

**Step 2: Check the collection agent (Promtail/Grafana Alloy)**

- Is the agent running? `kubectl get pods -l app=promtail`
- Check agent logs for errors: `kubectl logs -l app=promtail --tail=50`
- Common issues:
  - Agent can't reach Loki (network policy, wrong URL, TLS mismatch).
  - File discovery not matching log files (wrong `__path__` glob pattern).
  - Pipeline stage errors (regex parse failures silently dropping lines).
  - Rate limiting -- Loki rejecting pushes with `429 Too Many Requests` (check `loki_distributor_ingester_append_failures_total`).

**Step 3: Test label selectors**

Start with the broadest possible query and narrow down:

```logql
# Does this namespace have ANY logs?
{namespace="production"}

# Does this specific service have logs?
{namespace="production", service="order-api"}

# Now add your filter
{namespace="production", service="order-api"} |= "error"
```

If `{namespace="production"}` returns results but `{service="order-api"}` doesn't, the `service` label isn't being set on those log streams. Check the agent's label configuration -- the label might be named `app` instead of `service`, or the relabeling pipeline isn't extracting it.

**Step 4: Check time range**

Loki stores logs in time-bounded chunks. If you're querying "Last 5 minutes" but the logs were ingested with a timestamp from an hour ago (clock skew, delayed ingestion), they won't appear. Try expanding the time range.

---

**Scenario B: Query timing out**

**Step 1: Understand why Loki queries are slow**

Loki's query performance depends almost entirely on how many chunks it must decompress and scan. The label selector determines which chunks are read; the line filter scans content within those chunks. A broad label selector = reading many chunks = slow.

**Step 2: Check if the label strategy is forcing full scans**

```logql
# BAD: scans ALL logs in the namespace (potentially TB of data)
{namespace="production"} |= "order failed"

# GOOD: narrows to one service first, then filters content
{namespace="production", service="order-api"} |= "order failed"
```

If you only have one or two labels (like `namespace`), Loki must scan all streams in that namespace. The fix is to add more labels to the log streams at the agent level:

```yaml
# In Promtail/Alloy config -- add service label from pod metadata
pipeline_stages:
  - labels:
      service: ""  # extract from kubernetes pod labels
```

**Step 3: Check for too many streams (high label cardinality in Loki)**

```promql
# How many active streams does Loki have?
loki_ingester_streams_created_total

# Streams per tenant
loki_ingester_streams_created_total - loki_ingester_streams_removed_total
```

If stream count is extremely high, you might have high-cardinality labels on log streams (like `pod` with frequent rollouts, or `filename` with per-request log files). Unlike Prometheus, Loki labels should be LOW cardinality -- typically just `namespace`, `service`, `environment`, and maybe `level`. Everything else should be log content, not labels.

**Step 4: Restructure for performance**

- **Add coarse labels, not fine ones.** `service` and `level` (info/warn/error) are ideal. `pod` is acceptable if churn is low. `request_id` as a label will destroy Loki.
- **Use line filters before parsers.** `{service="order-api"} |= "error" | json` is faster than `{service="order-api"} | json | level="error"` because the simple string match happens before the expensive JSON parsing.
- **Reduce query time range.** "Last 1 hour" is orders of magnitude faster than "Last 7 days."
- **Use Loki's query limits.** Configure `max_query_length`, `max_query_series`, and `max_entries_limit_per_query` in Loki's config to prevent runaway queries from consuming all resources.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>24. Tell me about a time you designed or significantly overhauled a monitoring and alerting setup — what was the existing state, what tools did you choose and why, how did you decide what to monitor and alert on, and what impact did it have on incident response?</summary>

**What the interviewer is looking for:**
- Ability to assess an existing system critically and identify gaps.
- Thoughtful tool selection based on requirements, not hype.
- Structured approach to deciding what to monitor (RED/USE, SLOs, business metrics).
- Measurable impact on reliability and incident response.

**Suggested structure (STAR):**

1. **Situation** -- Describe the existing monitoring state. What was broken or missing? (e.g., "We had basic CloudWatch alarms but no service-level metrics. Incidents were discovered by customer complaints, not alerts.")

2. **Task** -- What was your goal? (e.g., "Reduce mean-time-to-detect from hours to minutes. Give the team dashboards they'd actually use during incidents.")

3. **Action** -- Walk through your decisions:
   - **Tool selection.** Why you chose specific tools (Prometheus vs Datadog, Loki vs ELK). Mention cost, operational complexity, team familiarity.
   - **What to monitor.** How you applied RED method for services, USE for infrastructure. How you identified the key SLIs (error rate, latency, availability).
   - **Alerting philosophy.** How you decided what pages vs what goes to Slack. How you set thresholds (based on SLOs, historical data, or business requirements).
   - **Rollout.** How you instrumented services (prom-client, OpenTelemetry), built dashboards, iterated based on team feedback.

4. **Result** -- Quantify the impact. (e.g., "MTTD dropped from ~2 hours to under 5 minutes. On-call stopped being a guessing game -- the dashboard told you where to look within 30 seconds of an alert.")

**Example outline to personalize:**

> "When I joined, our monitoring was scattered CloudWatch metrics with no service-level visibility. During incidents, engineers would SSH into boxes and grep logs. I proposed and implemented a PLG stack -- Prometheus for metrics, Loki for logs, Grafana for dashboards. I chose PLG over Datadog because we had 40+ services and Datadog's per-host pricing would have been $X/month. I built RED dashboards for each service tier, set up alerting rules based on our 99.9% SLO, and configured Alertmanager routing so critical alerts paged and warnings went to Slack. The biggest win was the fleet overview dashboard -- during the next incident, the on-call engineer identified the failing service in under a minute instead of the usual 30-minute investigation."

</details>

<details>
<summary>25. Describe a time you dealt with a cardinality explosion or a scaling issue in your metrics infrastructure — how did you discover it, what was the root cause, how did you fix it, and what guardrails did you put in place to prevent it from happening again?</summary>

**What the interviewer is looking for:**
- Understanding of cardinality as a concept and why it matters operationally.
- Systematic debugging approach (not just "we restarted Prometheus").
- Root cause analysis that goes beyond the symptom.
- Preventive measures that show systems thinking.

**Suggested structure:**

1. **Discovery** -- How did you notice? (e.g., Prometheus OOM alert, dashboards timing out, disk usage spike, or proactive monitoring of `prometheus_tsdb_head_series`.)

2. **Investigation** -- Walk through the diagnostic steps:
   - Checked TSDB status page for top metrics by series count.
   - Identified the offending metric and label (e.g., "An HTTP histogram with `path` label containing raw URLs -- 50K unique paths").
   - Traced back to the code change that introduced it (recent PR, new library, middleware added without review).

3. **Immediate fix** -- What you did to stop the bleeding:
   - Added `metric_relabel_configs` to drop the label at scrape time.
   - Deleted the historical series via admin API.
   - Deployed a code fix to normalize the label in the application.

4. **Prevention** -- Guardrails you established:
   - Cardinality alert on `prometheus_tsdb_head_series`.
   - PR review checklist for new metrics/labels.
   - CI check that scrapes `/metrics` from a test instance and counts series.
   - Documentation on label best practices shared with the team.

**Example outline to personalize:**

> "We got paged on a Saturday because Prometheus was consuming 28GB of RAM and approaching OOM. I checked the TSDB status endpoint and found that `http_request_duration_seconds_bucket` had 2.3 million series -- 10x what it should have been. The `path` label had 15,000 unique values because a recently deployed service was using `req.originalUrl` instead of the route pattern. I immediately added a relabel rule to drop the `path` label from that metric, deployed a code fix to normalize paths on Monday, and then set up a cardinality alert at 500K series with a Slack notification. I also added a section to our onboarding docs about label cardinality and made it a required check in our PR template for any code that touches metrics."

</details>

<details>
<summary>26. Tell me about a time you used metrics, logs, and dashboards together to debug a production incident — walk through the investigation flow, which signals led you to the root cause, and how the observability setup helped (or failed) you.</summary>

**What the interviewer is looking for:**
- A clear narrative that shows you can correlate signals across different observability pillars.
- The investigation flow: alert/symptom -> metrics for scope -> logs for detail -> root cause.
- Honest reflection on what worked and what didn't in your setup.
- Ability to tell a structured incident story under time pressure.

**Suggested structure:**

1. **The alert/symptom** -- How did you learn something was wrong? (Alert fired, customer reported, on-call page.)

2. **Metrics for scope and triage** -- What did dashboards tell you?
   - Which service was affected (error rate spike on the RED dashboard).
   - How widespread (one endpoint or all? one region or all?).
   - When it started (correlate with deployments, traffic changes).
   - What resources looked abnormal (CPU, memory, connections, queue depth).

3. **Logs for root cause** -- How did you drill into the specifics?
   - Used Grafana's "Explore" or a dashboard data link to jump from the metric spike to logs for that service and time range.
   - Filtered logs by error level, found the specific exception/stack trace.
   - Identified the root cause (e.g., database connection pool exhaustion, a bad config deploy, upstream dependency timeout).

4. **Resolution and retrospective** -- What you did to fix it and what you improved in the observability setup afterward.

**Example outline to personalize:**

> "At 2 AM, our HighErrorRate alert fired for the checkout service. I opened the service RED dashboard and saw 5xx errors spiking from 0.1% to 12% starting at 1:47 AM. The latency heatmap showed a bimodal distribution -- most requests were normal, but a chunk were hitting a 30-second timeout. I checked the resource dashboard -- CPU and memory were fine, but the 'active database connections' gauge was pegged at the pool max (50). I jumped to Loki from the dashboard link, filtered for `{service="checkout"} |= "error"`, and found repeated `Error: Connection pool exhausted` messages. Tracing back, a migration job had been deployed at 1:45 AM that held long transactions. I killed the migration, connections freed up, and errors dropped to zero within 2 minutes. In the retro, I added a dedicated dashboard panel for connection pool usage and an alert on pool saturation > 80%."

**Key point:** Show the flow between observability tools. The interviewer wants to see that you use metrics for breadth ("what and where"), logs for depth ("why"), and dashboards for speed ("how quickly can I orient myself"). If something failed (e.g., "we didn't have connection pool metrics, so it took 20 extra minutes to identify"), that's valuable too -- it shows you learn from gaps.

</details>

<details>
<summary>27. Describe a time you tackled alert fatigue on your team — what was causing it, how did you measure the problem, what changes did you make to alerting rules or routing, and how did you know the improvements worked?</summary>

**What the interviewer is looking for:**
- Recognition that alert fatigue is a real operational risk (people stop responding to alerts).
- Data-driven approach to identifying the problem (not just "alerts were annoying").
- Specific, concrete changes to alerting rules, routing, or thresholds.
- Measurable improvement, not just "it felt better."

**Suggested structure:**

1. **The problem** -- Describe the alert fatigue situation:
   - How many alerts per week/day was the team receiving?
   - What was the signal-to-noise ratio? (e.g., "80% of pages required no action.")
   - What impact was it having? (Engineers ignoring alerts, slow incident response, on-call burnout.)

2. **Measuring it** -- How did you quantify the problem?
   - Counted alerts per week from Alertmanager or PagerDuty analytics.
   - Categorized alerts: actionable vs noise vs duplicate vs resolved-before-anyone-looked.
   - Tracked acknowledgment time -- increasing ack time = fatigue signal.

3. **Changes made** -- Walk through specific improvements:
   - **Deleted or downgraded noisy alerts.** CPU > 70% doesn't need a page. Moved to Slack warning or deleted entirely.
   - **Added `for` durations.** Alerts without `for` were flapping. Added 5-10 minute `for` clauses.
   - **Added minimum traffic guards.** Error rate alerts firing on low-traffic services at 2 AM (1 request fails = 100% error rate). Added `and rate(...) > 1` clauses.
   - **Improved grouping.** 50 individual pod alerts became 1 grouped service alert via Alertmanager `group_by`.
   - **Added inhibition rules.** Database-down alert now suppresses all service alerts that depend on it.
   - **Separated severity levels.** Only true user-facing outages page. Everything else goes to Slack.

4. **Measuring improvement** -- How you verified it worked:
   - Alert count dropped from X/week to Y/week.
   - Percentage of actionable alerts went from 20% to 80%.
   - On-call satisfaction survey improved.
   - MTTA (mean time to acknowledge) decreased because engineers trusted alerts again.

**Example outline to personalize:**

> "Our on-call was getting 40+ alerts per week, and engineers started ignoring the Slack channel entirely. I pulled PagerDuty analytics for the past month and categorized every alert: 35% were flapping (CPU/memory threshold oscillating), 25% were duplicates (same root cause triggering 10+ alerts), 20% resolved before anyone looked, and only 20% were genuinely actionable. I made three changes: added `for: 5m` to all alerts that lacked it (killed flapping), added inhibition rules so database-down suppressed downstream service alerts (killed duplicates), and moved all non-customer-facing alerts from PagerDuty to Slack (reduced pages by 60%). After a month, weekly pages dropped from 40 to 8, and the team's on-call NPS went from -20 to +15."

</details>
