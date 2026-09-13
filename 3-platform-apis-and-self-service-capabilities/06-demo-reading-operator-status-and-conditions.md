> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Reading Operator Status and Conditions

> Guide showing how to read operator status and conditions with cert-manager, covering CRDs, Issuer and Certificate creation, status messages, Secrets, and reconciliation for troubleshooting

CRDs (CustomResourceDefinitions) define the API for custom resources, but an operator is the component that acts on those resources. An operator is a controller: it continuously watches custom resources and reconciles the cluster state to match the desired state declared in those resources. For example, you create a `Certificate` resource and the operator provisions the actual TLS certificate; delete the `Certificate` and the operator will clean up or re-create associated objects as needed.

This lesson walks through how to read operator status and conditions using cert-manager as a concrete example. In both exams and production troubleshooting, you often need to infer what an operator is doing without access to its source code. cert-manager exposes several CRDs that map to TLS lifecycle concepts (Certificate, Issuer, CertificateRequest, Order, Challenge, etc.). The operator watches these resources and reconciles them.

## Verify cert-manager CRDs exist

First, confirm that cert-manager's CRDs are installed:

```bash theme={null}
kubectl get crds | grep cert-manager
```

Example output:

```bash theme={null}
certificaterequests.cert-manager.io      2026-04-11T09:47:38Z
certificates.cert-manager.io             2026-04-11T09:47:38Z
challenges.acme.cert-manager.io          2026-04-11T09:47:38Z
clusterissuers.cert-manager.io           2026-04-11T09:47:38Z
issuers.cert-manager.io                  2026-04-11T09:47:38Z
orders.acme.cert-manager.io              2026-04-11T09:47:38Z
```

### Quick reference: cert-manager CRDs and their roles

|               Resource | Use case                                                          | Example              |
| ---------------------: | ----------------------------------------------------------------- | -------------------- |
|            Certificate | Declares the desired certificate (subject, DNS names, secretName) | `Certificate`        |
| Issuer / ClusterIssuer | Defines how certificates are obtained (ACME, selfSigned, CA)      | `Issuer`             |
|     CertificateRequest | Temporary resource used while issuing a certificate               | `CertificateRequest` |
|      Order / Challenge | ACME-specific resources used during domain validation             | `Order`, `Challenge` |

## Create a test Issuer (self-signed)

To issue a certificate you need two things: an Issuer that defines how the certificate will be issued and a Certificate resource that declares what you want. For quick testing, a self-signed Issuer is convenient.

Save this manifest as `/root/selfsigned-issuer.yaml`:

```yaml theme={null}
# /root/selfsigned-issuer.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: operator-lab
spec:
  selfSigned: {}
```

Apply the Issuer and inspect its status:

```bash theme={null}
kubectl apply -f /root/selfsigned-issuer.yaml
kubectl describe issuer selfsigned-issuer -n operator-lab
```

Example output from creating and describing the Issuer:

```bash theme={null}
issuer.cert-manager.io/selfsigned-issuer created
```

```text theme={null}
Name:             selfsigned-issuer
Namespace:        operator-lab
API Version:      cert-manager.io/v1
Kind:             Issuer
Spec:
  Self Signed:    true
Status:
  Conditions:
    Last Transition Time:  2026-04-11T10:12:02Z
    Observed Generation:   1
    Reason:                IsReady
    Status:                True
  Type:                    Ready
Events:                   <none>
```

Notice the `Status` block and the `Conditions` entry reporting `Ready: True`. This indicates the operator successfully reconciled the Issuer.

## Create a Certificate that uses the Issuer

Next, create a `Certificate` that references the self-signed Issuer. Save this as `/root/test-certificate.yaml`:

```yaml theme={null}
# /root/test-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-cert
  namespace: operator-lab
spec:
  secretName: test-cert-tls
  duration: 2160h       # 90 days
  renewBefore: 360h     # 15 days before expiry
  subject:
    organizations:
      - Example Corp
  commonName: test.example.com
  dnsNames:
    - test.example.com
    - www.test.example.com
  issuerRef:
    name: selfsigned-issuer
    kind: Issuer
```

Apply and check status:

```bash theme={null}
kubectl apply -f /root/test-certificate.yaml
kubectl get certificate -n operator-lab
kubectl describe certificate test-cert -n operator-lab
```

Expected results after applying:

```bash theme={null}
certificate.cert-manager.io/test-cert created
```

```bash theme={null}
kubectl get certificate -n operator-lab
NAME        READY   SECRET          AGE
test-cert   True    test-cert-tls   18s
```

Describing the Certificate reveals status conditions, validity times, renewal time, and related events:

```bash theme={null}
kubectl describe certificate test-cert -n operator-lab
```

Example excerpt:

```text theme={null}
Status:
  Conditions:
    Last Transition Time:  2026-04-11T00:13:33Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
  Not After:              2026-04-11T00:13:33Z
  Not Before:             2026-04-11T00:13:33Z
  Renewal Time:           2026-06-25T10:13:33Z
  Revision:               1

Events:
  Normal  Issuing     65s  cert-manager-certificates-trigger         Issuing certificate as Secret does not exist
  Normal  Generated   65s  cert-manager-certificates-key-manager     Stored new private key in temporary Secret resource "test-cert-ktrpn"
  Normal  Issuing     64s  cert-manager-certificates-request-manager Created new CertificateRequest resource "test-cert-1"
  Normal  Issuing     63s  cert-manager-certificates-issuing         The certificate has been successfully issued
```

The `Message` and `Reason` fields provide the operator’s human-readable summary of the resource state. When troubleshooting, these are your primary starting points.

## Inspect the Secret created by the operator

cert-manager stores the issued certificate and private key in a Kubernetes Secret named as specified by `secretName` in the Certificate resource.

Check the Secret:

```bash theme={null}
kubectl get secret -n operator-lab
kubectl describe secret test-cert-tls -n operator-lab
```

Example output showing a TLS Secret with three keys:

```text theme={null}
NAME            TYPE                DATA  AGE
test-cert-tls   kubernetes.io/tls   3     113s
```

```text theme={null}
Type:  kubernetes.io/tls

Data
====
ca.crt:  1188 bytes
tls.crt: 1188 bytes
tls.key: 1675 bytes
```

These artifacts were created by cert-manager as a result of reconciling the `Certificate` resource.

## Demonstrate the reconciliation loop

To see reconciliation in action, delete the Secret and watch the operator recreate it:

```bash theme={null}
kubectl delete secret test-cert-tls -n operator-lab
kubectl get secret -n operator-lab
kubectl describe secret test-cert-tls -n operator-lab
```

Example interaction:

```bash theme={null}
secret "test-cert-tls" deleted from operator-lab

# A short time later the controller recreates it:
kubectl get secret -n operator-lab
NAME            TYPE                DATA  AGE
test-cert-tls   kubernetes.io/tls   3     14s
```

Operators continuously watch their owned resources and reconcile until the observed state matches the desired state. Deleting side-effect resources (for example, Secrets or ConfigMaps created by an operator) does not permanently break the system — the operator will typically restore them.

<Callout icon="lightbulb" color="#1CB2FE">
  Always start troubleshooting custom resources by inspecting their `status.conditions`. Use `kubectl describe <kind> <name> -n <namespace>` (or `kubectl get <kind> <name> -n <namespace> -o yaml`) to see the operator-reported `Status`, `Reason`, human-readable `Message`, and related events.
</Callout>

## Key takeaways (exam & production)

1. Use `kubectl get` and `kubectl describe` on custom resources to read their status and conditions. The operator reports its view through the resource's `status`.
2. Operators continuously reconcile and will restore resources they manage. Deleting dependent resources (e.g., Secrets created by an operator) may result in them being recreated automatically.
3. If a resource is not in the desired state, `status.conditions` usually contains a `False` condition with a `Reason` and `Message` that help diagnose what went wrong. Always check events shown by `kubectl describe` as well.

## Links and references

* cert-manager documentation: [https://cert-manager.io/docs/](https://cert-manager.io/docs/)
* Kubernetes documentation: [https://kubernetes.io/docs/](https://kubernetes.io/docs/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/756ffaae-767b-4743-9724-c05d3fbf9a18/lesson/33c8c726-73a9-4e4f-9b19-f426aa8cc191" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/756ffaae-767b-4743-9724-c05d3fbf9a18/lesson/208267db-ac21-46da-a59d-a686feee49f7" />
</CardGroup>
