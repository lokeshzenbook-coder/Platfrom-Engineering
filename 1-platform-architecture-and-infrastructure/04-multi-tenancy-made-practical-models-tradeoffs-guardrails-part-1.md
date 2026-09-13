> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Multi Tenancy Made Practical Models Tradeoffs Guardrails Part 1

> Guidance on multi-tenancy tradeoffs and practical guardrails for Kubernetes clusters including models, isolation mechanisms, and minimum controls like RBAC ResourceQuota and NetworkPolicies

Assuming a platform blueprint and appropriately sized resources, a critical question follows: what happens when multiple teams share the same infrastructure?

In this lesson you will learn how to compare multi-tenancy models, choose an appropriate model based on trust boundaries, implement namespace-level isolation using a guardrail stack, and understand what Kubernetes does and does not provide by default.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/learning-objectives-multi-tenancy-kubernetes.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=0d76c43e6b31e0f19749e072f856cb52" alt="The image lists learning objectives related to multi-tenancy models, trust boundaries, namespace-level isolation, and Kubernetes behavior, organized in a colorful, vertical numbered format." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/learning-objectives-multi-tenancy-kubernetes.jpg" />
</Frame>

## Why namespaces alone are not enough

Consider an example where four teams share one cluster and each team has its own namespace. It’s a common assumption that namespaces act as security boundaries—this is incorrect. By default, namespaces do not enforce security or resource boundaries between teams. Without guardrails, shared clusters are vulnerable to several common failure modes.

### Four common failure modes in a shared cluster

Failure mode 1 — secrets exposure:
If a developer runs the following command without RBAC restrictions, they may list secrets across namespaces:

```bash theme={null}
kubectl get secrets --all-namespaces
```

This can expose database credentials, API keys, and TLS private keys for every team. Without properly configured RBAC, there is no secret isolation between namespaces.

<Callout icon="warning" color="#FF6B6B">
  Secrets and credential leakage is one of the highest-risk outcomes in shared clusters. Ensure RBAC and least-privilege access are configured before allowing broad `kubectl` access.
</Callout>

Failure mode 2 — resource exhaustion:
If no ResourceQuota is configured, a single team can scale a workload to hundreds of replicas and consume cluster CPU and memory. Other teams’ pods may remain in Pending due to lack of capacity.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/failure-modes-secrets-resource-exhaustion.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=d4efa779bc4172ae5154bed6e41dce7d" alt="The image lists four failure modes: Secrets Exposure, Resource Exhaustion, Accidental Deletion, and Network Segmentation, with highlighted implications of Resource Exhaustion due to lack of ResourceQuota configuration." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/failure-modes-secrets-resource-exhaustion.jpg" />
</Frame>

Impact: cluster instability, unfair resource usage, and operational disruption.

Failure mode 3 — accidental deletion:
A developer intends to delete a deployment in Team A's namespace but runs the kubectl command against the wrong namespace. For example:

```bash theme={null}
# Dangerous — will delete in the current namespace (could be the wrong one)
kubectl delete deployment my-deployment

# Safer — specify the namespace explicitly
kubectl delete deployment my-deployment -n team-a
```

Without RBAC restrictions and careful workflows, accidental or mis-scoped commands can delete other teams’ resources.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/four-failure-modes-accidental-deletion.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=9552a5a8c6508e3fd90db247684ea947" alt="The image lists &#x22;Four Failure Modes&#x22; with an emphasis on &#x22;Failure Mode 3: Accidental Deletion,&#x22; showing a person accidentally deleting a program and highlighting the absence of RBAC (Role-Based Access Control)." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/four-failure-modes-accidental-deletion.jpg" />
</Frame>

Failure mode 4 — network segmentation:
By default, pods can communicate with any other pod in the cluster via cluster IPs. There is no default network-level firewall between namespaces. If a pod is compromised, an attacker can reach databases, admin APIs, and other teams’ workloads—enabling lateral movement across the cluster.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/failure-modes-network-errors-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=4eb24ea23337552aa8775def9e995c9b" alt="The image lists four failure modes: secrets exposure, resource exhaustion, accidental deletion, and network segmentation, with a diagram illustrating potential network errors and attacker access in a shared cluster." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/failure-modes-network-errors-diagram.jpg" />
</Frame>

In short: namespaces are not security boundaries. Kubernetes does not protect teams from each other by default—security, stability, and fairness require explicit configuration.

<Callout icon="lightbulb" color="#1CB2FE">
  Start with these minimum guardrails for a multi-team cluster: RBAC (least privilege), ResourceQuotas, and NetworkPolicies. These three reduce the most common cross-tenant risks.
</Callout>

## Why Kubernetes is not multi-tenant out of the box

* Network: by default all pods can reach all other pods using cluster IPs—no network segmentation.
* Resources: no per-namespace resource limits exist by default; a namespace can consume the entire cluster.
* RBAC: role-based access control must be explicitly configured. What a user can do depends entirely on your Role and ClusterRole bindings; some managed services apply broad defaults.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/kubernetes-multi-tenancy-issues-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=c40170bb33061eadffaea87d28460447" alt="The image outlines the issues with default multi-tenancy in Kubernetes, highlighting communication, resource limit, and RBAC configuration concerns. It mentions how pods can communicate freely, lack of resource limits, and the necessity for explicit RBAC configuration." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/kubernetes-multi-tenancy-issues-diagram.jpg" />
</Frame>

## Why organizations still share clusters

The primary reason organizations share clusters is cost efficiency. Running many clusters multiplies control plane, monitoring, upgrade, and operational overhead.

| Approach         | Isolation                  | Pros                                                      | Cons                                                                      |
| ---------------- | -------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------- |
| Cluster per team | Strong                     | Best isolation; minimal cross-tenant blast radius         | High cost, many control planes to manage, complex upgrades and monitoring |
| Shared cluster   | Moderate (with guardrails) | Cost-efficient; single control plane and monitoring stack | Requires discipline and tooling; misconfiguration can affect many teams   |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/cluster-comparison-team-vs-shared.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=2b772e970a87f4a349653e5011313c74" alt="The image compares two types of cluster setups: &#x22;Cluster per Team,&#x22; which offers strong isolation but is expensive and operationally heavy, and &#x22;Shared Cluster,&#x22; which is cost-efficient and simpler but requires discipline and guardrails." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/cluster-comparison-team-vs-shared.jpg" />
</Frame>

This is a fundamental trade-off: isolation costs money; sharing requires discipline.

## Two main multi-tenancy models

Choose a model based on trust boundaries and compliance requirements. The two primary approaches are soft multi-tenancy and hard multi-tenancy.

Soft multi-tenancy

* Use case: internal teams within the same organization (semi-trusted tenants).
* Isolation mechanisms: namespaces, RBAC, NetworkPolicies, ResourceQuotas (Kubernetes-native objects).
* Advantages: simpler, cost-effective, single cluster, minimal extra tooling.
* Downsides: tenants share the Linux kernel and control plane—kernel or control plane vulnerabilities can affect all tenants.
* Typical recommendation: start here for internal teams where risk is manageable.

Hard multi-tenancy

* Use case: untrusted tenants (external customers), or strict compliance (healthcare, finance).
* Isolation mechanisms: separate clusters or virtual clusters (for example, `vcluster`).
* Advantages: stronger boundaries and contained blast radius.
* Downsides: higher cost and greater operational complexity.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/soft-hard-multi-tenancy-comparison.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=53d9227bd6277155daa90cd24c199f5f" alt="The image compares &#x22;Soft Multi-Tenancy&#x22; and &#x22;Hard Multi-Tenancy&#x22; models, detailing their isolation methods, use cases, trust levels, advantages, and disadvantages." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/soft-hard-multi-tenancy-comparison.jpg" />
</Frame>

Decision rule: if tenants are internal teams in the same company, start with soft multi-tenancy. If tenants are external customers or you must meet strict compliance, adopt hard multi-tenancy.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/multi-tenancy-comparison-soft-hard.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=7c5cd6b4b160abedd330d770d3386975" alt="The image compares soft and hard multi-tenancy models, highlighting their isolation methods, use cases, trust levels, and pros and cons. Soft multi-tenancy is used within organizations, while hard multi-tenancy offers more security for external customers." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/multi-tenancy-comparison-soft-hard.jpg" />
</Frame>

Most organizations will adopt soft multi-tenancy for internal teams. The remainder of this lesson focuses on practical guardrails and isolation mechanisms for that model.

## Isolation mechanisms (soft multi-tenancy layers)

Compare options by isolation strength, cost, and operational complexity. Start with the simplest model that satisfies your security and compliance requirements and add layers only as needed.

| Mechanism                           | Isolation level | Cost & Complexity                                                                                  | Typical use case                                                               |
| ----------------------------------- | --------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Namespace-only                      | Low             | Minimal cost; low complexity                                                                       | Small, trusted dev teams where mistakes have low impact                        |
| Namespace + RBAC + ResourceQuota    | Medium          | Kubernetes-native; moderate implementation effort                                                  | Recommended minimum for multi-team clusters (even non-production)              |
| Add NetworkPolicies                 | Medium–High     | Low operational cost but requires a CNI that supports NetworkPolicies (see `https://www.cni.dev/`) | Production clusters where pods should be segmented by tenant                   |
| Virtual clusters (e.g., `vcluster`) | High            | Medium–high; gives per-tenant API separation while sharing nodes                                   | Teams needing cluster-level resources or scoped admin without affecting others |
| Separate clusters                   | Highest         | Highest cost and operational overhead                                                              | External customers, strict compliance, or unacceptable shared blast radius     |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/isolation-mechanisms-comparison-chart.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=d54a1bb3df1e6c3b4296d49c7722988f" alt="The image is a comparison chart of isolation mechanisms, detailing different approaches and their respective levels of isolation, cost, complexity, and suitability for various environments." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Multi-Tenancy-Made-Practical-Models-Tradeoffs-Guardrails-Part-1/isolation-mechanisms-comparison-chart.jpg" />
</Frame>

Principle: start with the simplest model that satisfies your security and compliance requirements. Do not over-engineer isolation for a small, trusted development team—add controls as your risk profile grows.

## Next steps and references

* Kubernetes RBAC docs: [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* NetworkPolicy / CNI: [https://www.cni.dev/](https://www.cni.dev/)
* Virtual clusters: `vcluster` — [https://github.com/loft-sh/vcluster](https://github.com/loft-sh/vcluster)

Use the guidance in this lesson to choose a tenancy model, implement the minimum guardrails (RBAC, ResourceQuota, NetworkPolicies), and iterate from there.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/473c5c5a-97c8-4094-8c6f-1c22e1dbfd25" />
</CardGroup>
