> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Admission Webhooks in Action

> Demonstrates Kubernetes admission controllers showing how LimitRanger injects default resources and ResourceQuota enforces namespace quotas and their interaction using a demo

Authentication tells Kubernetes who you are.\
Authorization tells Kubernetes what you are allowed to do.

There is a third gate before a request is accepted: admission control. Even after successful authentication and authorization, requests must pass through admission controllers. This is where much platform governance lives — injecting defaults, enforcing quotas, validating configuration, mutating objects, and denying non-compliant requests.

<Callout icon="lightbulb" color="#1CB2FE">
  Admission controllers run after authentication and authorization but before objects are persisted to etcd. Some admission controllers mutate objects (mutating admission controllers), while others only allow or deny requests (validating admission controllers).
</Callout>

## Which admission plugins are active?

On many clusters the kube-apiserver runs as a pod. Inspect the kube-apiserver help output to see admission-related flags and which plugins are enabled. For example:

```bash theme={null}
kubectl exec -n kube-system kube-apiserver-controlplane \
  -- kube-apiserver -h 2>&1 | grep enable-admission | head -5
```

You may see both the deprecated `--admission-control` help text and the newer `--enable-admission-plugins` output. The important point for this demo: LimitRanger (a mutating admission controller) and ResourceQuota (a validating admission controller) are enabled on the cluster.

Note: `--admission-control` is deprecated in favor of `--enable-admission-plugins` / `--disable-admission-plugins`. See the Kubernetes API server docs for details:

* [https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)

## LimitRanger (mutating) — injecting defaults

LimitRanger is a mutating admission controller enabled by default on many clusters. It can inject default resource requests and limits for containers based on a LimitRange object in the namespace.

Inspect the LimitRange in the demo namespace:

```bash theme={null}
kubectl describe limitrange default-limits -n admission-lab
```

Example output (abridged):

```plaintext theme={null}
Name:         default-limits
Namespace:    admission-lab
Type          Resource    Min      Max     Default Request    Default Limit
Container     cpu         100m     500m    100m               500m
Container     memory      128Mi    512Mi   128Mi              512Mi
```

This LimitRange indicates:

* If a container is created without resource *requests*, it will get a default `cpu: 100m` and `memory: 128Mi`.
* If a container is created without resource *limits*, it will get a default `cpu: 500m` and `memory: 512Mi`.

Create a Pod manifest that has no resource requests or limits (file: `pod-no-resources.yaml`):

```yaml theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: admission-lab
spec:
  containers:
  - name: app
    image: nginx
    # no resources specified here
```

Apply the Pod:

```bash theme={null}
kubectl apply -f pod-no-resources.yaml
```

Describe the created Pod to see the injected defaults:

```bash theme={null}
kubectl describe pod test-pod -n admission-lab
```

Excerpt of the container section (abridged):

```plaintext theme={null}
Containers:
  app:
    Image: nginx
    State: Running
    Limits:
      cpu: 500m
      memory: 512Mi
    Requests:
      cpu: 100m
      memory: 128Mi
```

Even though these values were not present in the manifest, LimitRanger (a mutating admission controller) injected the defaults before the Pod object was persisted.

## ResourceQuota (validating) — enforcing namespace resource usage

ResourceQuota is a validating admission controller: it denies requests that would exceed the hard limits set for a namespace. It does not mutate objects; it only accepts or rejects them.

Inspect the ResourceQuota in the namespace:

```bash theme={null}
kubectl describe resourcequota pod-quota -n admission-lab
```

Example output:

```plaintext theme={null}
Name:       pod-quota
Namespace:  admission-lab
Resource    Used  Hard
pods        1     4
```

This quota restricts the namespace to at most 4 Pods. ResourceQuota must be able to account for concrete resource values; when a Pod has no explicit requests/limits, LimitRanger provides those defaults so ResourceQuota can count the resources and decide whether to admit the request.

## How LimitRange and ResourceQuota work together

* LimitRange (mutating): injects default resource requests/limits at admission time when an object is created or recreated.
* ResourceQuota (validating): counts the (possibly mutated) resources and rejects requests that would exceed the hard limits.

Table summary:

| Controller    | Type       | Behavior                                                                      | Example use                                                        |
| ------------- | ---------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| LimitRanger   | Mutating   | Injects default `resources.requests` and `resources.limits` at admission time | `LimitRange` that sets default CPU/memory per container            |
| ResourceQuota | Validating | Counts resources in namespace and denies requests that exceed `hard` limits   | `ResourceQuota` restricting number of pods or aggregate CPU/memory |

## Demo: quota-buster deployment (5 replicas in namespace with pod quota 4)

Create a Deployment manifest `quota-buster.yaml` that requests 5 replicas:

```yaml theme={null}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-buster
  namespace: admission-lab
spec:
  replicas: 5
  selector:
    matchLabels:
      app: quota-buster
  template:
    metadata:
      labels:
        app: quota-buster
    spec:
      containers:
      - name: app
        image: nginx
        # no resources specified; LimitRange will inject defaults
```

Apply the Deployment:

```bash theme={null}
kubectl apply -f quota-buster.yaml -n admission-lab
```

Check pods and the Deployment status:

```bash theme={null}
kubectl get pods -n admission-lab
kubectl get deploy quota-buster -n admission-lab
```

Because the namespace `admission-lab` has a hard pod quota of 4, only four Pods can be admitted. The ReplicaSet/Deployment controller will attempt to create the fifth Pod, but ResourceQuota (validating) will cause that Pod creation to be rejected. The Deployment will end up in a state such as "4/5 ready" (4 available, 5 desired), illustrating the interaction between mutating defaults and validating quotas.

## Important behavior note

* LimitRange defaults are applied only at admission time when an object is created or recreated. Updating a LimitRange later does not retroactively change resource requests/limits of already running Pods. Only new Pods or Pods that are deleted and recreated will pick up the new defaults.

<Callout icon="warning" color="#FF6B6B">
  Admission controllers run at admission time. Mutating controllers change the object before it’s stored; validating controllers accept or reject the request. Changes to defaults (e.g., a LimitRange) do not mutate already existing objects.
</Callout>

That was a concise overview of mutating vs validating admission controllers and how they interact (LimitRanger + ResourceQuota). Try these steps in a test cluster to observe the behavior firsthand.

## Links and References

* [Kubernetes Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
* [LimitRange documentation](https://kubernetes.io/docs/concepts/policy/limit-range/)
* [ResourceQuota documentation](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/d53940b9-5f20-45c2-8878-1e625bc1e193" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/b2db939b-d7a6-4431-9c87-3ba55a87f53d" />
</CardGroup>
