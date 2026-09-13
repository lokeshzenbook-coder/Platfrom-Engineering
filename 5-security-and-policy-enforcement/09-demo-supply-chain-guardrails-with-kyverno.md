> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Supply Chain Guardrails with Kyverno

> Guide showing how to use Kyverno to validate, mutate, and generate Kubernetes resources for supply chain guardrails, with examples and Audit mode PolicyReports

[OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/) is a powerful policy engine, but it requires learning the Rego language. Kyverno offers an alternative: policies are written as Kubernetes-native YAML, so you can author validate, mutate, and generate rules without learning a new language.

This guide verifies Kyverno is running and demonstrates three common policy patterns:

* validate — allow or deny resources
* mutate — automatically modify resources
* generate — create resources in response to events

You’ll also learn how to use Audit mode and PolicyReports to safely roll out policies.

## At a glance — Kyverno rule types

| Rule Type | Purpose                                   | Typical Use Case                                    |
| --------- | ----------------------------------------- | --------------------------------------------------- |
| validate  | Allow or deny resources based on patterns | Enforce approved image registries, required labels  |
| mutate    | Modify resources at admission             | Inject labels, set default resource requests/limits |
| generate  | Create resources in response to events    | Auto-create NetworkPolicy, ConfigMap, RoleBindings  |

Quick links:

* Kyverno docs: [https://kyverno.io](https://kyverno.io)
* Kyverno policy examples: [https://kyverno.io/docs/writing-policies/](https://kyverno.io/docs/writing-policies/)
* OPA / Rego reference: [https://www.openpolicyagent.org/docs/latest/policy-language/](https://www.openpolicyagent.org/docs/latest/policy-language/)

## Prerequisites

* A running Kubernetes cluster with Kyverno installed in the `kyverno` namespace.
* `kubectl` configured for the target cluster.

## Verify Kyverno is running

Check Kyverno pods in the `kyverno` namespace:

```bash theme={null}
kubectl get pods -n kyverno
```

Example output:

```text theme={null}
NAME                                              READY   STATUS    RESTARTS   AGE
kyverno-admission-controller-659d58644b-8jbfb     1/1     Running   0          115m
kyverno-background-controller-778fbf669-cw58      1/1     Running   0          115m
kyverno-cleanup-controller-8c8f4578-cmbj          1/1     Running   0          115m
kyverno-reports-controller-6c666d96-7xgz7         1/1     Running   0          115m
```

If any Kyverno pods are not `Running`, check pod logs and events to diagnose installation issues.

## 1) Validate rule example — restrict image registries

This example enforces that Pod container images must come from `docker.io`. The policy uses Kyverno YAML pattern overlays — no Rego required.

File: `validate-registry.yaml`

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
              kinds:
                - Pod
      validate:
        message: "Image must be from docker.io"
        pattern:
          spec:
            containers:
              - image: "docker.io/*"
```

Notes:

* `validationFailureAction: Enforce` blocks requests that violate the policy. Switch to `Audit` to only record violations while allowing creation.
* Kyverno uses YAML pattern overlays so policies are readable and Kubernetes-native.

Apply the policy:

```bash theme={null}
kubectl apply -f validate-registry.yaml
# clusterpolicy.kyverno.io/restrict-image-registries created
```

Test with an untrusted image from `quay.io`:

File: `test-pod-untrusted.yaml`

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-pod
spec:
  containers:
    - name: test
      image: quay.io/prometheus/prometheus:latest
```

Attempt to create the Pod in namespace `team-dev`:

```bash theme={null}
kubectl apply -f test-pod-untrusted.yaml -n team-dev
```

If enforcement is enabled, the admission webhook will deny the request. Example error:

```text theme={null}
Error from server: error when creating "test-pod-untrusted.yaml": admission webhook "validate.kyverno.svc-fail" denied the request:
resource Pod/team-dev/untrusted-pod was blocked due to the following policies
restrict-image-registries:
  validate-registries: 'validation error: Image must be from docker.io. rule validate-registries'
    failed at path /spec/containers/0/image/
```

Now try a trusted image from `docker.io`:

File: `test-pod-trusted.yaml`

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: trusted-pod
spec:
  containers:
    - name: app
      image: docker.io/library/nginx
```

Apply it:

```bash theme={null}
kubectl apply -f test-pod-trusted.yaml -n team-dev
# pod/trusted-pod created
```

The trusted pod is admitted because it matches the policy pattern.

<Callout icon="lightbulb" color="#1CB2FE">
  You can switch `validationFailureAction` to `Audit` to collect violations without blocking resources.
</Callout>

## 2) Mutate rule example — auto-inject labels

Use a mutate rule to automatically inject labels into Pods created in the `team-dev` namespace using a strategic merge patch.

File: `mutate-labels.yaml`

```yaml theme={null}
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-labels
spec:
  rules:
    - name: add-managed-by
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - team-dev
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              managed-by: kyverno
              environment: dev
```

Apply the policy:

```bash theme={null}
kubectl apply -f mutate-labels.yaml
# clusterpolicy.kyverno.io/add-labels created
```

Create a pod (use a `docker.io` image so the validate policy does not block it):

```bash theme={null}
kubectl run mutation-test --image=docker.io/library/nginx -n team-dev
# pod/mutation-test created
```

Check the labels injected by Kyverno:

```bash theme={null}
kubectl get pod -n team-dev mutation-test --show-labels
```

Example output:

```text theme={null}
NAME            READY   STATUS    RESTARTS   AGE    LABELS
mutation-test   1/1     Running   0          17s   environment=dev,managed-by=kyverno,run=mutation-test
```

The labels `managed-by=kyverno` and `environment=dev` were added automatically by the mutate rule.

## 3) Generate rule example — auto-create NetworkPolicy on Namespace creation

A generate rule can create a default-deny `NetworkPolicy` whenever a `Namespace` is created.

File: `generate-netpol.yaml`

```yaml theme={null}
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-networkpolicy
spec:
  rules:
    - name: default-deny
      match:
        any:
          - resources:
              kinds:
                - Namespace
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny
        namespace: "{{request.object.metadata.name}}"
        spec:
          podSelector: {}
          policyTypes:
            - Ingress
            - Egress
```

Notes:

* The `namespace` field uses the template `{{request.object.metadata.name}}` so the generated NetworkPolicy is created inside the Namespace that triggered the rule.

Apply the policy:

```bash theme={null}
kubectl apply -f generate-netpol.yaml
# clusterpolicy.kyverno.io/generate-networkpolicy created
```

Create a namespace to trigger generation:

```bash theme={null}
kubectl create namespace auto-np-test
# namespace/auto-np-test created
```

Verify the generated NetworkPolicy:

```bash theme={null}
kubectl get networkpolicy -n auto-np-test
```

Example output:

```text theme={null}
NAME          POD-SELECTOR   AGE
default-deny  <none>         11s
```

Kyverno automatically created `default-deny` in `auto-np-test`.

## 4) Audit mode and PolicyReports

Audit mode allows resources that violate policies to be created while Kyverno records the violations in PolicyReport resources. This is useful for observing policy impact before enforcing.

Patch the `restrict-image-registries` policy to `Audit`:

```bash theme={null}
kubectl patch clusterpolicy restrict-image-registries \
  --type=merge \
  -p '{"spec":{"validationFailureAction":"Audit"}}'
# clusterpolicy.kyverno.io/restrict-image-registries patched
```

Now apply the untrusted pod again:

```bash theme={null}
kubectl apply -f test-pod-untrusted.yaml -n team-dev
# pod/untrusted-pod created
```

The Pod is created (not blocked), and Kyverno records the policy results as PolicyReport resources.

List policy reports in the namespace:

```bash theme={null}
kubectl get policyreport -n team-dev
```

Example output (summary):

```text theme={null}
NAME                                       KIND   PASS   FAIL   WARN   ERROR   SKIP   AGE
579d07e7-742c-4299-9147-8e7584106c30       Pod    1      1      0      0       0     0s
274e0a72-4abc-9fa5-9147-9b4ef95fc68e       Pod    1      0      0      0       0     7m9s
76b686df-cc1a-4e7e-821f-cba9b988d981       Pod    2      0      0      0       0     4m19s
```

Describe a PolicyReport for the untrusted pod to view details:

```bash theme={null}
kubectl describe policyreport 579d07e7-742c-4299-9147-8e7584106c30 -n team-dev
```

Example (truncated) output from the PolicyReport:

```text theme={null}
Results:
  - Message: mutated Pod/untrusted-pod in namespace team-dev
    Policy: add-labels
    Properties:
      Process: background scan
      Result: pass
      Rule: add-managed-by
      Scored: true
      Source: kyverno
    Timestamp: 2026-04-15T18:27:20Z

  - Message: validation error: Image must be from docker.io. rule validate-registries failed at path /spec/containers/0/image/
    Policy: restrict-image-registries
    Properties:
      Process: background scan
      Result: fail
      Rule: validate-registries
      Scored: true
      Source: kyverno
    Timestamp: 2026-04-15T18:27:30Z

Summary:
  Pass: 1
  Fail: 1
  Total: 2
```

This output shows:

* The mutate rule passed and injected labels.
* The validate rule failed (image not from `docker.io`), but because the policy is in Audit mode, the Pod was admitted and the violation was captured for visibility.

<Callout icon="lightbulb" color="#1CB2FE">
  Use Audit mode to evaluate policy impact and blast radius. Once you are confident, switch policies back to `Enforce` to block violating resources.
</Callout>

## Summary and best practices

* Kyverno policies are Kubernetes-native YAML — no Rego required.
* Use the three complementary rule types:
  * validate: block or audit resources
  * mutate: change resources at admission (strategic merge, JSON patches, etc.)
  * generate: create resources in response to events (e.g., default-deny NetworkPolicy)
* Start in Audit mode and review PolicyReports to safely roll out policies.
* Combine mutate + validate rules to auto-fix common issues and enforce desired state.

## Links and references

* Kyverno documentation: [https://kyverno.io/docs/](https://kyverno.io/docs/)
* Kyverno policy examples: [https://kyverno.io/docs/writing-policies/](https://kyverno.io/docs/writing-policies/)
* OPA Gatekeeper: [https://open-policy-agent.github.io/gatekeeper/](https://open-policy-agent.github.io/gatekeeper/)
* Rego language: [https://www.openpolicyagent.org/docs/latest/policy-language/](https://www.openpolicyagent.org/docs/latest/policy-language/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/a82e67a7-e75e-4ef0-be81-4862c665ef4f" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/11c61a43-720a-4ba7-b400-2bc8580dfd1f" />
</CardGroup>
