> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Visualization and Dashboards Turning Data into Insight

> How to connect Grafana to Prometheus and design dashboards and panels for rapid triage, monitoring, and visualization best practices

This lesson assumes metrics are collected by Prometheus and alerts are handled by Alertmanager. Here we focus on the visualization layer: Grafana. Dashboards are how you triage incidents, spot trends, and communicate platform health.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/visualization-dashboards-data-insight.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=d5fc418e474f73b992b8f1c99b8aa781" alt="The image is a blue gradient slide that reads &#x22;Visualization and Dashboards&#x22; and &#x22;Turning Data Into Insight,&#x22; with a copyright note for KodeKloud." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/visualization-dashboards-data-insight.jpg" />
</Frame>

Grafana converts time series into context-rich visuals so teams answer operational questions quickly. In this lesson you will learn how to connect Grafana to Prometheus, pick appropriate panel types, and structure dashboards optimized for quick triage.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/grafana-prometheus-learning-objectives-outline.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=65e0847df485f144f0532e4a3c46a692" alt="The image outlines learning objectives related to Grafana and Prometheus, highlighting understanding Grafana's role and configuring a Prometheus data source in Grafana." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/grafana-prometheus-learning-objectives-outline.jpg" />
</Frame>

Why this matters

Imagine a flash sale where latency spikes. An engineer manually queries Prometheus in the UI — each PromQL run takes time and context is missing. A colleague opens a pre-built Grafana dashboard and finds the root cause in under two minutes: a downstream payment service exhausted its connection pool. After that incident, Grafana dashboards became the mandatory first stop. Mean time to identify dropped from twenty minutes to under three.

What raw metrics look like:

```plaintext theme={null}
container_cpu_usage_seconds_total{container="web"} 4521.38
container_cpu_usage_seconds_total{container="db"} 1893.21
container_memory_usage_bytes{container="web"} 238901248
http_requests_total{code="200"} 1482901
http_requests_total{code="500"} 4291
kube_pod_status_phase{phase="Running"} 84
... 99,994 more time series
```

Raw metrics are essential but hard to interpret quickly. Grafana transforms those series into panels and dashboards that answer common questions like:

* Is the platform healthy?
* Which services have increasing error rates?
* Are we approaching resource limits?

How Grafana works

Grafana is organized into three layers:

| Layer        | Purpose                                             | Examples                                               |
| ------------ | --------------------------------------------------- | ------------------------------------------------------ |
| Data sources | Connectors to metric stores                         | `Prometheus`, `CloudWatch`                             |
| Dashboards   | Collections of panels organized to answer a concern | Service overview dashboards, cluster health dashboards |
| Panels       | Individual visualizations driven by queries         | Time series graphs, stat/gauge panels, tables          |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/prometheus-kubernetes-datasources-dashboards-panels.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=ca5e423e9e08df206f4c604a3648ed64" alt="The image describes the components for using Prometheus in Kubernetes, illustrating the relationships between datasources, dashboards, and panels. It specifies datasources like Prometheus and CloudWatch, dashboards focusing on specific services, and panels for visualizations like graphs and charts." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/prometheus-kubernetes-datasources-dashboards-panels.jpg" />
</Frame>

Connecting Grafana to Prometheus

Step-by-step: configure the Prometheus data source in Grafana and verify connectivity.

1. Grafana → Settings → Data sources → Add data source.
2. Select "Prometheus".
3. Set the URL to the in-cluster Prometheus service, for example `http://prometheus:9090`. Do not use `localhost` unless Grafana runs on the same host as Prometheus.
4. Toggle `isDefault` (or "Default") if you want new panels to use this data source automatically.
5. Click Save & Test.

If you need to confirm service names in Kubernetes:

```bash theme={null}
kubectl get svc
```

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/grafana-prometheus-connection-steps.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=e28795b5b455e67c06b458afa14cf768" alt="The image outlines steps for connecting Grafana to Prometheus, including navigating to datasources, selecting Prometheus, setting the URL, and marking it as default." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/grafana-prometheus-connection-steps.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Make sure the Prometheus URL is reachable from the Grafana pod. If Grafana cannot reach the service, check Kubernetes network policies, the service name, and the service port (Prometheus default is `9090`).
</Callout>

Example provisioning YAML (for automated provisioning in Grafana):

```yaml theme={null}
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
```

Common pitfalls

| Problem       | How to diagnose/fix                                               |
| ------------- | ----------------------------------------------------------------- |
| Wrong URL     | Verify the service name and port: `kubectl get svc`               |
| Port mismatch | Confirm Prometheus is listening on `9090` or your custom port     |
| Not default   | Toggle `isDefault` so panels use the Prometheus source by default |

Choosing the right panel for the question

Match visualization to intent:

| Panel type  | When to use                               | Example query                                       |
| ----------- | ----------------------------------------- | --------------------------------------------------- |
| Time series | Trends and correlations over time         | `rate(http_requests_total[5m])`                     |
| Stat        | Single summary numbers (QPS, error count) | `sum(rate(http_requests_total[1m]))`                |
| Gauge       | Current utilization (CPU%, memory%)       | `node_memory_usage_bytes / node_memory_total_bytes` |
| Table       | Detailed lists and metadata               | `topk(10, container_memory_usage_bytes)`            |
| Pie chart   | Distribution across categories            | `sum by(status)(http_requests_total)`               |

Common PromQL examples

```promql theme={null}
# Requests per second (5-minute rate)
rate(http_requests_total[5m])

# Number of healthy scrape targets (up == 1)
count(up == 1)

# Memory utilization (example)
node_memory_usage_bytes / node_memory_total_bytes

# Top 10 containers by memory usage
topk(10, container_memory_usage_bytes)
```

Organizing dashboards for rapid triage

Design dashboards top-down so the most important signals are at the top and the details are at the bottom. A practical layout:

1. Status overview (first row)
   * Stat panels answering "is everything broken?" — use clear color thresholds.
2. Traffic & errors (second row)
   * Time series for request rate, error rate, latency; correlate with deployments.
3. Resources (third row)
   * Gauges and time series for CPU, memory, disk with thresholds.
4. Details (bottom row)
   * Tables and links to logs or service-specific dashboards for drill-down.

This flow mirrors triage: glance → confirm → drill in.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/dashboard-building-guide-overview-traffic-errors.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=5855655f1ae7badefc692e3f0ee16ba0" alt="The image is a guide on building effective dashboards, outlining four rows: Status Overview, Traffic & Errors, Resources, and Details, with tips on information flow, template variables, thresholds, and linking dashboards." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/dashboard-building-guide-overview-traffic-errors.jpg" />
</Frame>

Best practices

* Use template variables for service, namespace, and cluster so dashboards are reusable across environments.
* Set meaningful thresholds and color rules to make issues obvious.
* Link overview dashboards to service-specific dashboards for fast context switching.
* Pre-build panels for common incident questions so responders avoid creating PromQL under pressure.
* Keep panels focused: one question per panel.

Bringing it all together — quick checklist

| Step | Action                                                                        |
| ---- | ----------------------------------------------------------------------------- |
| 1    | Add and verify the Prometheus data source in Grafana                          |
| 2    | Choose panel types that answer your operational questions                     |
| 3    | Structure dashboards top-down: overview → trends/errors → resources → details |
| 4    | Add variables, thresholds, and dashboard links for reuse and faster triage    |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/grafana-prometheus-setup-key-takeaways.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=7c4844a9ae5bacb5c0b99f08bad083cd" alt="The image presents key takeaways for setting up Grafana, focusing on its integration with Prometheus, datasource setup, panel type matching, and dashboard structuring." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Visualization-and-Dashboards-Turning-Data-into-Insight/grafana-prometheus-setup-key-takeaways.jpg" />
</Frame>

<Callout icon="warning" color="#FF6B6B">
  Grafana visualizes metrics but does not store them long-term — ensure Prometheus retention, remote-write, or a long-term store is configured if you need extended history for investigations.
</Callout>

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/9bd090c8-8d99-4742-b50c-ae63e516e6b9/lesson/1e920eab-1149-4389-8995-a12dd4293354" />
</CardGroup>
