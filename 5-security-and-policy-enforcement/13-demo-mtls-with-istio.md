> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo mTLS with Istio

> Guide demonstrating Istio mTLS setup in Kubernetes, enabling sidecar injection, enforcing strict mutual TLS, applying AuthorizationPolicy to restrict service access, and validating with istioctl and in-cluster tests

RBAC controls who can interact with the Kubernetes API. But intra-cluster service-to-service traffic is unauthenticated and unencrypted by default: a compromised pod can communicate freely with other services. Mutual TLS (mTLS) provides both encryption in transit and identity verification between workloads. Istio implements mTLS transparently via sidecar proxies so your application code does not need to change.

In this guide you'll:

* Verify cluster and namespace state,
* Enable Istio automatic sidecar injection for the demo namespace,
* Observe default mTLS behavior (PERMISSIVE),
* Enforce STRICT mTLS with a PeerAuthentication resource,
* Apply an AuthorizationPolicy to restrict which services may call the payment API,
* Validate the policy via in-cluster curl requests,
* Run `istioctl analyze` for a final diagnostic check.

***

## 1) Check current state

Confirm Istio control plane pods and application pods in the `mesh-demo` namespace. Before enabling injection, application pods run without the sidecar (single container).

Check Istio control plane pods:

```bash theme={null}
kubectl get pods -n istio-system
```

Example output:

```text theme={null}
NAME                                   READY   STATUS    RESTARTS   AGE
istio-egressgateway-8455f4dc86-9xm62   1/1     Running   0          10m
istio-ingressgateway-767bc4085b-t6x9k  1/1     Running   0          10m
istiod-659d96fcd-xrr5n                 1/1     Running   0          12m
```

Check application pods in the demo namespace (no sidecars yet):

```bash theme={null}
kubectl get pods -n mesh-demo
```

Example output:

```text theme={null}
NAME                           READY   STATUS    RESTARTS   AGE
order-api-764fd948dc-7xztj     1/1     Running   0          5m
payment-api-756c646c67-dxmx2   1/1     Running   0          5m
web-frontend-8586897ccf-xvbfm  1/1     Running   0          5m
```

Run an initial analysis (recommended):

```bash theme={null}
istioctl analyze -n mesh-demo
```

Example informational output:

```text theme={null}
Info [IST0102] (Namespace mesh-demo) The namespace is not enabled for Istio injection. Run 'kubectl label namespace mesh-demo istio-injection=enabled' to enable it.
Info [IST0118] (Service mesh-demo/order-api) Port name (port: 80, targetPort: 80) doesn't follow Istio's port naming convention.
...
```

***

## 2) Enable automatic sidecar injection and restart workloads

Label the namespace to enable automatic Istio sidecar injection. Then restart the deployments so new pods are created with the sidecar proxy.

```bash theme={null}
kubectl label namespace mesh-demo istio-injection=enabled
kubectl rollout restart deployment/web-frontend deployment/payment-api deployment/order-api -n mesh-demo
```

After pods restart, each workload should have two containers (application + `istio-proxy`). Verify:

```bash theme={null}
kubectl get pods -n mesh-demo
```

Example output after restart:

```text theme={null}
NAME                                READY   STATUS    RESTARTS   AGE
order-api-764fd948dc-7xztj          2/2     Running   0          1m
payment-api-756c646c67-dxmx2        2/2     Running   0          1m
web-frontend-8586897ccf-xvbfm       2/2     Running   0          1m
```

Inspect a pod to see the effective mTLS mode:

```bash theme={null}
istioctl describe pod -n mesh-demo web-frontend-8586897ccf-xvbfm
```

Look for:

```bash theme={null}
Effective PeerAuthentication:
  Workload mTLS mode: PERMISSIVE
```

<Callout icon="lightbulb" color="#1CB2FE">
  Permissive mTLS accepts both plain-text (HTTP) and mTLS connections. It's a safe default during setup because it allows mixed clients while sidecars are being rolled out, but it does not enforce encryption or strongly verify caller identity.
</Callout>

***

## 3) Enforce strict mTLS (PeerAuthentication)

To require encrypted, authenticated connections between workloads in `mesh-demo`, create a PeerAuthentication resource with `mtls.mode: STRICT`.

peer-auth-strict.yaml:

```yaml theme={null}
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: mesh-demo
spec:
  mtls:
    mode: STRICT
```

Apply it:

```bash theme={null}
kubectl apply -f peer-auth-strict.yaml
```

Verify:

```bash theme={null}
kubectl describe peerauthentications.security.istio.io default -n mesh-demo
```

Expected spec summary:

```YAML theme={null}
Spec:
  mtls:
    Mode: STRICT
```

<Callout icon="warning" color="#FF6B6B">
  Applying `PeerAuthentication` with `mode: STRICT` will cause any plain-text (non-mTLS) connections to fail. Ensure all workloads in the namespace have sidecars injected and are up-to-date before applying strict mTLS to avoid service disruption.
</Callout>

***

## 4) Restrict calls with an AuthorizationPolicy

mTLS ensures encrypted, authenticated connections, but it does not restrict which workloads can connect to others. Use an AuthorizationPolicy to enforce a zero-trust pattern: allow only specific principals (service accounts) to call the payment API.

Create this AuthorizationPolicy scoped to the `payment-api` workload:

authz-policy.yaml:

```yaml theme={null}
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-api-policy
  namespace: mesh-demo
spec:
  selector:
    matchLabels:
      app: payment-api
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/mesh-demo/sa/web-frontend
      to:
        - operation:
            methods: ["GET", "POST"]
```

Key notes:

* `selector.matchLabels` scopes the policy to pods labeled `app: payment-api`.
* `principals` uses the SPIFFE identity form (here: `cluster.local/ns/mesh-demo/sa/web-frontend`) derived from the caller’s service account.
* `operation.methods` limits allowed HTTP methods. You can also add `paths` or other HTTP attributes.

Apply the policy:

```bash theme={null}
kubectl apply -f authz-policy.yaml
```

Table: Resources created

| Resource Type       | Purpose                                       | Example                 |
| ------------------- | --------------------------------------------- | ----------------------- |
| PeerAuthentication  | Enforce namespace/workload mTLS mode          | `peer-auth-strict.yaml` |
| AuthorizationPolicy | Allow specific principals to call payment-api | `authz-policy.yaml`     |

***

## 5) Validate behavior with in-cluster curl

Test from a pod with an unauthorized identity (order-api). The request should be rejected by Istio (HTTP 403):

```bash theme={null}
kubectl exec -n mesh-demo deployment/order-api -- \
  curl -s -o /dev/null -w "%{http_code}" http://payment-api.mesh-demo.svc.cluster.local/api/pay
```

Expected output:

```text theme={null}
403
```

Test from the allowed principal (web-frontend). The request should be permitted; the upstream service may return `404` if the path is missing, which still indicates the request was authorized and reached the service:

```bash theme={null}
kubectl exec -n mesh-demo deployment/web-frontend -- \
  curl -s -o /dev/null -w "%{http_code}" http://payment-api.mesh-demo.svc.cluster.local/api/pay
```

Expected output (example):

```text theme={null}
404
```

Interpretation:

* `403` from `order-api`: Istio denied the request at the sidecar (authorization failure).
* `404` from `web-frontend`: Policy allowed the request and the call reached the payment-api, but the specific path was not found.

This demonstrates zero-trust networking: connections are both authenticated (mTLS) and authorized (AuthorizationPolicy).

***

## 6) Re-run analysis

Run `istioctl analyze` to surface configuration issues such as namespace injection, port naming, selector mismatches, or other diagnostic hints:

```bash theme={null}
istioctl analyze -n mesh-demo
```

Example informational warnings:

```text theme={null}
Info [IST0118] (Service mesh-demo/payment-api) Port name (port: 80, targetPort: 80) doesn't follow Istio's port naming convention.
Info [IST0118] (Service mesh-demo/web-frontend) Port name (port: 80, targetPort: 80) doesn't follow Istio's port naming convention.
```

Use `istioctl analyze` as a routine pre-deployment check before pushing broader changes to production.

***

Summary

* Enabled Istio sidecar injection for the `mesh-demo` namespace.
* Confirmed PERMISSIVE mTLS initially, then enforced STRICT mTLS using a PeerAuthentication.
* Applied an AuthorizationPolicy to restrict which service account can call the payment API and limited allowed operations.
* Validated enforcement with in-cluster curl tests: unauthorized requests were rejected by Istio, authorized requests reached the service.
* Re-ran `istioctl analyze` to surface configuration warnings to address.

Links and references

* [Istio PeerAuthentication docs](https://istio.io/latest/docs/reference/config/security/peer_authentication/)
* [Istio AuthorizationPolicy docs](https://istio.io/latest/docs/reference/config/security/authorization-policy/)
* [istioctl analyze](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl-analyze/)
* [SPIFFE and SPIRE](https://spiffe.io/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/64424caf-3564-4c76-93f1-0d88101072d6" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/caedb06f-ceb3-4c51-9c37-be0a7855e26b" />
</CardGroup>
