> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Pod Security Standards Your Baseline Safety Net

> Explains Kubernetes Pod Security Standards, the three profiles, enforcement modes, and namespace label configuration to enforce baseline runtime protections and layer custom policies.

Pod Security Standards (PSS) are Kubernetes' built‑in mechanism for restricting what Pods may do at runtime. PSS enforces a baseline safety net directly in the API server—no installation required—making it a lightweight, reliable first line of defense. It replaces the deprecated PodSecurityPolicy removed in Kubernetes 1.25.

In this article you'll learn the three PSS levels, how enforcement modes are applied and combined, and how to enable PSS using only namespace labels.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/pod-security-learning-objectives-list.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=d2a23c007718f9f26fc03c6ae6c2bb8f" alt="The image lists learning objectives related to Pod Security, including understanding security levels, applying enforcement modes, configuring security with namespace labels, and implementing security settings for compliance." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/pod-security-learning-objectives-list.jpg" />
</Frame>

## The three security levels

PSS defines three named profiles that express increasing restrictions. Use them to scope what pod configurations are allowed in a namespace.

* Privileged — no restrictions. Pods may use host network, hostPID, privileged containers, run as root, etc. Reserve this for cluster infrastructure namespaces (for example `kube-system`), CNI plugins, or monitoring agents that genuinely require host access.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/privileged-security-level-max-risk.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=cfbc92dd5868693618e845de039231e9" alt="The image outlines the first of three security levels, labeled &#x22;Privileged.&#x22; It describes the level as having maximum risk with no restrictions, allowing pods extensive actions such as running as root and accessing the host network." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/privileged-security-level-max-risk.jpg" />
</Frame>

* Baseline — the recommended default for most application namespaces. Baseline blocks dangerous host-level privileges (for example, `hostNetwork`, `hostPID`, `privileged` containers) while remaining compatible with typical containerized applications.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/security-levels-diagram-baseline-risk.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=578f76bbbce29db05197aa8ac1ccf928" alt="The image depicts a diagram of three security levels: Privileged, Baseline, and another unspecified level, with a focus on the Baseline level, which describes its moderate risk and security features for typical web apps and microservices." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/security-levels-diagram-baseline-risk.jpg" />
</Frame>

* Restricted — the strictest profile. Restricted includes Baseline restrictions and additionally requires pods to run as non-root users, disables privilege escalation, limits Linux capabilities, and encourages read-only root filesystems and selinux/seccomp profiles where appropriate. Use Restricted for sensitive workloads that must satisfy strong runtime controls.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/security-levels-privileged-baseline-restricted.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=1ff34d443204ff6f5a5bafa055207059" alt="The image shows a diagram of three security levels labeled as Privileged, Baseline, and Restricted, with details under Restricted about minimizing risk and maximizing security." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/security-levels-privileged-baseline-restricted.jpg" />
</Frame>

Key guidance: use `baseline` as your default for application namespaces, `restricted` for high‑security workloads, and `privileged` only where host access is required.

| Level      | Summary                                                       | Use case                                                       |
| ---------- | ------------------------------------------------------------- | -------------------------------------------------------------- |
| Privileged | No runtime restrictions; highest risk                         | Cluster infra (CNI, monitoring agents), kube-system components |
| Baseline   | Blocks host-level privileges; compatible with most apps       | Default application namespaces                                 |
| Restricted | Enforces non-root, minimal capabilities, seccomp/read-only fs | Sensitive workloads, compliance zones                          |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/security-levels-pod-configurations-table.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=b3b32f5a9add914e978f8b0d6391fa0f" alt="The image is a table showing the configurations blocked at each security level (Privileged, Baseline, Restricted) for pod specifications, such as privileged containers, host networking, and running as root." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/security-levels-pod-configurations-table.jpg" />
</Frame>

Memorize the distinction: Baseline prevents dangerous host access; Restricted additionally locks down the container runtime and user/privileges.

## Enforcement modes — how to apply the levels

PSS supports three independent enforcement modes that you can set per namespace. Best practice is to stack modes so teams can iterate toward stricter profiles without breaking running workloads.

* `enforce` — hard blocker. Pods that violate the specified level are rejected during admission.
* `audit` — log only. Violations are recorded in audit logs but pods are allowed.
* `warn` — user-visible warning. Pods are allowed, but API responses include a warning message.

| Mode      | Effect                            | Typical usage                                |
| --------- | --------------------------------- | -------------------------------------------- |
| `enforce` | Rejects non-compliant pods        | Apply to `baseline` for app namespaces       |
| `audit`   | Records violations in audit logs  | Monitor `restricted` impact before enforcing |
| `warn`    | Returns warnings in API responses | Educate developers about upcoming changes    |

A common pattern: enforce `baseline` to immediately block host-level risks, and use `audit`/`warn` for `restricted` so teams can remediate before you enable enforcement.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/enforcement-modes-pod-violations-diagram.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=241d8eac38b860f03cab3ef8cc3fcd02" alt="The image illustrates three enforcement modes: &#x22;Enforce&#x22; (hard block), &#x22;Warn&#x22; (visible warning), and &#x22;Audit&#x22; (silent logging), for handling pod violations. Each mode is depicted with a brief description of its function." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/enforcement-modes-pod-violations-diagram.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Recommended practice: Enforce `baseline` for all application namespaces. Use `audit`/`warn` at `restricted` to surface work needed to move workloads to the stricter profile without breaking them.
</Callout>

## Applying Pod Security Standards with namespace labels

PSS is label-driven: no CRDs, no external admission controllers required. You apply levels and modes by labeling namespaces with the `pod-security.kubernetes.io/*` keys.

Example labels:

```yaml theme={null}
pod-security.kubernetes.io/enforce: baseline   # blocks pods that violate baseline
pod-security.kubernetes.io/audit: restricted   # logs restricted violations
pod-security.kubernetes.io/warn: restricted    # warns about restricted violations
```

To set these labels, use `kubectl label`. Example — apply `restricted` for `enforce` and `warn` on the `payments` namespace:

```bash theme={null}
kubectl label namespace payments \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
```

Verify labels:

```bash theme={null}
kubectl get namespace payments --show-labels
# or list all namespaces that have pod-security labels
kubectl get namespace --show-labels | grep pod-security
```

You can also pin behavior to a specific API behavior version using `pod-security.kubernetes.io/enforce-version` (for example `v1.27`) or set it to `latest`. If omitted, the cluster default behavior is used.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/pss-namespace-labels-kubernetes-guide.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=adb159ed3582b274d0fce63ea67a611c" alt="The image is a guide for applying PSS namespace labels in Kubernetes, detailing label format with specific prefixes, modes, and levels." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/pss-namespace-labels-kubernetes-guide.jpg" />
</Frame>

<Callout icon="warning" color="#FF6B6B">
  Do not casually label core system namespaces (for example `kube-system`) as `restricted` or `baseline` unless you’ve validated operator and add-on compatibility. Mislabeling control-plane or infrastructure namespaces can break cluster components.
</Callout>

## PSS vs Gatekeeper / Kyverno

PSS provides a zero-install baseline covering critical pod-level security fields. It is intentionally limited in scope. For organization-specific constraints you should layer a policy engine.

* PSS: built-in, minimal setup, enforces core runtime controls.
* Gatekeeper / Kyverno: support complex, custom policies (for example, image registries, required labels, resource quotas, mutation, multi-field constraints).

Use PSS as the mandatory floor across all namespaces, then add Gatekeeper or Kyverno for additional operational policies.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/pss-gatekeeper-kyverno-comparison-chart.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=74f67ed15cf903fec8c91e69ce53c345" alt="The image is a comparison chart of PSS, Gatekeeper, and Kyverno, highlighting aspects such as installation, scope, granularity, mutation, and use cases. It suggests using PSS as a baseline with Gatekeeper or Kyverno for custom policies." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/pss-gatekeeper-kyverno-comparison-chart.jpg" />
</Frame>

## Summary / Key takeaways

* PSS levels: `privileged` (most permissive), `baseline` (recommended default), `restricted` (most restrictive).
* Enforcement modes per-namespace: `enforce` (block), `audit` (log), `warn` (inform). Stack modes so teams can migrate safely.
* PSS uses namespace labels only—no installation required. Use `kubectl label` to manage them and consider `enforce-version` when pinning behavior.
* For `restricted` compliance, important settings include running as non-root, disabling privilege escalation, minimizing capabilities, enforcing seccomp, and using a read-only root filesystem where feasible.
* Treat PSS as your baseline safety net and layer Gatekeeper or Kyverno for organization-specific policies.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/pod-security-levels-key-takeaways.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=8b52438377d889663e778dd8195321ae" alt="The image outlines four key security takeaways related to pod security levels, modes, namespace label application, and settings for restricted pods." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Pod-Security-Standards-Your-Baseline-Safety-Net/pod-security-levels-key-takeaways.jpg" />
</Frame>

## Links and references

* [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
* [Kubernetes Namespace Labels for Pod Security](https://kubernetes.io/docs/concepts/security/pod-security-standards/#enforcement)
* [Gatekeeper (Open Policy Agent)](https://open-policy-agent.github.io/gatekeeper/)
* [Kyverno](https://kyverno.io/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/f777f6c0-ca36-4564-9abf-6ef8f548f02f" />
</CardGroup>
