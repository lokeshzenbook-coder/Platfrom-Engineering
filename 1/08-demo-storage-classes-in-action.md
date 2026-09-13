> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Storage Classes in Action

> Guide to Kubernetes StorageClasses and dynamic volume provisioning, demonstrating PVC/PV lifecycle, reclaim policies Delete versus Retain, WaitForFirstConsumer behavior, and production best practices.

Every application that stores data needs storage. On Kubernetes, workloads request storage via PersistentVolumeClaims (PVCs), and the cluster provisions storage according to a StorageClass. Think of a StorageClass as a platform-provided menu of storage options: fast SSDs for databases, cheap spinning disks for archival data, cloud block volumes, or replicated storage for production workloads. Teams request the option they need and Kubernetes handles provisioning.

This guide shows how to:

* List and inspect StorageClasses
* Demonstrate dynamic provisioning using a StorageClass (`fast`)
* Compare reclaim behaviors (`Delete` vs `Retain`)
* Apply best practices for production clusters

Keywords: Kubernetes StorageClass, dynamic provisioning, reclaim policy, PVC, PV, WaitForFirstConsumer

## List available StorageClasses

Get the StorageClasses in the cluster:

```bash theme={null}
kubectl get sc
```

Example output:

```text theme={null}
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE        ALLOWVOLUMEEXPANSION   AGE
archive      rancher.io/local-path   Retain          WaitForFirstConsumer     false                  16m
fast         rancher.io/local-path   Delete          WaitForFirstConsumer     false                  16m
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer     false                  16m
```

Note: The cluster’s default StorageClass (if set) is indicated by an annotation/marker in some `kubectl get sc` output. If a PVC omits `storageClassName`, Kubernetes uses the default StorageClass automatically (if one exists).

<Callout icon="lightbulb" color="#1CB2FE">
  If a PVC does not set `storageClassName`, Kubernetes assigns the default StorageClass (if one exists). This behavior is important in production clusters to avoid unexpected storage types being provisioned.
</Callout>

## Inspect a StorageClass

To view details for the `fast` StorageClass:

```bash theme={null}
kubectl describe storageclass fast
```

Example output (trimmed):

```text theme={null}
Name:                  fast
IsDefaultClass:        No
Provisioner:           rancher.io/local-path
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
AllowVolumeExpansion:  <unset>
Parameters:            <none>
MountOptions:          <none>
Events:                <none>
```

Key StorageClass fields and what they mean:

| Field                  | Description                                                       | Typical Values / Notes                                                    |
| ---------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `Provisioner`          | The CSI/plugin responsible for creating volumes                   | e.g., `rancher.io/local-path`, `kubernetes.io/aws-ebs`, `ebs.csi.aws.com` |
| `ReclaimPolicy`        | What happens to the underlying PV/storage when the PVC is deleted | `Delete` (auto-remove), `Retain` (manual cleanup)                         |
| `VolumeBindingMode`    | When the PV is provisioned relative to Pod scheduling             | `Immediate` or `WaitForFirstConsumer` (prevents cross-zone leaks)         |
| `AllowVolumeExpansion` | Whether PVCs can request more capacity after creation             | `true` / `false`                                                          |
| `Parameters`           | Provisioner-specific options                                      | Varies per driver                                                         |

Volume binding mode note: `WaitForFirstConsumer` delays provisioning until a Pod using the PVC is scheduled — this avoids provisioning in the wrong zone/node and is highly recommended in multi-zone clusters.

## Demonstrate dynamic provisioning (fast StorageClass)

Create a PVC that requests 1Gi using the `fast` StorageClass.

Save this as `pvc-fast.yaml`:

```yaml theme={null}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage
  namespace: storage
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 1Gi
```

Apply the PVC:

```bash theme={null}
kubectl apply -f pvc-fast.yaml
# persistentvolumeclaim/app-storage created
```

Watch the PVC in the `storage` namespace:

```bash theme={null}
kubectl get pvc -n storage -w
```

Because `VolumeBindingMode` is `WaitForFirstConsumer`, the PVC will initially be `Pending` until a Pod mounts it. Create a Pod that consumes the PVC to trigger provisioning:

```bash theme={null}
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: app-storage
EOF
# pod/app created
```

After scheduling, the PVC should transition from `Pending` to `Bound`:

```text theme={null}
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
app-storage   Bound    pvc-1dcc29bc-a8a1-4c97-a5f9-bb0e19dadc7   1Gi        RWO            fast           30s
```

A PV is created dynamically to satisfy the claim:

```bash theme={null}
kubectl get pv
```

Example:

```text theme={null}
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STORAGECLASS   STATUS   CLAIM
pvc-1dcc29bc-a8a1-4c97-a5f9-bb0e19dadc7    1Gi        RWO            Delete           fast           Bound    storage/app-storage
```

Explanation: Because the `fast` StorageClass has `ReclaimPolicy: Delete`, the underlying storage will be deleted automatically when the PVC (and its binding) is removed.

## Demonstrate reclaim policy difference: Retain (archive StorageClass)

The `archive` StorageClass in this cluster uses `ReclaimPolicy: Retain`. Create a PVC using that class.

Save this as `pvc-archive.yaml`:

```yaml theme={null}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: archive-storage
  namespace: storage
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: archive
  resources:
    requests:
      storage: 500Mi
```

Apply it:

```bash theme={null}
kubectl apply -f pvc-archive.yaml
# persistentvolumeclaim/archive-storage created
```

Create a Pod that mounts the `archive-storage` claim:

```bash theme={null}
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: archive-app
  namespace: storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: archive-storage
EOF
# pod/archive-app created
```

After scheduling, the PVC will bind and a PV will be provisioned for the `archive` StorageClass. The PV will show `RECLAIM POLICY: Retain`:

```bash theme={null}
kubectl get pv
```

Example (trimmed):

```text theme={null}
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STORAGECLASS   STATUS   CLAIM
pvc-aaaa1111-...                           500Mi      RWO            Retain           archive        Bound    storage/archive-storage
```

## What happens on deletion with different reclaim policies

1. For fast (ReclaimPolicy: Delete)
   * Delete the Pod and the PVC:
     ```bash theme={null}
     kubectl delete pod -n storage app
     kubectl delete pvc -n storage app-storage
     ```
   * Result: The PV and the underlying storage resource are deleted automatically.

2. For archive (ReclaimPolicy: Retain)
   * Delete the Pod, then delete the PVC:
     ```bash theme={null}
     kubectl delete pod -n storage archive-app
     kubectl delete pvc -n storage archive-storage
     ```
   * Result: The PV remains in the cluster and moves to `Released` state. The underlying data is preserved and requires administrator action to reclaim or reuse the volume.

Example after deleting a PVC for a `Retain` PV:

```bash theme={null}
kubectl get pv
```

Example output:

```text theme={null}
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STORAGECLASS   STATUS     CLAIM
pvc-aaaa1111-...                           500Mi      RWO            Retain           archive        Released   <none>
```

When a PV is `Released` with `Retain`, an administrator must:

* Inspect and back up data if needed
* Clean or wipe data to make the volume reusable
* Remove or update `claimRef` on the PV to allow re-binding
* Or manually delete the underlying storage resource

<Callout icon="lightbulb" color="#1CB2FE">
  Choose reclaim policies based on workload needs:

  * Use `Retain` for critical, stateful workloads (databases, logs) to prevent accidental data loss.
  * Use `Delete` for ephemeral or test workloads to automate cleanup.
    Also confirm correct access modes (`ReadWriteOnce`, `ReadWriteMany`) and zone affinity by using `WaitForFirstConsumer` in multi-zone clusters.
</Callout>

## Recap / Best practices

* Inspect StorageClasses before creating PVCs:
  * `kubectl get sc` and `kubectl describe sc <name>`
* Understand reclaim behavior:
  * `Delete` — automated cleanup
  * `Retain` — manual intervention required
* Use `WaitForFirstConsumer` to avoid cross-zone provisioning in multi-zone clusters.
* Prefer explicitly setting `storageClassName` in PVCs unless you intentionally rely on a default StorageClass.
* Confirm access modes and size requirements match application needs.
* Monitor resource lifecycle:
  * `kubectl get pvc -n <ns> -w`
  * `kubectl get pv`

## Links and references

* [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
* [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [Persistent Volume Claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)

Explore your cluster’s StorageClasses, create PVCs and Pods, and observe how Kubernetes dynamically provisions and manages storage according to StorageClass settings.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/d09019f0-e51f-485c-aa53-736128144afd" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/989346de-0207-4837-af11-bf456d188972/lesson/b55e9c11-0076-440f-b2bb-735326813468" />
</CardGroup>
