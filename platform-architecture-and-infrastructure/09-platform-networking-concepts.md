> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Platform Networking Concepts

> Overview of Kubernetes networking covering Services, service types, Ingress versus Gateway API, load balancing, service discovery with CoreDNS, and NetworkPolicy for pod-level network security

We have covered platform organization, compute resource management, and storage. Now we connect everything together: networking.

Kubernetes networking answers three fundamental questions:

* How do Pods find each other inside the cluster? (service discovery)
* How does external traffic reach your applications? (Ingress / Gateway)
* How do you control which pods can talk to which other pods? (network security)

In this guide you will learn:

* The four Kubernetes Service types and when to use each
* How Services provide stable endpoints and load balancing for ephemeral pods
* When to use Ingress vs Gateway API for external traffic
* How NetworkPolicy implements pod-level firewalls (default-deny patterns)
* How CoreDNS enables service discovery via DNS

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/kubernetes-learning-objectives-service-types.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=2f2009947ed5917a06075a4542d4524c" alt="The image displays learning objectives for Kubernetes: explaining the four service types and distinguishing between Ingress and Gateway API." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/kubernetes-learning-objectives-service-types.jpg" />
</Frame>

Problem scenario (why Services matter)

A payments microservice hardcoded the IP address of a downstream fraud-detection pod. After a routine deployment the fraud pod restarted, received a new IP, and the payments service could no longer reach it. Payment authorizations silently failed for 45 minutes before monitoring caught the issue, resulting in many declined legitimate transactions.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/payments-microservice-fraud-issue-transaction.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=1401d012abbde4be1f56dd83b8893c1f" alt="The image illustrates a communication issue between a payments microservice and a fraud detection service, with an IP address mismatch leading to a declined transaction of $320K. An hourglass and clock indicate a time delay of 45 minutes." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/payments-microservice-fraud-issue-transaction.jpg" />
</Frame>

Why this happened

Each Pod gets its own IP, but Pod IPs are ephemeral — they change when a pod restarts, is rescheduled, or replaced during a rollout. Hard-coding a Pod IP works temporarily but fails silently when the pod’s IP changes (timeouts, connection refused), which is why we use higher-level abstractions.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/kubernetes-pods-ip-addresses-issue.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=d4da643fc83fcedbb1a05b3b7effe159" alt="The image illustrates how Kubernetes pods receive new IP addresses when they restart, highlighting the issue with hardcoding IP addresses as they change with pod rescheduling." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/kubernetes-pods-ip-addresses-issue.jpg" />
</Frame>

Three core networking challenges

1. Load balancing — How does a frontend distribute traffic across multiple, dynamically changing backend replicas?
2. Service discovery — How does a frontend find backend replicas that may appear after the frontend is deployed?
3. External access — How do Internet users reach applications when pods have private, non-routable IPs?

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/pod-ips-challenges-load-balancing-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=60484ba059c66a6d2a1d9d73fcdb2ed5" alt="The image outlines three challenges related to pod IPs: load balancing, discovery, and external access, depicted as a target diagram with arrows pointing to different colored sections." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/pod-ips-challenges-load-balancing-diagram.jpg" />
</Frame>

Services: stable endpoints for ephemeral pods

A Kubernetes Service gives you a stable network endpoint that load-balances to the current set of backend pods. Typical flow:

* Client pod calls a Service by DNS name (e.g., `api-service`).
* CoreDNS resolves that name to the Service’s ClusterIP (for example `10.96.50.100`).
* The Service holds a label selector to discover backend pods and maintains an endpoints list.
* A node-level proxy (kube-proxy or CNI equivalent) load-balances traffic sent to the ClusterIP across healthy pods.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/kubernetes-service-endpoints-load-balancing.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=99f506a62ebb8cfd3b717505ed8fdd11" alt="The image illustrates a Kubernetes service providing stable endpoints for pods, showing a client pod calling an API service, which resolves via DNS to a service IP and load balances to multiple pods." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/kubernetes-service-endpoints-load-balancing.jpg" />
</Frame>

Kubernetes Service types (comparison)

| Service Type        | Where it is reachable                | Typical use case                                            | Notes                                                                                |
| ------------------- | ------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| ClusterIP (default) | Inside the cluster only              | Internal APIs, databases                                    | Most services (\~90–95%) should be ClusterIP                                         |
| NodePort            | `<node-ip>:<nodePort>` on every node | Development, on-prem clusters without cloud LB              | Static port on nodes; limited routing features                                       |
| LoadBalancer        | External cloud load balancer IP      | Cloud environments where a single service needs a public IP | Each LoadBalancer Service usually provisions a separate cloud LB (cost implications) |
| ExternalName        | Resolves to an external hostname     | Point a service name to an external DNS name (CNAME)        | No proxying or load-balancing; DNS only                                              |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/kubernetes-service-types-comparison.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=8f6d5bfc47172102e50ae1a5d0a082d4" alt="The image is a comparison of Kubernetes service types: ClusterIP, NodePort, LoadBalancer, and ExternalName, showing access types and use cases for each." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/kubernetes-service-types-comparison.jpg" />
</Frame>

Ingress vs Gateway API (external HTTP routing)

For external HTTP(S) traffic, prefer an Ingress controller over many LoadBalancer Services to reduce cost and centralize routing (host/path routing, TLS termination). An Ingress resource maps hosts/paths to backend Services; an Ingress controller implements the routing and configures a reverse proxy.

Example Ingress (path-based routing):

```yaml theme={null}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 8080
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 8080
```

Internet traffic to `app.example.com` arrives at the Ingress controller; `/api` routes to `api-svc` and `/web` routes to `web-svc`. Common controllers include ingress-nginx, Contour (Envoy), and Traefik.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/smart-http-routing-ingress-controllers.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=f060b004f36531013af03aa0daed980f" alt="The image illustrates the concept of &#x22;Smart HTTP Routing&#x22; using Ingress resources and controllers, detailing the roles of implementing routing rules, watching for resources, and configuring reverse proxies. It includes logos and indicates common ingress controllers like Nginx." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/smart-http-routing-ingress-controllers.jpg" />
</Frame>

Gateway API — modern replacement for Ingress

Gateway API is now generally available and solves many limitations of Ingress by offering:

* Better protocol support (HTTP, TCP, UDP, gRPC)
* Extensibility and richer traffic features (splitting, mirroring, header-based routing)
* Clear role separation between infrastructure, platform, and app teams

Example HTTPRoute with Gateway API:

```yaml theme={null}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-svc
          port: 8080
```

Gateway API introduces three role-oriented resources:

* GatewayClass — chosen by infrastructure providers to define load balancer types (similar to StorageClass).
* Gateway — created by platform operators to instantiate LBs and listeners.
* Routes (HTTPRoute, TCPRoute, UDPRoute, etc.) — defined by application teams to express routing.

This model enables separation of responsibilities: infrastructure picks technology, platform manages exposure (ports/protocols), and app teams manage routing rules.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/ingress-gateway-api-comparison-features.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=165674930f07e54e3a67d34ee75250d8" alt="The image compares the Ingress and Gateway API in terms of features and structure, highlighting the Gateway API's role-oriented model with three key resources for better flexibility and role separation." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/ingress-gateway-api-comparison-features.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Gateway API supports advanced traffic features not available in Ingress: traffic splitting (canaries), request mirroring, header-based routing, and native TCP/UDP routing. Prefer Gateway API for new platform designs when supported by your controller.
</Callout>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/gateway-api-ingress-diagram-features.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=f45886fa40b9d9470fd3d6ada80bdcfd" alt="The image is a diagram explaining the Gateway API as the future of ingress, highlighting its features and role-oriented model with three key resources: GatewayClass, Gateway, and HTTPRoute. It also lists features like traffic splitting for canary deployments, request mirroring, header-based routing, and native TCP/UDP routing." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/gateway-api-ingress-diagram-features.jpg" />
</Frame>

NetworkPolicy: pod-level firewalls

By default, pods can talk to any other pod in the cluster. NetworkPolicy provides pod-to-pod firewall rules so you can implement least privilege (default deny, explicit allow). Example: allow only frontend pods to reach backend pods on TCP/8080.

```yaml theme={null}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-only-from-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Key NetworkPolicy points:

* Namespace-scoped: policies in namespace A do not affect namespace B.
* Additive model: if any policy allows a connection, it is allowed. There is no explicit deny object — you deny by omission.
* Requires a CNI implementation that supports NetworkPolicy; otherwise policies are ignored.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/pod-level-firewalls-network-policies-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=5dc15d4acfbc4ed23b75916e5eb88d89" alt="The image explains pod-level firewalls through network policies that control pod-to-pod traffic, highlighting that they are namespace-scoped, additive, and require a compatible CNI plugin." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/pod-level-firewalls-network-policies-diagram.jpg" />
</Frame>

<Callout icon="warning" color="#FF6B6B">
  NetworkPolicy rules are only enforced if your CNI plugin implements them. Test policies in a non-production environment first — otherwise policies may have no effect and pods will remain fully open.
</Callout>

Service discovery with CoreDNS (DNS naming best practices)

CoreDNS automatically creates DNS records for Services — this is the recommended way to discover services instead of using IPs. Three DNS name forms:

* Short name (same namespace): `api-service`
* Namespace-qualified (cross-namespace): `api-service.payments`
* Fully qualified domain name (FQDN): `api-service.payments.svc.cluster.local`

Examples:

```bash theme={null}
# Same namespace (convenient)
$ curl http://api-service:8080

# Cross-namespace (explicit)
$ curl http://api-service.payments:8080

# Fully qualified (unambiguous)
$ curl http://api-service.payments.svc.cluster.local:8080
```

CoreDNS resolves short names by appending the pod's namespace and cluster domain. Use short names inside a namespace for convenience; use namespace-qualified names for cross-namespace calls to avoid collisions; and use the FQDN in config or environment variables when you need absolute clarity.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/dns-best-practices-pods-services.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=a16aab597ecd2ddc6549bfb89e2972de" alt="The image presents best practices for how pods find services using DNS, emphasizing the use of DNS names over IPs and avoiding hardcoded IPs." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/dns-best-practices-pods-services.jpg" />
</Frame>

Best practices summary

* Use Services (ClusterIP) to abstract pod IPs and enable stable DNS names and virtual IPs. Never hard-code Pod IPs.
* Prefer Gateway API for new platforms (role separation, protocol flexibility, advanced traffic controls). Ingress remains widely used and supported.
* Enforce least-privilege networking with NetworkPolicy: adopt a default-deny posture and add explicit allow rules. Confirm CNI support before enforcing policies.
* Use DNS names for service discovery: short names within a namespace, namespace-qualified names for cross-namespace calls, and FQDN for full clarity.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/networking-services-key-takeaways-dns-gateway.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=38274ef48feff203e337955cf8c1c1a0" alt="The image lists four key takeaways related to networking and services, highlighting the use of DNS names, Gateway API, NetworkPolicies, and services for abstracting pod IPs." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-Architecture-and-Infrastructure/Platform-Networking-Concepts/networking-services-key-takeaways-dns-gateway.jpg" />
</Frame>

Links and references

* Kubernetes Services: [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
* Ingress: [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)
* Gateway API: [https://gateway-api.sigs.k8s.io/](https://gateway-api.sigs.k8s.io/)
* NetworkPolicy: [https://kubernetes.io/docs/concepts/services-networking/network-policies/](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
* CoreDNS: [https://coredns.io/](https://coredns.io/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/427743b0-2f67-473d-bcfe-6aa9ac24ddd9" />
</CardGroup>
