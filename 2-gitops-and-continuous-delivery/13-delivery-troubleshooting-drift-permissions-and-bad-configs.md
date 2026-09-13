> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Delivery Troubleshooting Drift Permissions and Bad Configs

> Guide to diagnosing and resolving Argo CD GitOps delivery failures focusing on drift, RBAC permissions, and invalid manifests with diagnostic commands and remediation steps

Everything we've covered so far assumes sync will succeed. When it doesn't, diagnosing the failure quickly is critical. Argo CD applications reported as OutOfSync or SyncFailed usually fall into three categories:

* Drift (external controllers or manual changes)
* Permissions (RBAC / service account limitations)
* Invalid configurations (bad manifests, missing CRDs, missing namespaces)

This guide provides a compact, systematic troubleshooting toolkit with the key diagnostic commands and remediation steps to resolve most GitOps delivery failures.

Three categories cover about ninety percent of GitOps failures. For each category you'll find: the diagnostic question, quick commands to inspect the problem, and typical remediation patterns.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Delivery-Troubleshooting-Drift-Permissions-and-Bad-Configs/learning-objectives-technical-issues-tools.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=89431fc83035e4eee70fafd4fe085c51" alt="The image lists four learning objectives related to resolving technical issues, validating manifests, and using specific tools and commands effectively. The objectives are visually highlighted with numbered tags on a gradient background." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Delivery-Troubleshooting-Drift-Permissions-and-Bad-Configs/learning-objectives-technical-issues-tools.jpg" />
</Frame>

Real-world example
A platform team repeatedly synced an Argo CD application. Git matched the desired manifests, the cluster was reachable, and each sync initially succeeded — but within seconds the app returned to OutOfSync. Root cause: an HPA controlling replicas. Git declared `spec.replicas: 3`, while the HPA scaled the Deployment to 7. Argo CD reconciled the Deployment back to 3; the HPA then bumped it to 7, causing a continuous drift loop.

Key lesson: not every difference between Git and the cluster is an error. Controllers (HPAs, operators) and the API server often mutate objects. Recognize expected drift and tell Argo CD what to ignore.

Common symptoms and usual causes

* Perpetual OutOfSync — often drift from HPAs, operators, or manual edits that continuously change cluster state.
* SyncFailed — typically permission problems: Argo CD cannot create/update/delete resources.
* Degraded / Progressing — resources were applied but are not becoming healthy; usually bad configurations (image pull errors, insufficient resources, missing dependencies).
* Unknown health — Argo CD cannot determine health for a resource type, often because CRDs or health checks are missing.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Delivery-Troubleshooting-Drift-Permissions-and-Bad-Configs/sync-failures-symptoms-outlines.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=a6c6c4811e3d6dbb87c42e2f2f88504c" alt="The image outlines four common symptoms of sync failures: Perpetual OutOfSync, SyncFailed, Degraded/Progressing, and Unknown, along with their potential causes." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Delivery-Troubleshooting-Drift-Permissions-and-Bad-Configs/sync-failures-symptoms-outlines.jpg" />
</Frame>

Quick reference: symptoms → likely cause → first diagnostic command

|                Symptom | Likely cause                                    | First diagnostic command                                      |
| ---------------------: | ----------------------------------------------- | ------------------------------------------------------------- |
|    Perpetual OutOfSync | Drift (HPA, operator, controller, manual edits) | `argocd app diff <app>`                                       |
|             SyncFailed | RBAC / permissions                              | `kubectl auth can-i ... --as=system:serviceaccount:<ns>:<sa>` |
| Degraded / Progressing | Bad manifests, image pull, resources            | `kubectl describe <resource>` / `kubectl logs`                |
|         Unknown health | Missing CRD / custom health checks              | `kubectl get crd` / check Argo CD health overrides            |

Diagnosis and remediation — start with the most common: drift

Drift (most common cause of perpetual OutOfSync)

* Diagnostic question: Why does my app stay OutOfSync even after syncing?
* Diagnostic tool: Argo CD diff (UI or CLI Diff tab) to see field-by-field differences between Git and live cluster.

Example CLI:

```bash theme={null}
# Compare application manifests in Git vs cluster
argocd app diff my-app
```

The diff highlights differences such as `spec.replicas` (Git: 3, live: 7) or extra annotations/fields present only in the cluster.

Common drift sources:

* HPA modifying `spec.replicas`.
* Operators adding fields or `status` subtrees.
* Controllers adding annotations or labels.
* Manual `kubectl` edits or patches.
* API server server-side defaulting.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Delivery-Troubleshooting-Drift-Permissions-and-Bad-Configs/kubernetes-drift-sources-hpa-operators.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=824b57a4b5687ff41945cf268cbcbf87" alt="The image outlines four common sources of drift in Kubernetes configurations: HPA modifying replicas, operators adding fields, manual kubectl edits, and defaulting by the API server." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Delivery-Troubleshooting-Drift-Permissions-and-Bad-Configs/kubernetes-drift-sources-hpa-operators.jpg" />
</Frame>

When drift is expected: ignore it
If a controller is intentionally managing a field (for example, an HPA managing `spec.replicas`), add an `ignoreDifferences` entry in the `Application` spec so Argo CD does not treat that as drift.

Example Application snippet:

```yaml theme={null}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  # ... source / destination / syncPolicy ...
  ignoreDifferences:
    - group: apps
      kind: Deployment
      name: my-app
      jsonPointers:
        - /spec/replicas
```

<Callout icon="warning" color="#FF6B6B">
  Only ignore fields that are purposefully controlled by external controllers. Overusing `ignoreDifferences` can mask genuine drift or configuration mistakes.
</Callout>

Permissions (RBAC) issues

* Symptoms: `SyncFailed` with `forbidden`, partial syncs (some resources created while others fail), or inability to delete resources during auto-prune.
* Cause: Argo CD's service account lacks the required create/update/delete permissions for certain resource kinds, or the ApplicationProject restricts allowed kinds.

Diagnostic tools:

* `kubectl auth can-i` with impersonation to simulate Argo CD controller permissions.
* Controller logs for detailed error messages (`argocd-application-controller`).

Commands:

```bash theme={null}
# Example: check whether the Argo CD application-controller can create Deployments
kubectl auth can-i create deploy --as=system:serviceaccount:argocd:argocd-application-controller

# View Argo CD application-controller logs for permission errors
kubectl logs -n argocd deploy/argocd-application-controller
```

Typical fixes:

* Grant missing permissions using `ClusterRole` / `ClusterRoleBinding` for cluster-scoped needs.
* Configure namespace-scoped `Role` / `RoleBinding` for limited privileges.
* Add required resource kinds to the ApplicationProject allow list if the project policy is blocking them.
* If CRDs cause permission or create errors, ensure CRDs are installed before Argo CD tries to manage CR instances.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Delivery-Troubleshooting-Drift-Permissions-and-Bad-Configs/argocd-permission-issues-fix-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=6137e14aceed9db9eaa5f3b241fbb95d" alt="The image illustrates permission issues related to ArgoCD, highlighting missing ClusterRole bindings, namespace-scoped limitations, and CRDs not being allowed, along with a suggested fix." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Delivery-Troubleshooting-Drift-Permissions-and-Bad-Configs/argocd-permission-issues-fix-diagram.jpg" />
</Frame>

Bad manifests and validation

* Symptoms: `SyncFailed` or resources stuck in Degraded/Progressing because YAML is invalid, fields have wrong types, CRDs are missing, or target namespaces don't exist.
* Best practice: validate manifests before they reach Argo CD (CI pre-merge checks).

Common issues to check:

* Invalid YAML (indentation, missing colons).
* Schema validation errors (wrong types or unknown fields).
* CRDs not installed in the cluster.
* Target namespaces don't exist or lack access.

Validation (server-side dry-run validates against the API server and catches schema / CRD / permission issues):

```bash theme={null}
# Dry-run apply against the cluster API server
kubectl apply --dry-run=server -f .

# Render Helm or Kustomize and validate against the cluster
helm template . | kubectl apply --dry-run=server -f -
kubectl kustomize . | kubectl apply --dry-run=server -f -
```

Use `argocd app diff` locally too:

```bash theme={null}
# Compare rendered local manifests with the live cluster before committing
argocd app diff my-app --local ./manifests
```

<Callout icon="lightbulb" color="#1CB2FE">
  Add `kubectl apply --dry-run=server` (or rendered Helm/Kustomize validation) to CI to block PRs that would fail validation in the cluster. This prevents the most common sync failures caused by invalid manifests or missing CRDs.
</Callout>

Additional troubleshooting tips

* Check controller logs and `kubectl describe` / `kubectl logs` for failing pods.
* For custom resource types, confirm CRDs exist and are compatible with the manifests being applied.
* Use Argo CD UI Diff/Sync history to see when drift started and what changed.
* When using operators, check operator documentation for which fields they manage so you can ignore them in Argo CD.

Summary — Five key points to remember

* Use `argocd app diff` to see exact differences between Git and the cluster.
* Use `ignoreDifferences` for fields intentionally managed by external controllers (e.g., HPA-driven replicas).
* Use `kubectl auth can-i` (impersonating the Argo CD service account) and `argocd-application-controller` logs to diagnose RBAC problems.
* Add `kubectl apply --dry-run=server` (or rendered Helm/Kustomize validation) to CI to catch schema/CRD/permission issues before merge.
* Most GitOps issues fall into drift, permissions, or bad configurations. Learning to diagnose each category reduces debugging time from hours to minutes.

Further reading and references

* Argo CD documentation: [https://argo-cd.readthedocs.io/](https://argo-cd.readthedocs.io/)
* Kubernetes RBAC: [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* Kubernetes API conventions (defaulting & server-side apply): [https://kubernetes.io/docs/reference/using-api/server-side-apply/](https://kubernetes.io/docs/reference/using-api/server-side-apply/)

This lesson concludes the module. You have learned GitOps troubleshooting techniques for repository design, configuration templating, Argo CD, and Kubernetes-native delivery troubleshooting.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/6a1ab174-fb6a-4448-8b53-689c6ea9fccc" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/9581df5b-9e92-49bc-87e8-2e256be4e549" />
</CardGroup>
