> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Cost Visibility with OpenCost

> Demo showing how OpenCost maps Kubernetes resource requests and usage to dollar costs, using UI and API to find overprovisioning and enable rightsizing and scaling to reduce waste.

Many platform engineers are surprised to learn their Kubernetes clusters are wasting money — not because pods are idle, but because workloads are overprovisioned. A team may request 2 CPUs per pod while actually using 50 millicores. Multiply that across dozens of deployments and the wasted spend adds up.

The root cause is visibility: without tooling you can’t easily map spend to the team, namespace, or workload responsible. [OpenCost](https://opencost.io) fixes this by mapping Kubernetes resource usage to dollar cost per namespace, deployment, and pod in (near) real time.

In this walkthrough we'll:

* Verify OpenCost and Prometheus are running.
* Inspect the OpenCost UI.
* Map pod and deployment resource requests to the costs OpenCost attributes.
* Show how to make changes (scale/rightsizing) and where to look for the impact.
* Demonstrate the OpenCost API for automation.

Prerequisites

* A running Kubernetes cluster.
* OpenCost deployed in the `opencost` namespace.
* Prometheus available (OpenCost consumes Prometheus metrics).

Check that the OpenCost pod is running:

```bash theme={null}
kubectl get pods -n opencost
NAME                          READY   STATUS    RESTARTS   AGE
opencost-5b676b669f-pc9hk     2/2     Running   3          78m
```

OpenCost consumes metrics from Prometheus. In this environment Prometheus runs in the `prometheus-system` namespace.

For day-to-day exploration use the [OpenCost UI](https://docs.opencost.io) — it provides a visual breakdown and lets you drill down by namespace, deployment, or pod. Below is an example OpenCost dashboard showing namespace cost allocation.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Demo-Cost-Visibility-with-OpenCost/opencost-cost-allocation-dashboard-chart-table.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=888ecaabd0ecb3557b558df35a1a952e" alt="The image shows a cost allocation dashboard from OpenCost, displaying a chart and table that breaks down costs by namespace for a selected date. It includes details on CPU, GPU, RAM, personal volume, efficiency, and total costs in USD." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Demo-Cost-Visibility-with-OpenCost/opencost-cost-allocation-dashboard-chart-table.jpg" />
</Frame>

List relevant pods across namespaces so we can map costs back to workloads:

```bash theme={null}
kubectl get pods --all-namespaces
NAMESPACE           NAME                                           READY   STATUS    RESTARTS   AGE
opencost            opencost-5b676b669f-pc9hk                      2/2     Running   3          78m
prometheus-system   prometheus-server-86854bc5f-qbmn5              1/1     Running   0          78m
prometheus-system   prometheus-kube-state-metrics-978b594b-cnzq8   1/1     Running   0          78m
team-backend        api-d5c5679c                                   2/2     Running   0          60m
team-data           cache-2dcff5f6d-2jm2s                          1/1     Running   0          100m
team-frontend       web-55fcdc9cf-49f9b                             1/1     Running   0          32m
team-frontend       web-5592c8e0f-7h2qm                             1/1     Running   0          32m
team-frontend       web-55fcdc9cf-lwz4q                             1/1     Running   0          32m
```

Summary of the running workloads (observations from the pod list):

* team-frontend: several `web` replicas (3 total in this sample).
* team-backend: two `api` replicas.
* team-data: a single `cache` replica (Redis).

Inspect each deployment’s resource requests — OpenCost attributes cost primarily from resource requests (requests/limits) rather than instantaneous CPU usage, so requests drive allocated cost.

Inspect the frontend deployment (`web`) in `team-frontend`:

```bash theme={null}
kubectl describe deploy web -n team-frontend
```

In this deployment the container requests 500m CPU and 512Mi memory and the deployment is configured with 3 replicas (500m \* 3 = 1.5 CPU requested across the namespace). Those requests drive the cost allocation, even if actual usage is much lower.

Inspect the backend API deployment:

```bash theme={null}
kubectl describe deploy api -n team-backend
```

Example (trimmed) output:

```bash theme={null}
Name:                   api
Namespace:              team-backend
Replicas:               2 desired | 2 updated | 2 total | 2 available
Pod Template:
  Containers:
   nginx:
    Image:    nginx
    Limits:
      cpu:     200m
      memory:  256Mi
    Requests:
      cpu:      100m
      memory:   128Mi
Conditions:
  Available        True    MinimumReplicasAvailable
  Progressing      True    NewReplicaSetAvailable
```

The backend API requests 100m CPU and 128Mi memory per replica, so with 2 replicas it requests \~200m CPU in total.

Inspect the cache deployment in `team-data` (Redis):

```bash theme={null}
kubectl describe deploy cache -n team-data
```

Example (trimmed) output:

```bash theme={null}
Name:                   cache
Namespace:              team-data
Replicas:               1 desired | 1 updated | 1 total | 1 available
Pod Template:
  Containers:
   redis:
    Image:    redis
    Limits:
      cpu:  500m
    Requests:
      cpu:     250m
      memory:  1Gi
Conditions:
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
Events:
  Normal  ScalingReplicaSet  38m  deployment-controller  Scaled up replica set cache from 1 to 2
  Normal  ScalingReplicaSet  34m  deployment-controller  Scaled down replica set cache from 2 to 1
```

Although Redis requests 250m CPU and 1Gi memory, this example only has a single replica.

Deployment resource-request summary

| Deployment | Namespace       | Replicas | CPU request (per pod) | Memory request (per pod) | Total CPU requested |
| ---------- | --------------- | -------: | --------------------: | -----------------------: | ------------------: |
| web        | `team-frontend` |        3 |                  500m |                    512Mi |             1.5 CPU |
| api        | `team-backend`  |        2 |                  100m |                    128Mi |             0.2 CPU |
| cache      | `team-data`     |        1 |                  250m |                      1Gi |            0.25 CPU |

Why is Team Frontend more expensive in OpenCost?

OpenCost attributes cost based on resource requests (and observed usage) to allocate spend to namespaces and workloads. Team Frontend requested 500m per replica across three replicas (1.5 CPUs total requested), while Backend requested only 100m per replica with two replicas (0.2 CPUs total). Those request differences explain the larger cost for frontend.

<Callout icon="lightbulb" color="#1CB2FE">
  OpenCost maps requested resources (requests/limits) and observed usage to cost. Over-requesting increases allocated cost even if actual usage is low — that’s why rightsizing requests and replica counts matters.
</Callout>

Actionable steps: scale or rightsize based on OpenCost insights

You can use the cost signals to change replica counts or adjust requests. For example, scale the frontend down and scale the cache up to better match demand (and to move costs where they belong):

```bash theme={null}
# Scale frontend web down to 1 replica
kubectl scale deployment web -n team-frontend --replicas=1
# Scale cache up to 2 replicas
kubectl scale deployment cache -n team-data --replicas=2
```

Example responses:

```bash theme={null}
deployment.apps/web scaled
deployment.apps/cache scaled
```

Return to the [OpenCost UI](https://docs.opencost.io) and refresh. As Prometheus metrics are aggregated you should see namespace and deployment costs adjust to reflect the new requested resources and replica counts.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Demo-Cost-Visibility-with-OpenCost/opencost-cost-allocation-dashboard-pie-chart.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=9b66adb58e55da308836233a7c99cd0f" alt="The image is a cost allocation dashboard from OpenCost displaying a pie chart and a table showing namespace daily costs, with details on CPU, GPU, RAM, PV, efficiency, and total cost." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Demo-Cost-Visibility-with-OpenCost/opencost-cost-allocation-dashboard-pie-chart.jpg" />
</Frame>

OpenCost API for automation

If you prefer automation or integration, OpenCost exposes an API you can query. Example: get compute allocation aggregated by namespace over the last hour and format results with `jq`:

```bash theme={null}
curl -sG "http://localhost:3904/allocation/compute?window=1h&aggregate=namespace" | \
jq -r '
  (.data[0] // {}) 
  | to_entries[]
  | {namespace: .key, cpuCost: .value.cpuCost, ramCost: .value.ramCost}
'
```

Sample JSON output (trimmed):

```json theme={null}
{
  "namespace": "kube-flannel",
  "cpuCost": 0.00271,
  "ramCost": 0.00018
}
{
  "namespace": "kube-system",
  "cpuCost": 0.02307,
  "ramCost": 0.00085
}
{
  "namespace": "opencost",
  "cpuCost": 0.00054,
  "ramCost": 0.00039
}
{
  "namespace": "team-backend",
  "cpuCost": 0.00543,
  "ramCost": 0.00091
}
{
  "namespace": "team-data",
  "cpuCost": 0.00764,
  "ramCost": 0.00041
}
{
  "namespace": "team-frontend",
  "cpuCost": 0.05045,
  "ramCost": 0.00676
}
```

Use cases for the API:

* Scheduled cost reports.
* CI/CD gates to prevent cost spikes from PRs.
* Automated alerts when request-to-usage ratios indicate overprovisioning.

Conclusion

For exploration and fast triage, the [OpenCost UI](https://docs.opencost.io) is convenient; for automation and programmatic workflows, use the API. Either way, OpenCost provides the visibility needed to identify overprovisioning, allocate cost to teams, and take concrete remediation steps like rightsizing requests or adjusting replica counts.

Links and references

* [OpenCost website](https://opencost.io)
* [OpenCost documentation (UI & API)](https://docs.opencost.io)
* [Prometheus](https://prometheus.io)
* jq: [https://stedolan.github.io/jq/](https://stedolan.github.io/jq/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/4dfb9257-7cc3-4837-b314-207a8d758584" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/9d957351-7ded-4b48-a240-cd70699a17e1" />
</CardGroup>
