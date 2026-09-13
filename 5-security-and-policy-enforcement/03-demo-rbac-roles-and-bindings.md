> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo RBAC Roles and Bindings

> Guide to Kubernetes RBAC explaining Roles, ClusterRoles, RoleBindings, ClusterRoleBindings, service account bindings, testing permissions with kubectl auth can-i and debugging for least privilege

Everything in Kubernetes starts with one question: who is allowed to do what and where? Role-Based Access Control (RBAC) answers that question. It's a cornerstone of Kubernetes security: misconfigured RBAC can leave your cluster unusable or dangerously over-permissive.

The model is simple and powerful:

* Roles (or ClusterRoles) define permissions (verbs on resources).
* RoleBindings (or ClusterRoleBindings) attach those permissions to identities: users, groups, or service accounts.

In this guide we'll cover namespaced Roles and cluster-scoped ClusterRoles, show how to bind them to users and service accounts, and demonstrate how to test and debug permissions.

## Check namespaces and service accounts

Start by listing your namespaces and the service accounts in team namespaces:

```bash theme={null}
# List namespaces
kubectl get namespaces

# Example output
# NAME              STATUS   AGE
# default           Active   18h
# kube-flannel      Active   18h
# kube-node-lease   Active   18h
# kube-public       Active   18h
# kube-system       Active   18h
# team-alpha        Active   5m
# List service accounts in each team namespace
kubectl get sa -n team-alpha
kubectl get sa -n team-beta

# Example output
# team-alpha:
# NAME         AGE
# default      6m
# deploy-bot   6m
#
# team-beta:
# NAME         AGE
# default      6m
# readonly-sa  6m
```

## Developer Role (namespaced)

This Role grants CRUD + watch/list/get permissions for several common resources in the `team-alpha` namespace. It's scoped to the namespace so it won't affect other teams.

```yaml theme={null}
# developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: team-alpha
rules:
- apiGroups: [""]            # core API group
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]       # apps API group
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Apply the Role, then create a RoleBinding that grants this Role to a user named `alice` in the `team-alpha` namespace:

```bash theme={null}
# Apply the Role manifest
kubectl apply -f developer-role.yaml
# Expected output:
# Bind the role to a user 'alice' in the team-alpha namespace
kubectl create rolebinding alice-developer --role=developer --user=alice -n team-alpha
# Expected output:
# rolebinding.rbac.authorization.k8s.io/alice-developer created
```

Test Alice's permissions using `kubectl auth can-i` with impersonation via `--as`:

```bash theme={null}
# Impersonate alice and ask if she can create pods in team-alpha
kubectl auth can-i create pods --namespace=team-alpha --as=alice
# The same check in team-beta should be denied (no binding exists there)
kubectl auth can-i create pods --namespace=team-beta --as=alice
# Expected output: no
```

This enforces least privilege: Alice can create resources only in the namespace where the RoleBinding exists.

## Cluster-wide Viewer Role (ClusterRole)

To provide read-only access across the entire cluster use a ClusterRole and ClusterRoleBinding. The following ClusterRole allows `get`, `list`, and `watch` for common cluster resources:

```yaml theme={null}
# viewer-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: viewer
rules:
- apiGroups: [""]                 # core API group
  resources: ["pods", "services", "namespaces", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]             # apps API group
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
```

Apply and bind cluster-wide to a user `bob`:

```bash theme={null}
kubectl apply -f viewer-clusterrole.yaml
# Expected output:
kubectl create clusterrolebinding bob-viewer --clusterrole=viewer --user=bob
# Expected output:
# clusterrolebinding.rbac.authorization.k8s.io/bob-viewer created
```

Verify Bob's permissions (impersonating Bob):

```bash theme={null}
# Bob can list pods in both team-alpha and team-beta because viewer is cluster-scoped
kubectl auth can-i list pods --as=bob -n team-alpha
kubectl auth can-i list pods --as=bob -n team-beta
# But Bob cannot create pods because viewer is read-only
kubectl auth can-i create pods --as=bob -n team-beta
# Expected output: no
```

## Binding Roles to Service Accounts

Service accounts are identities used by automation, controllers, and CI/CD systems. Bind Roles to service accounts using the `--serviceaccount` flag; the format is `namespace:serviceaccount-name`.

<Callout icon="lightbulb" color="#1CB2FE">
  When impersonating a service account with `kubectl auth can-i`, use the username format:
  `system:serviceaccount:<namespace>:<serviceaccount-name>` — for example:
  `system:serviceaccount:team-alpha:deploy-bot`.
</Callout>

Bind the `developer` Role to the `deploy-bot` service account in `team-alpha`:

```bash theme={null}
kubectl create rolebinding deploy-bot-developer \
  --role=developer \
  --serviceaccount=team-alpha:deploy-bot \
  -n team-alpha
# Expected output:
# rolebinding.rbac.authorization.k8s.io/deploy-bot-developer created
```

Test the permissions as that service account:

```bash theme={null}
# Can the deploy-bot create deployments in team-alpha?
kubectl auth can-i create deployments \
  --as=system:serviceaccount:team-alpha:deploy-bot \
  -n team-alpha
# The readonly service account in team-beta should not be able to get pods (no binding).
kubectl auth can-i get pods \
  --as=system:serviceaccount:team-beta:readonly-sa \
  -n team-beta
# Expected output: no
```

## Default deny and RBAC debugging

Kubernetes RBAC is deny-by-default. If no binding grants a permission, it is denied — you do not need explicit deny rules.

To debug RBAC:

* Use `kubectl auth can-i` to check whether an identity can perform an action.
* Use `--as` to impersonate users or service accounts.
* For deeper debugging, increase verbosity on API calls (e.g., `kubectl --v=8`) or inspect Role/ClusterRole and RoleBinding/ClusterRoleBinding objects.

<Callout icon="warning" color="#FF6B6B">
  Always review Role and ClusterRole changes carefully. Test with `kubectl auth can-i --as=...` before applying changes to production to avoid accidentally granting too much access.
</Callout>

## Quick comparison

| Resource Type      | Scope          | Use case                                                   | Creation command example                        |
| ------------------ | -------------- | ---------------------------------------------------------- | ----------------------------------------------- |
| Role               | Namespaced     | Grant permissions within a single namespace                | `kubectl apply -f developer-role.yaml`          |
| RoleBinding        | Namespaced     | Attach a namespaced Role to a user/serviceaccount          | `kubectl create rolebinding ... -n <namespace>` |
| ClusterRole        | Cluster-scoped | Define cluster-wide or API-group scoped permissions        | `kubectl apply -f viewer-clusterrole.yaml`      |
| ClusterRoleBinding | Cluster-scoped | Attach a ClusterRole to a user/serviceaccount cluster-wide | `kubectl create clusterrolebinding ...`         |

## Summary / Best practices

* Roles and RoleBindings are namespaced; ClusterRoles and ClusterRoleBindings are cluster-scoped.
* Prefer namespaced Roles when access only needs to be limited to a team or project.
* Use `kubectl auth can-i` with `--as` to test permissions for users and service accounts before and after changes.
* Follow the principle of least privilege: grant only the permissions required.
* Remember: by default, Kubernetes denies actions that aren't explicitly granted.

## Links and references

* [Kubernetes RBAC docs](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* [kubectl auth can-i documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#auth)
* [Managing Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

Practice these principles in a safe lab environment by creating Roles, ClusterRoles, RoleBindings, and ClusterRoleBindings and validating access with `kubectl auth can-i`.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/cfeb58dd-916f-46fd-805a-ff58b8242fcc" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/cfac0a94-3720-4891-9aac-3f80c4454333" />
</CardGroup>
