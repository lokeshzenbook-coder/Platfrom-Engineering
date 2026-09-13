> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Platform Security Simplified Threats Guardrails and Trust

> Summarizes Kubernetes threat model and layered platform controls to enforce least privilege, network isolation, admission policies, service mesh, and supply chain security for shared clusters

Welcome to the final domain: Security and Policy Enforcement.

This module turns everything you've built into a platform that is secure by default. Read on to learn the Kubernetes threat model, a practical defense-in-depth strategy, and how each security layer maps to specific tools and controls. Subsequent lessons will build on these foundations.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/kubernetes-security-learning-objectives.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=a285627753c953a7a491c0a742b7837a" alt="The image lists five learning objectives related to Kubernetes security, focusing on security challenges, threat models, defense strategies, and exam security domains." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/kubernetes-security-learning-objectives.jpg" />
</Frame>

Why this matters — a real-world scenario

Imagine a fintech startup operating a shared Kubernetes cluster for six teams. One team installs a debugging tool that runs as `cluster-admin` and exposes a web shell via a `NodePort`. An attacker discovers that endpoint, escalates to cluster-admin, and enumerates secrets across namespaces—database credentials, API keys, payment processor tokens.

<Callout icon="warning" color="#FF6B6B">
  Exposed debugging tools, excessive privileges like `cluster-admin`, and publicly reachable NodePorts are a common attack chain. Treat any privileged tooling as high-risk and restrict network exposure.
</Callout>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/shared-platforms-risk-nodeport-debugging.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=3df98c3978ab199a778b7a8d227e10cd" alt="The image illustrates the risk of shared platforms, showing a person with a laptop gaining access through a debugging tool and a NodePort web shell, with cluster-admin access and secret enumeration across namespaces." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/shared-platforms-risk-nodeport-debugging.jpg" />
</Frame>

The financial and regulatory impact can be severe: incident response costs, lost revenue, and compliance fines. The root cause is rarely a single misstep; it’s a chain of failures:

* Improper RBAC: teams can create cluster role bindings or use `cluster-admin`.
* Missing admission controls: privileged containers and unsafe specs are allowed.
* No network isolation: sensitive endpoints are reachable from outside.
* No image scanning: known CVEs and malicious artifacts are deployed.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/root-causes-shared-platforms-risk.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=974a9398063dfeb979ee31d93781f753" alt="The image outlines four root causes related to shared platforms and amplified risk: no RBAC restrictions, no admission control, no network policies, and no image scanning." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/root-causes-shared-platforms-risk.jpg" />
</Frame>

What makes Kubernetes uniquely challenging to secure?

* Multi-tenancy by design: misconfigurations in one namespace can affect others.
* A powerful API server: it can create pods, mount secrets, and perform cluster-wide actions.
* Flat default network: pods can reach each other unless isolation is applied.
* Public images and supply chain risk: base images or charts can contain vulnerabilities or backdoors.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/shared-platforms-risks-multi-tenancy-api.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=16c467a446fd0e32c9db536213ff2d08" alt="The image outlines the risks associated with shared platforms, highlighting features like multi-tenancy, powerful API surfaces, flat network structure, and software supply chain vulnerabilities." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/shared-platforms-risks-multi-tenancy-api.jpg" />
</Frame>

As a platform engineer your goal is simple: make the secure path the easy path. That means enforcing platform-level defaults, applying least privilege everywhere, and building layered defenses that don’t rely on perfect human behavior.

Understand the attackers

Common threat vectors you must defend against:

* External attackers: scanning for exposed services, leaked `kubeconfig` files, or unauthenticated dashboards.
* Insider or misconfiguration misuse: overly-permissioned users or automation making destructive mistakes.
* Supply chain attacks: vulnerable base images, malicious Helm charts, or unsigned artifacts.
* Lateral movement: compromised pods using service account tokens to enumerate secrets and pivot.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/kubernetes-threat-model-categories-outline.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=fc022959a33ffaae5c2c2530116bb4c1" alt="The image outlines the Kubernetes Threat Model, detailing four threat categories: External Attacker, Insider/Misconfigured, Supply Chain Attack, and Lateral Movement. Each category includes examples, such as public dashboard vulnerabilities and misuse of permissions." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/kubernetes-threat-model-categories-outline.jpg" />
</Frame>

Map threats to layered controls (defense in depth)

Defense in depth means overlapping controls so a single failed control doesn’t lead to catastrophe. A practical Kubernetes stack typically includes:

* Role-Based Access Control (RBAC): define who can do what.
* Admission controllers and webhooks: validate and mutate resources before creation.
* Pod Security Standards (PSS): enforce safe runtime profiles (no privileged containers, limited capabilities).
* Network Policies: control pod-to-pod and external traffic.
* Service mesh with mTLS: encrypt traffic and enforce service identity.
* Supply chain security: image scanning, signing, and provenance verification.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/defense-in-depth-six-security-layers.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=7955de2cdb92fc7c893e6696fadca08f" alt="The image illustrates &#x22;Defense in Depth: Six Security Layers,&#x22; which include RBAC, Admission Control, Pod Security, Network Policy, Service Mesh, and Supply Chain Security, each with a brief description." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/defense-in-depth-six-security-layers.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Defense in depth reduces single points of failure. Apply platform-wide defaults so teams inherit secure settings automatically, and require explicit exceptions for risky actions.
</Callout>

How these layers connect — follow a request

1. A user authenticates to the API server (OIDC, client certs, etc.).
2. RBAC evaluates whether the user can create a `Deployment` in the namespace.
3. Admission webhooks validate or mutate the `Deployment`/`Pod` spec against policy.
4. Pod Security Standards enforce runtime constraints before the pod is scheduled.
5. If scheduled, Network Policies determine which peers the pod can reach.
6. A service mesh enforces mTLS to encrypt communication and verify service identities.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/kubernetes-flowchart-layers-user-authentication.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=3cfb9590bf0651d546beafe7e7ceb0c7" alt="The image depicts a flowchart illustrating how layers work together in a Kubernetes setup, covering user authentication, RBAC checks, pod security, network policy, and traffic encryption. Each step has a brief description of its function in the process." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Platform-Security-Simplified-Threats-Guardrails-and-Trust/kubernetes-flowchart-layers-user-authentication.jpg" />
</Frame>

That’s six checkpoints before a workload is running and communicating—so even if one control fails, others remain to contain impact.

Threat vectors vs. recommended controls

| Threat Vector                                      | Primary Controls                                                   | Examples / Tools                                                                |
| -------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| External attacker (exposed services, leaked creds) | Network Policies, API auditing, least-privilege RBAC               | Calico/Antrea NetworkPolicy, audit logs, restrict `NodePort`                    |
| Insider / misconfiguration                         | RBAC, Admission Webhooks, PSS enforcement                          | `rbac.authorization.k8s.io`, Open Policy Agent (OPA) Gatekeeper, Kubernetes PSS |
| Supply chain attacks                               | Image scanning, artifact signing, provenance checks                | Clair/Trivy, Sigstore Cosign, OCI registries with immutable tags                |
| Lateral movement from compromised pod              | Service mesh (mTLS), network segmentation, short-lived credentials | Istio/Linkerd, NetworkPolicy isolation, projected service account tokens        |

Recommended links and references

* Kubernetes RBAC: [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* Pod Security Standards: [https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
* NetworkPolicy guide: [https://kubernetes.io/docs/concepts/services-networking/network-policies/](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
* Sigstore / Cosign for signing images: [https://sigstore.dev/](https://sigstore.dev/)

Summary

* Shared platforms amplify risk; one weak workload can impact all tenants.
* The Kubernetes threat model includes external attackers, insider mistakes, supply chain compromise, and lateral movement.
* Defense in depth—RBAC, admission control, PSS, Network Policies, service mesh, and supply chain controls—reduces single points of failure.
* Make the secure choice the default: platform-level enforcement, least privilege, and automated checks before runtime.

By the end of this module you should be able to identify likely attack chains, pick the right controls for each risk, and implement platform defaults that protect teams without blocking productivity.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/2b352493-edac-45fb-9bbe-a54dfafe892d" />
</CardGroup>
