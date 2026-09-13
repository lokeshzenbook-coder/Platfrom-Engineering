> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Metrics Architecture Collection Storage and Querying

> Overview of Prometheus metrics architecture, collection, metric types, labeling, PromQL querying and Kubernetes integration for reliable, scalable observability and consistent dashboards

Metrics are the first pillar of observability — they answer "what is happening" in your system. Collecting metrics at scale, however, introduces challenges: fragmentation, inconsistent labeling, and difficulty correlating signals across components.

By the end of this lesson you'll understand how Prometheus collects metrics, the three primary metric types and when to use each, how labels enable multi-dimensional queries, and enough PromQL to build Grafana dashboards and alerts.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/prometheus-metrics-learning-objectives.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=017c6888288b2b7a2d2fd8a84dc9829c" alt="The image lists learning objectives related to metrics collection and Prometheus, including understanding architectures, the pull-based model, differentiating metric types, using labels, and writing PromQL queries." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/prometheus-metrics-learning-objectives.jpg" />
</Frame>

## Why consistent labeling matters — a concrete example

A logistics platform ran 60 microservices across three Kubernetes clusters. Each team chose different metric approaches. During a database connection-pool exhaustion incident, the platform team needed to correlate:

* API error rates
* Database connection counts
* Pod memory usage

Those metrics existed across three different systems, with different time granularities and distinct label sets — there was no reliable way to join them. What should have been a five-minute correlation became a ninety-minute merge of spreadsheets.

After migrating to Prometheus and enforcing consistent labeling, the same correlation was a single-line PromQL query and returned instant results.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/microservices-teams-tasks-time-illustration.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=fee2b7e02c8c269845be0b0f982a77ca" alt="The image illustrates three teams handling microservices, with respective tasks and time taken: Team 1 pushed StatsD to Graphite in 10 seconds, Team 2 exposed JSON endpoints in 30 seconds, and Team 3 adopted Prometheus exporters in 60 seconds." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/microservices-teams-tasks-time-illustration.jpg" />
</Frame>

## The problem: data everywhere

Without a unified approach you get:

* No single source of truth
* Format fragmentation
* Push-sprawl
* Difficult correlation and inconsistent dashboards

A unified collection architecture prevents multiple dashboards from telling different stories about the same incident. Prometheus is the de-facto standard for that unified approach.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/metrics-types-common-issues-outline.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=1ae0c8c863bf6f5949920e3dfbb045bf" alt="The image outlines common types of metrics teams have and explains why these metrics setups often fail, highlighting issues like lack of a single source of truth, format fragmentation, push sprawl, and lack of correlation." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/metrics-types-common-issues-outline.jpg" />
</Frame>

## Prometheus architecture — four conceptual layers

Prometheus provides a predictable, Kubernetes-native collection model. Conceptually it has four layers:

* Targets: applications expose a `/metrics` HTTP endpoint.
* Service discovery: finds targets automatically (Kubernetes, Consul, etc.).
* Prometheus server: scrapes targets and stores time-series samples in its TSDB.
* PromQL: queries the time-series database for analysis, dashboards, and alerts.

Why pull instead of push?

* Central control: scrape intervals and targets are controlled centrally.
* Health detection and debugging: if a target stops responding, Prometheus notices immediately and you can inspect the exact `/metrics` output.

Operational benefits:

* No client-side scrape scheduling
* Easier debugging (query the target directly)
* Strong Kubernetes-native integration

<Callout icon="warning" color="#FF6B6B">
  Prometheus is primarily pull-based. For short-lived batch jobs that can't be scraped, the official escape hatch is the Pushgateway. Use Pushgateway sparingly — it’s an exception, not the default pattern. See the Prometheus push practices for details: [https://prometheus.io/docs/practices/pushing/](https://prometheus.io/docs/practices/pushing/)
</Callout>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/prometheus-pull-collection-process-diagram.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=9609a50897b4115ddef1ca24432bcd81" alt="The image is a diagram explaining Prometheus' pull-based collection process, highlighting components like scrape targets, service discovery, time series database, and PromQL. It also lists benefits such as central control, health monitoring, no client-side configuration, ease of debugging, and Kubernetes-native features." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/prometheus-pull-collection-process-diagram.jpg" />
</Frame>

## Metric types — what to use and how to query them

Choosing the correct metric type and query pattern is critical. Applying the wrong PromQL function to a metric type is the most common mistake.

| Metric Type | What it answers                                                                     | How to compute/query                                                                | Notes / Example                                                                                                   |
| ----------- | ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Counter     | "How many" (monotonic, non-decreasing)                                              | Use range-vector functions: `rate()` (per-second) or `increase()` over a window     | Counters can reset on restart — `rate()` and `increase()` handle resets. Example: `rate(http_requests_total[5m])` |
| Gauge       | "How much right now" (can increase/decrease)                                        | Query instant value directly or use other non-monotonic functions                   | Do not use `rate()` on gauges. Example: `container_memory_usage_bytes{namespace="production"}`                    |
| Histogram   | "How long" or distribution (bucketed cumulative counters, plus `_count` and `_sum`) | Apply `rate()` to `_bucket` counters and use `histogram_quantile()` for percentiles | Example: `histogram_quantile(0.95, sum by (le) (rate(request_duration_seconds_bucket[5m])))`                      |

Key rules:

* Avoid `rate()` on gauges — use the raw gauge or appropriate functions.
* To interpret counters, always use `rate()` or `increase()`.
* For histograms, operate on the bucket counters (often with `rate()`) and use `histogram_quantile()` for percentiles.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/metrics-counter-gauge-histogram-diagram.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=fc5e7b0521019e07fcc7165cfd42d1c2" alt="The image describes three types of metrics: Counter, Gauge, and Histogram, each explaining their function, example, and query usage." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/metrics-counter-gauge-histogram-diagram.jpg" />
</Frame>

## Labels: dimensionality by design

Labels are one of Prometheus’s most powerful features. Without labels you’d need a unique metric name for every dimension combination — that doesn’t scale. Labels let you:

* Filter subsets (e.g., by `namespace`, `pod`, `service`)
* Group and aggregate (e.g., `sum by (service)`)
* Correlate metrics across services when labels are consistent

Kubernetes injects many useful labels automatically (for example, `namespace`). Establish platform-wide labeling conventions (team, service, environment) to make cross-service queries straightforward.

## PromQL patterns you’ll use daily

PromQL can look intimidating, but most needs are covered by three patterns:

1. Instant vector — current values
2. Range vector + function — rates/changes over time
3. Aggregation — group and summarize by labels

Examples:

```promql theme={null}
# Instant vector: current memory per pod (instant query)
container_memory_usage_bytes{namespace="production"}
```

```promql theme={null}
# Range vector + rate: requests per second over a 5m window
rate(http_requests_total[5m])
```

```promql theme={null}
# Aggregation: total error rate (5xx) by service over 5m
sum by (service) (
  rate(http_requests_total{code=~"5.."}[5m])
)
```

<Callout icon="lightbulb" color="#1CB2FE">
  Remember: counters may reset when a process restarts. Functions like `rate()` and `increase()` are resilient to resets. Use gauges for instantaneous values and avoid `rate()` on non-monotonic series.
</Callout>

## Prometheus in Kubernetes — discovery and common targets

Common discovery methods in Kubernetes:

* Pod annotations: quick for simple setups. Example:

```yaml theme={null}
# Pod annotation example
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

* ServiceMonitor: CRD from the Prometheus Operator (recommended for scale). ServiceMonitors let you declaratively manage scrape targets and relabeling.
* Static configuration: use for external systems or legacy services that are not discovered via Kubernetes.

Typical metric sources:

* kube-state-metrics: exposes desired/actual state of Kubernetes objects — deployments, replicasets, pods. ([https://github.com/kubernetes/kube-state-metrics](https://github.com/kubernetes/kube-state-metrics))
* node-exporter: node-level hardware metrics — CPU, memory, disk, network. ([https://github.com/prometheus/node\_exporter](https://github.com/prometheus/node_exporter))
* cAdvisor: container-level CPU and memory metrics. ([https://github.com/google/cadvisor](https://github.com/google/cadvisor))
* Your app’s `/metrics` endpoint: business and application-specific metrics.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/prometheus-kubernetes-integration-overview.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=4f5baf86e0210c514c1bb9f5dba7b90d" alt="The image provides an overview of integrating Prometheus in Kubernetes, outlining methods such as pod annotations, ServiceMonitor, and static configuration, along with resources like kube-state-metrics, node-exporter, and cAdvisor. It briefly describes each component’s role in monitoring metrics within Kubernetes clusters." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/prometheus-kubernetes-integration-overview.jpg" />
</Frame>

## Quick reference — common PromQL snippets

* Current memory per pod:

```promql theme={null}
container_memory_usage_bytes{namespace="production"}
```

* Requests per second (5m):

```promql theme={null}
rate(http_requests_total[5m])
```

* Aggregate 5xx error rate by service:

```promql theme={null}
sum by (service) (
  rate(http_requests_total{code=~"5.."}[5m])
)
```

* 95th percentile latency from histograms:

```promql theme={null}
histogram_quantile(0.95, sum by (le) (rate(request_duration_seconds_bucket[5m])))
```

## Summary — core takeaways

* Prometheus is pull-based: this enables central control and automatic health detection.
* There are three primary metric types (counter, gauge, histogram). Use the proper query patterns:
  * Counters → `rate()` / `increase()`
  * Gauges → instant values or non-monotonic functions
  * Histograms → operate on bucket counters and use `histogram_quantile()`
* Labels make metrics dimensional and enable filtering, grouping, and correlation. Enforce consistent labeling conventions.
* PromQL supports instant, range, and aggregation queries — these cover most dashboarding and alerting needs.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/prometheus-key-takeaways-pull-metrics-queries.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=b5866c12116107c3aa1d2418405132a5" alt="The image outlines four key takeaways about Prometheus: its pull-based design, metric types and query patterns, the importance of labels for metrics, and the use of PromQL as a query language." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Metrics-Architecture-Collection-Storage-and-Querying/prometheus-key-takeaways-pull-metrics-queries.jpg" />
</Frame>

## Links and further reading

* Prometheus docs: [https://prometheus.io/docs/](https://prometheus.io/docs/)
* PromQL basics: [https://prometheus.io/docs/prometheus/latest/querying/basics/](https://prometheus.io/docs/prometheus/latest/querying/basics/)
* Prometheus Operator and ServiceMonitor: [https://github.com/prometheus-operator/prometheus-operator](https://github.com/prometheus-operator/prometheus-operator)
* kube-state-metrics: [https://github.com/kubernetes/kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
* node-exporter: [https://github.com/prometheus/node\_exporter](https://github.com/prometheus/node_exporter)
* Pushgateway best practices: [https://prometheus.io/docs/practices/pushing/](https://prometheus.io/docs/practices/pushing/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/9bd090c8-8d99-4742-b50c-ae63e516e6b9/lesson/dafd2315-36be-4318-b3e4-cc408da620c5" />
</CardGroup>
