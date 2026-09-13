> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Service Mesh Integration for Progressive Delivery

> Explains using service mesh and Istio Envoy sidecars with Argo Rollouts to enable precise L7 canary routing, weighted traffic control, mirroring, and automated progressive delivery.

Canary deployments route a small percentage of production traffic to a new release while monitoring metrics. Kubernetes Services alone cannot perform precise, HTTP-level weighted routing, so a service mesh (e.g., Istio) is commonly used to enable L7 (layer-seven) traffic control like weighted routing, header-based routing, traffic mirroring, and fault injection. In this lesson we'll:

* Explain why Kubernetes Services are insufficient for precise canary percentages.
* Show how sidecar proxies enable L7 routing.
* Configure Istio VirtualService and DestinationRule for canary routing.
* Demonstrate how Argo Rollouts automates canary weight changes and analysis.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/kubernetes-service-mesh-istio-objectives.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=61bc74c5796312748dd8f2c462c05f2e" alt="The image lists four learning objectives related to Kubernetes, service mesh, Istio, and Argo Rollouts. It includes understanding routing, traffic control, and integration features." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/kubernetes-service-mesh-istio-objectives.jpg" />
</Frame>

## Scenario: Why a 5% Canary Failed

A platform team attempted a 5% canary using only a Kubernetes Service. Their cluster had 19 stable pods (v1) and 1 canary pod (v2). They expected roughly 5% of requests to go to v2 because the canary made up 1/20 endpoints. However, Kubernetes Services only load-balance at layer four (L4), so any percentage is only an approximation based on endpoint count—not a precise, HTTP-aware distribution.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/kubernetes-load-balancing-stable-canary.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=4ae2452637d2657dd9adcf1783ca5850" alt="The image illustrates that Kubernetes services, using round-robin load balancing, cannot split traffic effectively between stable and canary versions, with 95% going to stable and 5% to canary." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/kubernetes-load-balancing-stable-canary.jpg" />
</Frame>

## Why the L4-only approach fails

* No precise traffic control: Services balance connections across endpoints; percentages can only be approximated by changing replica counts.
* No user-based routing: Services cannot target specific users (for example, based on cookies or headers).
* No request-level mirroring or observability: You cannot copy HTTP requests to a canary without traffic-aware proxies.
* Limited routing logic: Services cannot inspect HTTP paths, headers, or implement weighted percentages at L7.

Table: Limitations of Kubernetes Service for Canary Routing

| Limitation                     | Impact                                                           |
| ------------------------------ | ---------------------------------------------------------------- |
| L4-only routing                | Cannot perform HTTP-level weighting or path/header-based routing |
| Replica-count dependence       | Percentages must be approximated by changing pod counts          |
| No traffic mirroring           | Cannot send a copy of live requests to canary for testing        |
| No request-level observability | Harder to analyze canary behavior at the request level           |

A Kubernetes Service will route to pods roughly equally: e.g., two v1 pods and one v2 pod -> v1 ≈ 66%, v2 ≈ 33%. You cannot reliably enforce 95%/5% with a Service alone.

## How a Service Mesh Solves This

Istio (as an example) injects an Envoy sidecar proxy into each pod. Envoy intercepts all inbound/outbound traffic and applies HTTP-aware routing configured by Istiod (the control plane). Because Envoy operates at L7 it can:

* Apply exact weighted routing (e.g., 95% stable, 5% canary).
* Route based on headers, cookies, or query params.
* Mirror traffic to canary for testing without impacting users.
* Inject faults (latency, errors) for resilience testing.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/service-mesh-traffic-control-envoy.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=e84dc627b132a2a4dafcc5f61f0b5faf" alt="The image illustrates a service mesh traffic control setup with a control plane (Istiod) configuring proxies for two pods, each using Envoy. It explains that Envoy intercepts all traffic and applies routing rules." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/service-mesh-traffic-control-envoy.jpg" />
</Frame>

Service mesh features commonly used for progressive delivery:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/service-mesh-traffic-control-capabilities.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=fec1be41ad04925697e21b23e9363f4b" alt="The image outlines four service mesh traffic control capabilities: weighted routing, header-based routing, traffic mirroring, and fault injection, each with a brief description." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/service-mesh-traffic-control-capabilities.jpg" />
</Frame>

* Weighted routing: precise percentage-based routing independent of replica counts.
* Header-based routing: target specific users or groups via cookies/headers.
* Traffic mirroring: duplicate requests to canary for testing/perf analysis.
* Fault injection: simulate failures to validate resilience.

These capabilities are declared as YAML resources (e.g., Istio VirtualService and DestinationRule), which can be stored in Git for GitOps-style deployments.

## Istio objects used for Canary Routing

Three Istio resources typically participate in a canary:

| Resource             | Purpose                                              | Notes                                                                                                   |
| -------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `VirtualService`     | Defines HTTP routing rules and weight splits         | Contains `http` route list with `destination` entries and `weight` values                               |
| `DestinationRule`    | Defines subsets (logical names mapped to pod labels) | Subset names are referenced in `VirtualService`                                                         |
| Kubernetes `Service` | DNS/service discovery for the host name              | Istio routes traffic for a service name; the service still exists but L7 routing is handled by the mesh |

Key insight: the mesh can send exactly 5% of traffic to a single canary pod while the remaining 95% goes to stable pods, because Envoy applies weights at request time.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/istio-virtualservice-canary-deployments-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=08451f5f555a08ef8fcaff9da5800d2e" alt="The image is a diagram explaining Istio VirtualService for Canary deployments, highlighting components like Virtual Service, Destination Rule, and Service, along with their roles in traffic management." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/istio-virtualservice-canary-deployments-diagram.jpg" />
</Frame>

## Example Istio configuration

VirtualService — the `http` route array contains `destination` entries with `host`, `subset`, and `weight`. Weights are interpreted as percentages and typically add up to 100 (proxies will normalize values if needed).

```yaml theme={null}
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-vs
spec:
  hosts:
    - my-app
  http:
    - route:
        - destination:
            host: my-app
            subset: stable
          weight: 95
        - destination:
            host: my-app
            subset: canary
          weight: 5
```

DestinationRule — define subsets that map to pod labels. The VirtualService references these subset names.

```yaml theme={null}
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
```

<Callout icon="lightbulb" color="#1CB2FE">
  Weights in the VirtualService are interpreted as percentages (typically summing to 100); the routing decision is made by the sidecar proxies (Envoy) based on these weights, not by the Kubernetes Service.
</Callout>

<Callout icon="warning" color="#FF6B6B">
  Ensure sidecar injection is enabled for the workloads and that subset labels in the DestinationRule match Pod labels exactly. Otherwise, istiod/Envoy cannot route to the intended subset and traffic will fall back to the default route.
</Callout>

## Automating canaries with Argo Rollouts

Argo Rollouts can orchestrate canary progression by updating Istio VirtualService weights and running automated analysis. Typical flow:

1. Argo Rollouts creates a canary ReplicaSet alongside the stable ReplicaSet.
2. At each step, Argo updates the Istio `VirtualService` weights (e.g., 5%, then 50%, then 100%).
3. After each weight change, Argo runs analysis (for example, Prometheus queries) to evaluate metrics like error rate or latency.
4. If analysis fails, Argo Rollouts can automatically roll back by reverting `VirtualService` weights and scaling down the canary ReplicaSet.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/argo-rollouts-istio-integration-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=ff7dacf649c105435db1bd1509ebeff3" alt="The image outlines the integration of Argo Rollouts with Istio, highlighting features like creating canary ReplicaSets, updating VirtualService weights, running analysis with Prometheus, and enabling auto-rollback." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Service-Mesh-Integration-for-Progressive-Delivery/argo-rollouts-istio-integration-diagram.jpg" />
</Frame>

This combination — Istio's L7 routing (Envoy) and Argo Rollouts' automation and analysis — enables safe, automated progressive delivery with precise traffic control and request-level observability.

## Links and references

* Istio Service Mesh basics: [https://learn.kodekloud.com/user/courses/istio-service-mesh](https://learn.kodekloud.com/user/courses/istio-service-mesh)
* Envoy proxy: [https://www.envoyproxy.io/](https://www.envoyproxy.io/)
* Argo Rollouts: [https://argoproj.github.io/argo-rollouts/](https://argoproj.github.io/argo-rollouts/)
* Prometheus: [https://prometheus.io/](https://prometheus.io/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/5595a5c2-6b2e-4aeb-9fdb-4234e8da8beb" />
</CardGroup>
