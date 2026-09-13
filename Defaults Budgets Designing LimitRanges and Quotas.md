> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Defaults Budgets Designing LimitRanges and Quotas

> Guidance for designing Kubernetes LimitRange and ResourceQuota defaults, tiered namespace quotas, and iterative monitoring to balance developer agility and cluster resource governance

This lesson expands on ResourceQuota and LimitRange concepts and answers the practical question: which values should you actually configure?

You will learn:

* Why one-size-fits-all quotas fail
* How to choose sensible defaults for common workloads
* How to design environment-specific quotas (namespace tiers)
* How to apply observed usage to refine limits and requests

Start with a common real-world failure.

A platform team at a logistics company applied a flat `ResourceQuota` of 2 CPUs to every namespace. Within 24 hours the data pipeline team could not deploy: their batch jobs required 8 CPUs per container. Meanwhile, a lightweight API team used only \~200m CPU of their 2-CPU allocation — most capacity was wasted. A single quota for all namespaces helped no one.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/data-pipeline-cpu-allocation-issue.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=51c53c612ddb05bc1fe306e7d0c8e970" alt="The image illustrates a problem where a data pipeline team needs 8 CPUs for deployment but is limited to a 2 CPU allocation per namespace, while a lightweight API team is also shown with 2 CPU allocation." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/data-pipeline-cpu-allocation-issue.jpg" />
</Frame>

Four questions every platform team asks

1. What should the default CPU request be?
   * Answer: it depends on the workload mix. A lightweight web API might request \~500m, a medium batch job \~2 CPUs, and an ML training pod may require 8+ CPUs. There is no single universal default.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/cpu-memory-quotas-namespace-discussion.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=0640c9f880ffca348a7ef6397de36b5b" alt="The image presents a discussion on default CPU and memory quotas for namespaces, detailing varying CPU needs for components like API Gateway, Data Pipeline, and ML Training, while questioning universal defaults." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/cpu-memory-quotas-namespace-discussion.jpg" />
</Frame>

2. How much memory quota per namespace is fair?
   * Answer: it depends on how many services a team runs, their workload intensity, and whether the namespace is for exploratory dev or customer-facing production.

3. How many Pods should we allow per namespace?
   * Answer: it depends on deployment patterns. A team with a single service and three replicas needs few Pods. A team running 20 microservices with autoscaling may need hundreds.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/resource-allocation-namespaces-comparison.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=b5e515fedaee7c56a587e311e0ea6293" alt="The image discusses issues related to resource allocation in namespaces, including CPU requests, memory quotas, and pod limits, with a comparison between single service and microservices teams." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/resource-allocation-namespaces-comparison.jpg" />
</Frame>

4. What happens if defaults are too restrictive?
   * Answer: low defaults increase developer friction (many quota escalation requests). High defaults defeat governance and waste cluster capacity.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/cpu-memory-requests-limits-namespace-issues.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=4388908b2afa77c6005985da7059c7cb" alt="The image presents a series of questions about CPU requests, memory quotas, and pod limits in namespaces, highlighting concerns about defaults being too restrictive and causing friction. An accompanying bar suggests that defaults set too low can lead to friction." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/cpu-memory-requests-limits-namespace-issues.jpg" />
</Frame>

You need a methodology, not a magic number. The rest of this lesson provides one.

Recognize workload variability

Workloads differ dramatically: web APIs, batch data pipelines, and ML training jobs each have distinct resource patterns. Most organizations host a mix of lightweight services, medium-weight batch jobs, and heavy compute workloads. Defaults designed for one category will either throttle or waste resources for others.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/workload-resource-profiles-mix-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=c7c8a63b5e2eb8127dc4728bad6c3d36" alt="The image illustrates that different workloads, such as Web API, Data Pipeline, and ML Training, have varied resource profiles. It emphasizes that organizations typically manage a mix of these workloads." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/workload-resource-profiles-mix-diagram.jpg" />
</Frame>

Use environment intent to guide quotas

Namespaces represent intent: dev, staging, production. Apply different policies accordingly:

* Dev namespaces: low quotas for experimentation, quick feedback loops.
* Staging namespaces: moderate quotas to run production-like tests.
* Production namespaces: higher quotas and stricter controls for availability and performance.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/namespaces-development-staging-production-infographic.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=eaa896c7d90052a81dd411b9a5eb44bb" alt="The image is an infographic explaining that namespaces serve different purposes in development, staging, and production environments, each with distinct quota and limit characteristics." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/namespaces-development-staging-production-infographic.jpg" />
</Frame>

Two core design principles

* Set LimitRange defaults to match common workload types and require explicit overrides for exceptional cases.
* Create namespace tiers (small, medium, large) and apply tier-specific ResourceQuotas.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/resource-management-design-principles.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=ba1bc2ce0fb5d16763c1779feae5cfd9" alt="The image illustrates design principles for resource management, suggesting setting LimitRange defaults and creating namespace tiers with different ResourceQuotas." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/resource-management-design-principles.jpg" />
</Frame>

Practical starting points (sensible, not sacred)

Use pragmatic defaults per-container for small, lightweight services and provide tiered namespace quotas for teams. These are starting points to be validated and adjusted.

* LimitRange defaults (per-container) for lightweight services:
  * default request: cpu: 100m, memory: 128Mi
  * default limit: cpu: 500m, memory: 512Mi
  * Multiplier guidance: limit/request commonly 2x–5x. CPU multipliers can be higher (3x–5x); memory multipliers should be tighter (2x–3x) to reduce OOM risk.
* ResourceQuota per namespace: select values based on expected service count and replica targets. Pairing ResourceQuota with LimitRange reduces Pod creation rejections by ensuring defaults are injected before quota checks.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/kubernetes-limitrange-resourcequota-defaults.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=3d8e3c1657e4780c77ecd3352ea289f8" alt="The image shows configuration defaults for &#x22;LimitRange&#x22; and &#x22;ResourceQuota&#x22; in Kubernetes, detailing CPU and memory allocations. It provides sensible starting points for resource management." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/kubernetes-limitrange-resourcequota-defaults.jpg" />
</Frame>

Adjust these starting points for team size and workload types. Monitor actual usage for 2–4 weeks and iterate; never set values once and forget them.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/kubernetes-resource-configuration-starting-points.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=7ae04107554b4ff5e00049a370cc3b5d" alt="The image shows recommended starting points for Kubernetes resource configurations, including LimitRange defaults and ResourceQuota per namespace. It provides initial values for CPU and memory requests and limits." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/kubernetes-resource-configuration-starting-points.jpg" />
</Frame>

LimitRange — a walk-through

LimitRange fields can be confusing. Key behaviors:

* `spec.limits[].default`: injected as container limits when a container omits explicit limits (caps burst).
* `spec.limits[].defaultRequest`: injected as container requests when omitted (scheduler reserves this).
* `spec.limits[].max` / `min`: optional hard bounds. If `max.cpu` is "4", a container limit >4 will be rejected.
* `type: "Container"` applies defaults and bounds per container. A Pod with three containers receives defaults three times (one per container). Use `type: "Pod"` to apply at Pod level instead.

Important: LimitRange only injects defaults when containers omit requests/limits. If developers specify explicit values, LimitRange will not override them, only validate against min/max.

YAML example:

```yaml theme={null}
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "4"
        memory: 4Gi
```

ResourceQuota — a walk-through

This ResourceQuota complements the LimitRange above.

* `requests.*` fields cap the sum of container requests in the namespace (the scheduler budget).
* `limits.*` fields cap the sum of container limits in the namespace (runtime/burst budget).
* Typical pattern: `limits.*` > `requests.*` to permit bursting (for example, `limits.cpu: "8"` vs `requests.cpu: "4"`).
* `pods` sets a hard cap on the number of Pods (prevents accidental sprawl).

You can also quota other resources (`services`, `configmaps`, `persistentvolumeclaims`, etc.) to avoid object sprawl.

Behavioral note: Quota admission rejects Pod creations that would exceed tracked usage. LimitRange injection happens before quota validation, so using both together lowers the chance of unexpected rejections.

YAML example:

```yaml theme={null}
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
```

Namespace tiers — put the methodology into practice

Create tier templates and apply them consistently. Review assignments quarterly.

| Tier               | Typical intent                              | Example ResourceQuota guidance                                                                                  |
| ------------------ | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Small (Dev)        | Experimentation, feature work               | `requests.cpu: "1"`, `limits.cpu: "2"`, `requests.memory: "2Gi"`, `limits.memory: "4Gi"`, `pods: "10"`          |
| Medium (Staging)   | Integration/QA, production-like tests       | `requests.cpu: "4"`, `limits.cpu: "8"`, `requests.memory: "8Gi"`, `limits.memory: "16Gi"`, `pods: "50"`         |
| Large (Production) | Production services, autoscaling and spikes | `requests.cpu: "8+"`, `limits.cpu: "16+"`, `requests.memory: "16Gi+"`, `limits.memory: "32Gi+"`, `pods: "100+"` |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/namespace-tiers-resource-allocations-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=afcd90e2ea48e71d84a3de8d4516e940" alt="The image illustrates namespace tiers with different resource allocations and purposes for small (Dev), medium (Staging), and large (Production) environments, detailing CPU, memory, pod count, and storage requirements." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Defaults-Budgets-Designing-LimitRanges-and-Quotas/namespace-tiers-resource-allocations-diagram.jpg" />
</Frame>

Monitoring and iteration

* Instrument cluster and namespace-level metrics (CPU, memory, pod counts, OOM events).
* Observe for 2–4 weeks across representative workloads.
* Adjust LimitRange defaults, min/max, and ResourceQuota hard limits based on real usage and growth trends.
* Communicate changes and provide a clear escalation path for teams that need temporary or permanent quota increases.

Resources and further reading

* Kubernetes LimitRange: [https://kubernetes.io/docs/concepts/policy/limit-range/](https://kubernetes.io/docs/concepts/policy/limit-range/)
* Kubernetes ResourceQuota: [https://kubernetes.io/docs/concepts/policy/resource-quotas/](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
* Monitoring practices: use Prometheus/Grafana or cloud provider metrics to capture long-term trends

Five key takeaways

* Start with sensible defaults and iterate based on observed usage.
* Use namespace tiers to reflect environment intent (dev, staging, prod).
* Always pair ResourceQuota with LimitRange: quotas enforce totals, LimitRange provides defaults and validation.
* Monitor, review, and evolve quotas — governance is not "set-and-forget."
* Good governance enables teams while protecting shared cluster resources and preventing accidental or malicious exhaustion.

<Callout icon="lightbulb" color="#1CB2FE">
  Design for variability: different workloads and environments require different default strategies. Start with sensible defaults, monitor real usage, and evolve quotas and limits to match observed needs.
</Callout>

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/cdcb2f9a-9116-40d8-b148-0bf234f5438e" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/c6feebc9-1b18-41f9-bc1d-65d36f2b27c3" />
</CardGroup>
