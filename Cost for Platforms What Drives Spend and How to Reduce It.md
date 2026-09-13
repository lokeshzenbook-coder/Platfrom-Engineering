> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Cost for Platforms What Drives Spend and How to Reduce It

> Explains Kubernetes cloud cost drivers, using OpenCost for visibility, and practical strategies to attribute, optimize, and reduce platform spending

This lesson explains the financial side of running Kubernetes: what causes cloud spend, how to make that spend visible, and how to reduce it systematically. By the end you'll have a practical framework and tools to answer: Where is our cloud spend going? Who is spending it? And what can we do to reduce it?

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/kubernetes-cost-management-learning-objectives.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=3209fedcf3a292166241824f92b1fc99" alt="The image outlines five learning objectives related to Kubernetes cost management, including identifying cost drivers, using OpenCost, analyzing spending, implementing cost optimization, and understanding cost management." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/kubernetes-cost-management-learning-objectives.jpg" />
</Frame>

## What actually drives Kubernetes costs?

Consider this real example: a mid-sized SaaS provider saw its monthly Kubernetes cloud bill jump from $30,000 to $120,000 in a year. The cloud invoice showed a large EC2 line item, but the platform team couldn't map those dollars to namespaces, teams, or workloads — so they couldn't tell whether the spend was justified or wasteful.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cloud-bill-surprise-unplanned-scaling-k8s.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=c6614851b098c27680692e644279563c" alt="The image highlights a problem labeled &#x22;Cloud Bill Surprise,&#x22; with subpoints on &#x22;Unplanned Scaling&#x22; and &#x22;K8s Infrastructure.&#x22;" width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cloud-bill-surprise-unplanned-scaling-k8s.jpg" />
</Frame>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/resource-consumption-engineering-teams-statement.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=d1fde3a358cb3c66a27692eb8817fa52" alt="The image presents a problem statement regarding resource consumption by engineering teams, highlighting questions about which teams and workloads were responsible and whether the usage was justified or wasteful, with a total EC2 cost of $84,000." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/resource-consumption-engineering-teams-statement.jpg" />
</Frame>

The operational solution is to introduce namespace-level cost allocation so teams and finance can map cloud dollars to platform constructs. With that visibility you can systematically eliminate unused capacity and right-size resources where it matters most.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/namespace-cost-allocation-optimization.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=a1e75de37a7b9463404e0fb8c96f40b7" alt="The image outlines a solution involving namespace-level cost allocation to identify spending, and using insights to optimize resources and eliminate unused capacity." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/namespace-cost-allocation-optimization.jpg" />
</Frame>

## Major cost categories (and where to focus)

Kubernetes cloud spend generally falls into three primary categories. Prioritize efforts where dollars are concentrated to get the highest ROI.

| Category        | Typical share of spend | What to focus on                                                                                                    |
| --------------- | ---------------------: | ------------------------------------------------------------------------------------------------------------------- |
| Compute (nodes) |                 60–70% | Pay-for-provisioned capacity. Address over-provisioned pods, right-size requests/limits, and use autoscaling.       |
| Storage         |                 15–25% | Persistent volumes, snapshots, and storage class choices (SSD vs HDD). Clean up stale snapshots and unused volumes. |
| Network         |                 10–20% | Egress and cross-region transfer can be costly. Optimize data movement patterns and caching.                        |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/kubernetes-costs-compute-storage-network.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=3928aed43652199cfcdf26d6303e0d5e" alt="The image outlines the factors driving Kubernetes costs, categorized into Compute (60-70% of total spend), Storage (15-25% of total spend), and Network (10-20% of total spend), detailing specific cost contributors for each category." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/kubernetes-costs-compute-storage-network.jpg" />
</Frame>

The single largest cost driver is provisioned capacity you do not actually use — the gap between what you pay for (node capacity) and what workloads consume. In many organizations that utilization gap represents 40–60% waste. Closing that gap is the highest-leverage optimization.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/kubernetes-cost-drivers-utilization-gap.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=45a6b1b8da777e6b1f10992d275e759d" alt="The image illustrates the cost drivers of Kubernetes, highlighting the gap between paid node capacity and actual workload consumption, with an emphasis on reducing the utilization gap to optimize costs." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/kubernetes-cost-drivers-utilization-gap.jpg" />
</Frame>

## The visibility gap: cloud bill vs platform usage

The cloud bill shows dollars by account, service, and line item — not by namespace, team, or workload. That mismatch is the visibility gap: finance sees charges, platform engineers see Kubernetes constructs, and neither can easily attribute dollars to business units.

Typical questions to answer:

| Question                               | Example                                            |
| -------------------------------------- | -------------------------------------------------- |
| How much is a team costing us?         | Payments team: `team=payments`                     |
| Which namespace wastes the most?       | Look for low-efficiency namespaces in cost reports |
| Over-provisioned or under-provisioned? | Compare requested resources vs actual usage        |

Bridging this gap requires a tool that understands cloud pricing and Kubernetes resource models. OpenCost is one such open-source solution.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/visibility-problem-cloud-billing-kubernetes.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=04d428509d86e6c5040bbbdca279add6" alt="The image illustrates a &#x22;visibility problem&#x22; between cloud billing at the infrastructure level and Kubernetes usage at the platform level, highlighting OpenCost as a bridging solution." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/visibility-problem-cloud-billing-kubernetes.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  [OpenCost](https://opencost.io) is an open-source, vendor-neutral project (CNCF incubating) that maps Kubernetes usage to cloud pricing so you can attribute dollars to namespaces, labels, and pods.
</Callout>

## OpenCost overview: how it works and what it provides

OpenCost runs inside Kubernetes as a set of pods. It scrapes usage from the Kubernetes Metrics API or Prometheus, overlays cloud pricing, and produces dollar-based allocations.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/opencost-kubernetes-cost-visibility-slide.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=20bbbaac5c8427eb1d975ce410d33ccc" alt="The image is a slide titled &#x22;OpenCost: Kubernetes Cost Visibility,&#x22; highlighting OpenCost as an open-source, vendor-neutral, CNCF incubating project." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/opencost-kubernetes-cost-visibility-slide.jpg" />
</Frame>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/opencost-kubernetes-cost-visibility-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=3bc22ab019a3f148e89a63a8c2a5348e" alt="The image illustrates &#x22;OpenCost: Kubernetes Cost Visibility,&#x22; showing a Kubernetes Cluster with an OpenCost Pod, alongside Metrics API and Pricing data components detailing per-hour node costs, per-GB storage costs, and data transfer costs." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/opencost-kubernetes-cost-visibility-diagram.jpg" />
</Frame>

What OpenCost gives you (three core capabilities):

1. Cost allocation by namespace or label
   * Attributes node, storage, and network costs to namespaces/labels according to pod consumption. Example: if a node costs $1,000/month and a team's pods consume 10% of that node, the team is allocated $100/month.

2. Resource-efficiency metrics
   * Shows CPU/memory utilization vs requests so you can surface over-provisioned (waste) and under-provisioned (risk) workloads.

3. Real-time and historical data
   * Time-series cost trends and anomaly detection (e.g., sudden spikes or month-over-month growth).

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/opencost-kubernetes-cost-visibility-dashboard.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=50f49f11bb20470745218d149a5dbbf8" alt="The image shows a dashboard of OpenCost displaying Kubernetes cost visibility, with a cost allocation chart for the last 7 days by namespace, indicating zero cost for various resources." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/opencost-kubernetes-cost-visibility-dashboard.jpg" />
</Frame>

## How OpenCost calculates costs

OpenCost follows a simple three-step model to turn metrics into dollars:

1. Get cloud pricing
   * Derive per-CPU-hour and per-GB-hour node costs (and storage/network rates) using cloud provider pricing.

2. Measure pod resource usage
   * Collect actual CPU/memory usage. OpenCost charges the higher of the pod's request or its actual usage so both waste and risk are visible.

3. Aggregate by namespace or label
   * Sum pod-level costs into namespace/label totals (for example, all pods labeled `team=payments`).

The result is a concrete allocation such as: the Payments namespace costs \$2,400/month — which closes the visibility gap and enables finance and engineering to act.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cost-allocation-namespace-team-payments.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=fc42792d021bc7ca40fb62829fac1011" alt="The image illustrates cost allocation by namespace, highlighting a monthly expense of $2,400 for the &#x22;team-payments&#x22; namespace, with roles for Finance in allocating cloud costs and Engineering Leadership in identifying costly workloads." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cost-allocation-namespace-team-payments.jpg" />
</Frame>

## Efficiency metric and the target zone

OpenCost computes efficiency as actual usage divided by requests. Interpretations:

* Over-provisioned: Low efficiency (e.g., 12.5%). Large waste — you're paying for far more than you need.
* Right-sized: Efficiency \~60–80% — the recommended target zone balancing cost and headroom for spikes.
* Under-provisioned: Efficiency >100% (e.g., 240%) — the pod is using more than requested and may cause throttling or OOM kills.

Aim for the 60–80% band. Below 50% indicates significant waste; above 90% suggests little headroom and higher operational risk. The right-sizing workflow looks like: profile usage, set requests near the 95th percentile, and target the 60–80% efficiency band.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cpu-usage-efficiency-comparison-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=69f5360d7642d8f822eae0512a293d9c" alt="The image compares three scenarios of CPU usage efficiency: over-provisioned with low efficiency, right-sized with balanced efficiency, and under-provisioned with high efficiency but potential risks." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cpu-usage-efficiency-comparison-diagram.jpg" />
</Frame>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cost-efficiency-spectrum-opencost-zones.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=2836b1d49d27ee8a8c7565febf17e3ba" alt="The image is a cost-efficiency spectrum comparing requested versus used efficiency, categorized into zones: &#x22;Wasted Money&#x22; (below 50%), &#x22;Target Zone&#x22; (60-80%), and &#x22;High Risk&#x22; (90%+), with the tool OpenCost used to show namespace placement on this spectrum." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cost-efficiency-spectrum-opencost-zones.jpg" />
</Frame>

<Callout icon="warning" color="#FF6B6B">
  Be careful when right-sizing: aggressive reductions can cause production outages. Always validate changes in a staging or canary environment and combine metrics with business context.
</Callout>

## How to reduce costs — an operational roadmap

Cost reduction is continuous. Organize efforts across short, medium, and long horizons and prioritize by dollar impact (start with compute/node costs).

| Horizon                  | Actions                                                                                                                                    | Typical impact                               |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------- |
| Quick wins (this week)   | Right-size requests from observed usage; kill idle dev environments; move non-critical batch jobs to HDD                                   | 20–40% reductions are common                 |
| Medium-term (this month) | Use spot/preemptible instances for tolerant workloads; tune HPA/VPA and cluster autoscaler                                                 | Significant recurring savings                |
| Long-term (quarter-plus) | Build cost-accountability (show teams their costs, set budget alerts); implement quotas and guardrails; embed efficiency into architecture | Cultural and architectural savings over time |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cost-reduction-strategies-optimization-tactics.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=8b9d497b1c6001682482f538229ceaae" alt="The image outlines cost reduction strategies categorized into quick wins, medium-term, and long-term approaches, each with specific tactics for optimizing resources and saving costs." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Cost-for-Platforms-What-Drives-Spend-and-How-to-Reduce-It/cost-reduction-strategies-optimization-tactics.jpg" />
</Frame>

Prioritize your top-dollar namespaces first (e.g., the ten costliest). Often, the biggest ROI comes from right-sizing those namespaces and removing idle nodes.

## Combine visibility with governance

Visibility plus governance creates sustainable cost control:

* Visibility (OpenCost): maps dollars to teams and workloads.
* Governance: quotas, limits, and automated policies prevent runaway consumption.
* Culture: show teams their costs and give them the tools to self-optimize.

When combined, these elements enable continuous cost optimization instead of reactive firefighting.

This concludes the module covering architecture, resources, multi-tenancy, governance, storage, networking, and cost.

## Links and references

* OpenCost: [https://opencost.io](https://opencost.io)
* Amazon EC2 docs: [https://learn.kodekloud.com/user/courses/amazon-elastic-compute-cloud-ec2](https://learn.kodekloud.com/user/courses/amazon-elastic-compute-cloud-ec2)
* Kubernetes docs: [https://kubernetes.io/docs/](https://kubernetes.io/docs/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/36ec1768-6ce1-430d-962a-e8359f9c6c1d" />
</CardGroup>
