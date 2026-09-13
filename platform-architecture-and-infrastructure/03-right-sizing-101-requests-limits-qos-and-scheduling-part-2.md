> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Right Sizing 101 Requests Limits QoS and Scheduling Part 2

> Explains Kubernetes resource requests, limits, QoS classes, and scheduling with guidance on right-sizing, profiling, and monitoring to improve utilization and avoid outages.

Requests and limits are the two primary knobs you use to control how Kubernetes assigns and enforces CPU and memory for Pods. Understanding them clearly will help you improve cluster utilization, avoid outages, and tune scheduling behavior.

* Requests: the guaranteed minimum resources a Pod needs. The scheduler uses requests to decide where to place a Pod. For example, if a Pod requests `1` CPU and `256Mi` memory, the scheduler will only place it on a node that reports at least those allocatable resources available.
* Limits: the maximum resources a Pod is allowed to consume at runtime. Limits are enforced by the kubelet via cgroups. Exceeding a CPU limit causes kernel-level throttling; exceeding a memory limit can cause the container to be terminated (OOMKilled).

In short: requests affect scheduling; limits affect runtime behavior.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/requests-limits-resource-management-diagram.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=3ad67fc485dbe5dc3b4ec75c8802c036" alt="The image explains the difference between &#x22;Requests&#x22; and &#x22;Limits&#x22; in resource management, highlighting their purposes, meanings, and consequences if exceeded." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/requests-limits-resource-management-diagram.jpg" />
</Frame>

Scheduling example — why requests matter

* Node capacity: 4 CPUs available.
* Three Pods arrive. Each Pod requests `1` CPU and sets a limit of `2` CPUs.
  * Scheduler sums requests: `1 + 1 + 1 = 3` CPUs requested.
  * Node has `4` CPUs allocatable, so all three Pods are admitted. `1` CPU remains allocatable.
* If a fourth Pod requests `2` CPUs, the scheduler will NOT schedule it because only `1` CPU remains allocatable.

Important: the scheduler only looks at requests during placement. It does not check whether those CPUs are idle or busy at the moment of scheduling.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/cpu-requests-limits-node-allocation.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=4e8824d654131dee7d6dc8fae9f9afa0" alt="The image explains how requests and limits work in a computing node, showing the allocation of CPU resources across different pods, with a node capacity of 4 CPUs and 3 CPUs already requested, leaving 1 CPU available. Pod D cannot be scheduled because the CPU is full." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/cpu-requests-limits-node-allocation.jpg" />
</Frame>

Runtime behavior — limits and contention

* In the example above, each Pod is guaranteed `1` CPU (its request) but allowed to burst up to `2` CPUs (its limit).
* If all three Pods simultaneously try to use their full limits, they would collectively want `6` CPUs while the node only has `4`. The kernel enforces CPU fairness using cgroups and the Completely Fair Scheduler (CFS), so CPU-bound containers will be throttled and see degraded throughput/latency.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/cpu-requests-limits-pods-node-diagram.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=9e2e6ad3cf46532b345e1dd9e6630f4f" alt="The image explains how CPU requests and limits work for pods within a node that has 4 CPUs available, illustrating that each pod is guaranteed 1 CPU but can burst up to 2 CPUs. If all pods burst their CPU usage, a total of 6 CPUs would be needed, but only 4 are available." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/cpu-requests-limits-pods-node-diagram.jpg" />
</Frame>

Quality of Service (QoS) classes

Kubernetes assigns a QoS class to every Pod based on the presence and relationship of requests and limits. QoS affects eviction order under memory pressure and can impact scheduling decisions indirectly.

| QoS class  | Criteria                                                           |                  Eviction priority | When to use                                                  |
| ---------- | ------------------------------------------------------------------ | ---------------------------------: | ------------------------------------------------------------ |
| Guaranteed | Every container in Pod has CPU and memory requests equal to limits |  Highest protection (evicted last) | Critical services needing predictable performance            |
| Burstable  | Pod has requests, or requests \< limits for at least one resource  | Middle (evicted before Guaranteed) | Typical services that can benefit from bursting              |
| BestEffort | No requests and no limits for any container                        |             Lowest (evicted first) | Noncritical work, testing, or batch jobs that can be dropped |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/kubernetes-qos-classes-explained.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=2273495e4b971d3e446a2c14317372fb" alt="The image explains Kubernetes Quality of Service (QoS) classes—Guaranteed, Burstable, and BestEffort—based on CPU and memory requests and limits, along with their eviction order under memory pressure." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/kubernetes-qos-classes-explained.jpg" />
</Frame>

Operational difference: CPU vs Memory

* Hitting a CPU limit results in throttling — your application will run slower but is generally not terminated.
* Hitting a memory limit typically results in an immediate OOM kill — the container process can be terminated abruptly.

<Callout icon="warning" color="#FF6B6B">
  Memory limit breaches are often fatal for a Pod. Always set memory limits intentionally and monitor OOMKilled events — they can indicate that requests were underestimated or the workload requires more memory headroom.
</Callout>

Right-sizing workflow — pick the right request and limit values

1. Profile actual usage
   * Run workloads under a realistic, representative load window (for example, 1 week) and collect CPU and memory metrics.
   * Tools: [Prometheus](https://prometheus.io/), `kubectl top`, Grafana, or other cluster monitoring solutions.
2. Choose requests from observed usage
   * A common approach is to set the request to the 95th percentile of observed usage for the profiling window. This captures normal spikes while avoiding over-provisioning.
3. Set reasonable limits
   * Set limits higher than requests to allow bursts. Common guidelines: `limits = 2–3× requests` for many workloads.
   * Some teams choose not to set CPU limits for latency-sensitive services but still set memory limits (memory limits should almost always be enforced).
4. Monitor and iterate
   * Watch CPU throttling metrics and OOMKilled events; validate that latency and throughput remain acceptable.
   * Use automation where possible — the Vertical Pod Autoscaler (VPA) can recommend request values based on observed in-cluster usage.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/right-sizing-resource-workflow-diagram.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=a3528b85de85945ff78b7cb9b9268947" alt="The image outlines a four-step workflow for right-sizing resources, including profiling actual usage, setting requests at the 95th percentile, setting limits at 2-3 times the request, and monitoring and adjusting performance." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-2/right-sizing-resource-workflow-diagram.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Four practical takeaways:

  1. Requests drive scheduling; limits enforce runtime. Confusing them risks outages or wasted nodes.
  2. QoS classes determine eviction order—use Guaranteed (requests == limits) for critical workloads.
  3. CPU throttles; memory kills. Set memory limits carefully and monitor OOMKilled events.
  4. Profile first: use the 95th percentile for requests, set limits 2–3× higher, and validate with monitoring or VPA.
</Callout>

References and further reading

* Kubernetes documentation — Resource Management for Pods and Containers: [https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
* Prometheus monitoring: [https://prometheus.io/](https://prometheus.io/)
* kubectl top: [https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#top](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#top)
* Vertical Pod Autoscaler (VPA): [https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)

Right-sizing reduces waste, improves scheduling accuracy, and lowers the risk of performance regressions and production failures. Follow a measurement-first approach and iterate using data and observability.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/197f7f9c-c8bd-4515-890d-5026b0471809" />
</CardGroup>
