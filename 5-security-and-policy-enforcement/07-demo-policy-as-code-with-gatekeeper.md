> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Policy as Code with Gatekeeper

> Guide demonstrating how to use OPA Gatekeeper to enforce custom Kubernetes admission policies using ConstraintTemplates and Constraints, requiring pod labels and using dryrun, warn, or deny enforcement modes.

In this lesson you'll learn how to enforce custom Kubernetes admission policies using OPA Gatekeeper. Built-in admission controllers (for example, LimitRanger and ResourceQuota) handle resource governance, but they don't cover arbitrary policy requirements such as:

* requiring specific labels on every Pod,
* restricting images to approved registries,
* disallowing the `:latest` image tag.

Gatekeeper extends the admission pipeline with policy-as-code using two primary resources:

* ConstraintTemplate — defines a policy (registers a new CRD and contains Rego logic).
* Constraint — an instance of that policy (sets parameters, scope, and enforcement).

This guide walks through a complete example that enforces the presence of `team` and `environment` labels on Pods.

Important references:

* Gatekeeper documentation: [https://open-policy-agent.github.io/gatekeeper/](https://open-policy-agent.github.io/gatekeeper/)
* OPA/Rego language: [https://www.openpolicyagent.org/docs/latest/policy-language/](https://www.openpolicyagent.org/docs/latest/policy-language/)
* Kubernetes admission controllers: [https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

Check Gatekeeper components are running

```bash theme={null}
kubectl get pods -n gatekeeper-system
```

Example output:

```text theme={null}
NAME                                              READY   STATUS    RESTARTS   AGE
gatekeeper-audit-65f8cb8f99-49x7d                 1/1     Running   0          16m
gatekeeper-controller-manager-59874747568-bz9tr   1/1     Running   0          16m
gatekeeper-controller-manager-59874747568-ksdgp   1/1     Running   0          16m
gatekeeper-controller-manager-59874747568-lmz7s   1/1     Running   0          16m
```

## ConstraintTemplate — define the policy and CRD

Create a ConstraintTemplate that registers a new CRD `RequiredLabels` and embeds the Rego logic to detect missing labels. Save this as `required-labels-template.yaml`:

```yaml theme={null}
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requiredlabels
spec:
  crd:
    spec:
      names:
        kind: RequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requiredlabels

        violation[msg] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label = input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("missing required labels: %v", [missing])
        }
```

What this template does

* `spec.crd.spec.names.kind` creates the CRD Kind `RequiredLabels`.
* `openAPIV3Schema` validates the constraint parameters (`labels` must be an array of strings).
* `targets.rego` contains the Rego policy executed during admission; it compares required labels with provided labels and generates violations when labels are missing.

Apply the template:

```bash theme={null}
kubectl apply -f required-labels-template.yaml
```

Expected response:

```text theme={null}
constrainttemplate.templates.gatekeeper.sh/requiredlabels created
```

Verify the CRD exists:

```bash theme={null}
kubectl get crd | grep requiredlabels
```

Example output:

```text theme={null}
requiredlabels.constraints.gatekeeper.sh   2026-04-15T16:11:26Z
```

## Constraint — instantiate the policy

Instantiate the policy by creating a Constraint of kind `RequiredLabels`. This configures parameters, scope, and the enforcement action. Save as `required-labels-constraint.yaml`:

```yaml theme={null}
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequiredLabels
metadata:
  name: require-labels
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels:
      - team
      - environment
```

Key fields

* `enforcementAction`: `deny`, `dryrun`, or `warn`. Use `deny` to block violating requests.
* `match.kinds`: scopes the constraint to core Pods.
* `parameters.labels`: the required label keys validated by the template schema.

Apply the constraint:

```bash theme={null}
kubectl apply -f required-labels-constraint.yaml
```

Example response:

```text theme={null}
requiredlabels.constraints.gatekeeper.sh/require-labels created
```

Quick reference: Gatekeeper resources

| Resource Type      | Purpose                                         | Example                           |
| ------------------ | ----------------------------------------------- | --------------------------------- |
| ConstraintTemplate | Defines policy logic and registers a CRD        | `required-labels-template.yaml`   |
| Constraint         | Instantiates a policy, sets scope & enforcement | `required-labels-constraint.yaml` |
| Rego policy        | Policy logic executed at admission              | included in `spec.targets.rego`   |

## Test: pod without required labels (expected to be denied)

Create `test-pod-no-labels.yaml`:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: unlabeled-pod
  namespace: policy-test
spec:
  containers:
    - name: nginx
      image: docker.io/library/nginx
```

Attempt to apply it:

```bash theme={null}
kubectl apply -f test-pod-no-labels.yaml -n policy-test
```

Expected admission error:

```text theme={null}
Error from server (Forbidden): error when creating "test-pod-no-labels.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [require-labels] missing required labels: ["environment", "team"]
```

## Test: pod with required labels (expected to be admitted)

Create `test-pod-with-labels.yaml`:

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  namespace: policy-test
  labels:
    team: platform
    environment: prod
spec:
  containers:
    - name: nginx
      image: nginx
```

Apply it:

```bash theme={null}
kubectl apply -f test-pod-with-labels.yaml -n policy-test
```

Example response:

```text theme={null}
pod/labeled-pod created
```

## Dry-run mode: audit without breaking workloads

Before enforcing a blocking policy, use `dryrun` to discover existing violations safely. Patch the constraint to `dryrun`:

```bash theme={null}
kubectl patch requiredlabels require-labels \
  --type=merge \
  -p '{"spec":{"enforcementAction":"dryrun"}}'
```

Expected response:

```text theme={null}
requiredlabels.constraints.gatekeeper.sh/require-labels patched
```

Now reapply the previously rejected unlabeled pod — Gatekeeper will allow creation but will record violations:

```bash theme={null}
kubectl apply -f test-pod-no-labels.yaml -n policy-test
```

Example response:

```text theme={null}
pod/unlabeled-pod created
```

List pods in namespace `policy-test`:

```bash theme={null}
kubectl get pods -n policy-test
```

Example output:

```text theme={null}
NAME           READY   STATUS    RESTARTS   AGE
labeled-pod    1/1     Running   0          14s
unlabeled-pod  1/1     Running   0          54s
```

Describe the constraint to inspect recorded violations:

```bash theme={null}
kubectl describe requiredlabels.constraints.gatekeeper.sh require-labels
```

Example excerpt:

```text theme={null}
Total Violations: 13
Violations:
  Enforcement Action: dryrun
  Group:
    Kind: Pod
    Message: missing required labels: {"environment", "team"}
    Name: unlabeled-pod
    Namespace: policy-test
    Version: v1
  Enforcement Action: dryrun
  Group:
    Kind: Pod
    Message: missing required labels: {"environment", "team"}
    Name: kube-scheduler-ctrlplane
    Namespace: kube-system
    Version: v1
  ...
```

Enforcement action options

| Action   | Behavior                                    | Use case                        |
| -------- | ------------------------------------------- | ------------------------------- |
| `deny`   | Blocks requests that violate the constraint | Full enforcement in production  |
| `dryrun` | Allows requests but records violations      | Safe auditing and onboarding    |
| `warn`   | Allows requests and emits warnings          | Non-blocking guidance for users |

<Callout icon="lightbulb" color="#1CB2FE">
  Start in `dryrun` to discover existing violations, remediate resources where needed, and then switch to `deny` to fully enforce the policy. This staged rollout pattern reduces disruption.
</Callout>

## Summary

* ConstraintTemplate defines a policy: it registers a CRD and contains Rego logic.
* Constraint is a policy instance: it configures parameters, scope, and enforcement mode.
* Use `dryrun` to audit and identify violations before enabling `deny`.
* Gatekeeper is an effective policy-as-code solution for Kubernetes: use it to enforce labels, image registries, tag policies, and other organizational rules.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/4808f9b3-7985-47bf-95e5-d3e2368d1aa5" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/ec93bd81-3b25-4469-9194-50fabe8b8a18" />
</CardGroup>
