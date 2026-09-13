> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Kyverno Kubernetes Native Policy Management

> Guide to Kyverno, a Kubernetes-native policy engine explaining validate, mutate, and generate rules, examples, comparisons with Gatekeeper, debugging, and rollout best practices.

Kyverno is a Kubernetes-native alternative to Gatekeeper that expresses policy using pure Kubernetes YAML. If you can write Kubernetes manifests, you can author Kyverno policies — no new language to learn.

In this guide we explain Kyverno’s three rule types (validate, mutate, generate), show concise policy examples and patterns, explain when to choose Kyverno over Gatekeeper, and cover debugging and common issues.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-learning-objectives-diagram.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=647cddfedf9192b9fe634924f5a9ddbf" alt="The image is a diagram listing learning objectives related to Kyverno, including understanding rule types, implementing policy patterns, and comparing Kyverno with Gatekeeper." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-learning-objectives-diagram.jpg" />
</Frame>

## Overview: Kyverno rule types

A single `Policy` or `ClusterPolicy` resource can contain multiple rules of three types:

* validate — check resources and reject non-compliant objects (similar to Gatekeeper, but expressed in YAML pattern matching instead of Rego).
* mutate — modify resources (add defaults, labels, env vars, etc.) via a mutating admission webhook.
* generate — create related resources automatically when a matching resource is created (for example, create a default-deny `NetworkPolicy` when a Namespace is created).

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-policy-three-rule-types-outline.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=8244866c57950eb6f637c535f0041811" alt="The image outlines the three rule types in a Kyverno policy: Validate, Mutate, and a third unspecified type, emphasizing that Kyverno uses pure Kubernetes YAML." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-policy-three-rule-types-outline.jpg" />
</Frame>

## Validate, Mutate, Generate — concise explanation

* Validate: Uses YAML pattern matching and wildcards to assert required fields and values. Rejections occur when the resource does not match the allowed pattern.
* Mutate: Runs as a mutating admission webhook and modifies the incoming resource before it is persisted. Useful for adding default labels, setting `imagePullPolicy`, injecting environment variables, etc.
* Generate: Automatically creates another resource in response to a matching resource creation (common for guardrails like `NetworkPolicy`, `ResourceQuota`, `LimitRange`, or service accounts).

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-policy-rules-validate-mutate-generate.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=2cd8f91b4b515855119da82e6bd57c31" alt="The image outlines three types of rules in a Kyverno policy: Validate, Mutate, and Generate, emphasizing that Kyverno uses pure Kubernetes YAML with no new language to learn." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-policy-rules-validate-mutate-generate.jpg" />
</Frame>

## Examples

Below are minimal, copy-ready policy examples that illustrate common patterns.

### 1) Validation example — restrict image registries

```yaml theme={null}
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-registries
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: >
          Images must be from
          registry.company.com
        pattern:
          spec:
            containers:
              - image: "registry.company.com/*"
```

Key points:

* A single `ClusterPolicy` holds all rules (no templates, no Rego).
* `spec.validationFailureAction: Enforce` causes non-compliant objects to be rejected. Use `Audit` while testing.

<Callout icon="lightbulb" color="#1CB2FE">
  Use `validationFailureAction: Audit` during rollout to collect violations without blocking resources. Review `PolicyReport` and `ClusterPolicyReport` entries before switching to `Enforce`.
</Callout>

### 2) Mutate example — add a default label

```yaml theme={null}
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-labels
spec:
  rules:
    - name: add-team-label
      match:
        any:
          - resources:
              kinds: ["Deployment", "Service"]
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              managed-by: platform-team
```

Notes:

* `patchStrategicMerge` merges labels and will not overwrite an existing label value; it adds the label only if missing.
* Use `patchJson6902` for precise add/remove/replace operations at JSON paths.

### 3) Combine mutate + validate in one policy (pattern)

```yaml theme={null}
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enforce-managed-by-label
spec:
  validationFailureAction: Enforce
  rules:
    - name: add-managed-by-label
      match:
        any:
          - resources:
              kinds: ["Deployment", "Service"]
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              managed-by: platform-team

    - name: require-managed-by-label
      match:
        any:
          - resources:
              kinds: ["Deployment", "Service"]
      validate:
        message: "The 'managed-by=platform-team' label is required"
        pattern:
          metadata:
            labels:
              managed-by: "platform-team"
```

Behavior:

* Resources without the label will have it added automatically by the mutate rule.
* If the label is removed later, the validate rule will reject updates that do not match (subject to webhook ordering and timing).

<Callout icon="warning" color="#FF6B6B">
  Mutations run before validations. Be aware of webhook ordering and timing: a validate rule may rely on a prior mutate rule to add required fields. Test to ensure the intended behavior.
</Callout>

### 4) Generate example — create a default-deny NetworkPolicy per Namespace

```yaml theme={null}
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: default-deny-ingress
spec:
  rules:
    - name: generate-default-netpol
      match:
        any:
          - resources:
              kinds: ["Namespace"]
      generate:
        kind: NetworkPolicy
        apiVersion: networking.k8s.io/v1
        name: default-deny-ingress
        namespace: "{{request.object.metadata.name}}"
        data:
          spec:
            podSelector: {}
            policyTypes: ["Ingress"]
```

Notes:

* The `namespace` field references the created Namespace using the expression `{{request.object.metadata.name}}`.
* Use `generate` to ensure guardrails are created automatically for new namespaces.

## Kyverno vs Gatekeeper — when to choose what

| Aspect          | Kyverno                                                     | Gatekeeper (OPA / Rego)                                          |
| --------------- | ----------------------------------------------------------- | ---------------------------------------------------------------- |
| Policy language | Pure Kubernetes YAML                                        | Rego                                                             |
| Rule types      | Validate, Mutate, Generate (in a single policy)             | Validate only                                                    |
| Best fit        | Mutation, simple-to-moderate validation using YAML patterns | Very complex validation logic requiring full Rego expressiveness |
| Learning curve  | Low for Kubernetes users                                    | Higher — must learn Rego                                         |
| Coexistence     | Can coexist with Gatekeeper (separate webhooks)             | Can coexist with Kyverno                                         |

Links and references:

* [Kyverno documentation](https://kyverno.io/docs/)
* [Gatekeeper / OPA documentation](https://www.openpolicyagent.org/)
* [Kubernetes admission controllers overview](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-opa-gatekeeper-comparison-chart.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=67bb2eb988438dbf869b6599be3b3f2c" alt="The image is a comparison chart between Kyverno and OPA Gatekeeper, evaluating them based on complex logic, architecture, and exam weight. It highlights Kyverno's limited pattern matching and simpler YAML policies against OPA Gatekeeper's full Rego language and deeper Rego questions." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-opa-gatekeeper-comparison-chart.jpg" />
</Frame>

Kyverno and Gatekeeper can run in the same cluster because they register independent admission webhooks.

## Debugging, status and common commands

* `kubectl get clusterpolicy` shows Kyverno ClusterPolicy objects and the READY column (policy loaded and valid).
* Kyverno emits `PolicyReport` and `ClusterPolicyReport` resources for audit-mode violations.
* Check controller logs when webhook errors occur.

Commands quick reference:

```bash theme={null}
# List all cluster policies and their status
kubectl get clusterpolicy

# Check policy reports (audit-mode violations)
kubectl get policyreport -A
kubectl get clusterpolicyreport

# Describe a PolicyReport for a specific namespace (example: payments)
kubectl describe policyreport -n payments

# Check Kyverno controller logs
kubectl logs -n kyverno deployment/kyverno
```

Common issues and how to troubleshoot:

* READY = false: usually indicates invalid policy YAML or a schema mismatch. Inspect `kubectl describe clusterpolicy <name>` and Kyverno logs.
* Policies not taking effect: verify `match` criteria are correct; it’s common for match scope to not include the resources you expect.
* No rejections while in `Enforce`: ensure the policy truly matches the resources and the pattern is correct.
* Webhook errors: check Kyverno controller health and logs; confirm webhook configurations and certificates.

## Key takeaways

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-kubernetes-yaml-rule-types.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=501c60a8a896549e3e046a6eb3f1f28e" alt="The image contains key takeaways about Kyverno, highlighting its use of Kubernetes YAML, rule types (validate, mutate, generate), and a safe rollout strategy." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Kyverno-Kubernetes-Native-Policy-Management/kyverno-kubernetes-yaml-rule-types.jpg" />
</Frame>

* Kyverno uses pure Kubernetes YAML — no extra language required.
* One policy resource can include validate, mutate, and generate rules.
* Adopt a safe rollout: audit mode first, review `PolicyReport`s, then switch to enforce.
* Choose Kyverno when you need mutation or YAML-native policies; choose Gatekeeper when Rego’s expressiveness is required.
* Kyverno and Gatekeeper can coexist in the same cluster.

Further reading:

* Kyverno docs: [https://kyverno.io/docs/](https://kyverno.io/docs/)
* Gatekeeper: [https://open-policy-agent.github.io/gatekeeper/](https://open-policy-agent.github.io/gatekeeper/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/1a5a6874-6438-434c-a731-163beffe04ea" />
</CardGroup>
