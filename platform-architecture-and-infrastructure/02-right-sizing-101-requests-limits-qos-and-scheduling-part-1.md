> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Right Sizing 101 Requests Limits QoS and Scheduling Part 1

> How to set Kubernetes resource requests and limits, QoS classes, and right-sizing workflow to improve scheduling, reduce costs, and prevent evictions and OOM failures

Beyond big-picture platform architecture, this lesson covers something that affects every Pod you deploy: how to tell Kubernetes what your workload actually needs. Here you'll learn the difference between resource requests and limits, how to configure them correctly, how Kubernetes assigns QoS classes, and the eviction order under memory pressure. You’ll also get a practical right-sizing workflow to reduce cost and improve reliability.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/kubernetes-resource-management-learning-objectives.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=d5348231b84a5ebddd4fff12f4b8e1fc" alt="The image lists four learning objectives focused on resource management in Kubernetes, such as differentiating between resource requests and limits, configuring them correctly, understanding Quality of Service (QoS) classes, and describing eviction priority under memory pressure." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/kubernetes-resource-management-learning-objectives.jpg" />
</Frame>

Problem summary: developers must deploy services but often don’t know the right CPU/memory values. They guess—and guessing typically goes two ways: overprovision “just to be safe,” or underprovision and hope the workload holds.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/three-people-laptops-cpus-memory.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=bf831206332687f87ece26e1d745e70d" alt="The image depicts three people sitting at a table with laptops, thinking about CPU and memory, with the title &#x22;The Problem: The Guessing Game&#x22; and labeled &#x22;Over-provisioned.&#x22;" width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/three-people-laptops-cpus-memory.jpg" />
</Frame>

Real-world example: an e‑commerce company with 200 microservices on Kubernetes saw a 40% quarterly AWS bill increase even though traffic and revenue didn’t grow proportionally. Investigation revealed many Pods requested 8 CPUs but used only \~0.2 CPU on average.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/aws-cost-spike-40-percent-increase.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=7de79dd4d61e0afa23c9c106a32c8156" alt="The image outlines a cost spike problem, highlighting that an AWS bill increased by 40% quarter-over-quarter without proportional growth in revenue, traffic, or infrastructure costs." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/aws-cost-spike-40-percent-increase.jpg" />
</Frame>

That’s 40× overprovisioning per Pod.

Why that costs money: the scheduler uses resource requests (not instantaneous usage) when placing Pods. If a Pod requests 8 CPUs, the scheduler reserves those 8 CPUs on the node for placement decisions. Those reserved CPUs can’t be used by other Pods, and the autoscaler may add nodes to satisfy requested capacity—driving cloud spend up.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/over-provision-problem-cpu-usage-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=997563f444db9c97fdd397c20c02c355" alt="The image illustrates the &#x22;Over-Provision Problem,&#x22; highlighting that developers requested 8 CPUs per pod, but the actual usage is only 0.2 CPUs." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/over-provision-problem-cpu-usage-diagram.jpg" />
</Frame>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/financial-impact-over-provisioning-cpus.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=f6f671efb890ee5dfdaadf5965e2cbf3" alt="The image illustrates the financial impact of over-provisioning CPUs for developers, showing a request for 8 CPUs per pod but actual usage is only 0.2 CPUs, leading to 7.8 CPUs sitting unused." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/financial-impact-over-provisioning-cpus.jpg" />
</Frame>

Conversely, omitting limits can be catastrophic during traffic spikes (Black Friday or viral campaigns). A container that consumes free memory can exhaust node RAM. When node memory runs out, the kernel may invoke the OOM killer and Kubernetes marks Pods as OOMKilled—often at the worst possible moment.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/under-provisioning-kubernetes-resource-limits.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=d1bb7b7dd77f9fb9eb891cf1c1c52b23" alt="The image outlines the issue of under-provisioning in Kubernetes, showing step-by-step scenarios from having no resource limits to service instability during peak demand." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/under-provisioning-kubernetes-resource-limits.jpg" />
</Frame>

Without proper sizing you get three main operational problems:

* The scheduler makes poor placement decisions because requests are inaccurate.
* Pods are evicted unpredictably because limits/requests weren’t planned.
* Nodes become either starved (no capacity) or idle (wasted capacity), increasing risk or cost.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/cascading-problems-scheduler-evictions-nodes.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=289509ea39fa8fe73c0b89ed471cd128" alt="The image outlines three cascading problems without proper sizing: poor placement decisions by the scheduler, unpredictable pod evictions, and nodes being either starved or wasting resources." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/cascading-problems-scheduler-evictions-nodes.jpg" />
</Frame>

High‑level solution (conceptually simple; requires platform policy + observability):

1. Set sensible default CPU and memory requests.
2. Enforce a maximum resource limit per namespace with ResourceQuota.
3. Require memory limits for containers (memory overcommit is unforgiving).
4. Right-size using real usage metrics and iterate.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/cpu-memory-resource-limits-solution.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=57ae92dae0ecc8841d0970f9663c528c" alt="The image outlines a solution with four key components: sensible default CPU/memory requests, maximum resource limits per namespace, required memory limits, and right-sizing based on real usage." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/cpu-memory-resource-limits-solution.jpg" />
</Frame>

Important clarification: Kubernetes is not aware of your workload’s true needs unless you tell it. The scheduler relies entirely on resource requests for placement decisions. If you omit requests, the scheduler treats the container as claiming zero resources—potentially causing noisy-neighbor problems by scheduling Pods onto already highly utilized nodes. If you omit limits, a container can consume unbounded CPU (subject to CPU isolation behavior) or memory at runtime, affecting co-located workloads.

A runaway process (memory leak or unbounded allocation) without limits can crash other workloads on the node. When a Pod has neither requests nor limits defined, it receives the BestEffort QoS class and is the first to be evicted under resource pressure.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/kubernetes-resource-allocation-besteffort-qos.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=91997ac35d02c80392633f3d5fbfe205" alt="The image illustrates Kubernetes resource allocation, showing a default pod with &#x22;Requests&#x22; and &#x22;Limits&#x22; set to none, labeled as &#x22;BestEffort QoS – First to be evicted&#x22;, with nodes A and B displayed to the side." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/kubernetes-resource-allocation-besteffort-qos.jpg" />
</Frame>

Kubernetes needs explicit resource hints. Quick recap of why each field matters:

* requests: used by the scheduler to decide Pod placement (affects packing and cluster autoscaler behavior).
* limits: enforced by the kubelet via cgroups. For CPU, exceeding the limit results in throttling; for memory, exceeding the limit results in an OOM kill.
* defaults: Kubernetes does not apply automatic sensible defaults; use LimitRange to inject defaults per namespace and ResourceQuota/admission controls to enforce policies.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/kubernetes-resource-hints-issues-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=6f95e77efb9e1c845daad985c13af5b0" alt="The image explains why Kubernetes needs explicit resource hints, highlighting issues related to missing requests, limits, and default behavior. It notes that without requests, scheduling decisions are affected; without limits, a pod can crash a node; and default behavior assumes manual configuration." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Right-Sizing-101-Requests-Limits-QoS-and-Scheduling-Part-1/kubernetes-resource-hints-issues-diagram.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Best practice: enforce platform-wide defaults using `LimitRange` and control consumption with `ResourceQuota`. Combine these with observability to make data-driven adjustments rather than guessing.
</Callout>

Quick reference: QoS class rules

| QoS Class  | Rule / How assigned                                                                                                                           | Eviction priority   |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| Guaranteed | Every container sets both `requests` and `limits`, and for each resource the request equals the limit (e.g. `cpu: "200m"` and `cpu: "200m"`). | Last to be evicted  |
| Burstable  | At least one container sets a request or limit, but Pod is not Guaranteed (requests \< limits or only some containers define them).           | Middle priority     |
| BestEffort | No container specifies any requests or limits.                                                                                                | First to be evicted |

Example Pod spec with sensible `requests` and `limits`:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: app
    image: my-app:latest
    resources:
      requests:
        cpu: "200m"       # 0.2 vCPU
        memory: "256Mi"
      limits:
        cpu: "500m"       # 0.5 vCPU
        memory: "512Mi"
```

<Callout icon="warning" color="#FF6B6B">
  Memory limits are critical. Exceeding memory triggers OOM kills that can cascade into customer-visible failures—so require memory limits and monitor tail latencies and OOM events.
</Callout>

Recommended right-sizing workflow (practical, repeatable)

1. Platform defaults: use `LimitRange` to inject conservative `requests` and `limits` automatically; add `ResourceQuota` per namespace to cap consumption.
2. Deploy and collect telemetry: gather CPU, memory, latency, and error-rate metrics over representative traffic patterns (including peaks).
3. Analyze percentiles: review P50/P95/P99 and tail behavior. Set `requests` to cover steady-state usage (e.g., P50/P95 depending on tolerance) and `limits` to protect the node and bound peak behavior.
4. Automate and iterate: use Horizontal Pod Autoscaler (HPA) for CPU or custom metrics; for memory patterns consider Vertical Pod Autoscaler or automated recommendations. Reassess after major releases and periodically.

In short: this is primarily a governance + observability problem. Provide explicit, sensible resource requests and limits and enforce them at the platform level. Use telemetry to right-size iteratively so the scheduler can make good placement decisions and your cluster remains stable and cost-effective.

Links and references

* [Kubernetes: Configure resources for containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
* [Kubernetes QoS Classes](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
* [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)
* [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/55fffdb1-b6d8-4915-af0e-b66a18c6a3fb" />
</CardGroup>
