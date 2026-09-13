> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Operators Controllers Reconcile Like a Pro

> Explains Kubernetes CRDs and controllers, the reconciliation loop, status and finalizers, and best practices for building reliable operators.

We can define platform APIs with CustomResourceDefinitions (CRDs) so users create custom resources (CRs). That validates YAML with the API server, but validation alone doesn't make things happen. If you create a Database CR today, nothing will be provisioned automatically — the YAML just sits in etcd until a controller acts on it.

Controllers are the missing piece that bridge declared intent to real-world resources. In this lesson we will:

* Explain why CRDs are useful only when paired with controllers.
* Describe the reconciliation loop and its governing principles.
* Describe controller architecture and the typical reconcile function shape.
* Cover status reporting with conditions and observedGeneration.
* Implement finalizers for cleanup on deletion.
* Compare common operator frameworks so you can pick the right one.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/crd-controller-architecture-learning-objectives.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=e4166ac19f928547972b24c406d2dc78" alt="The image outlines learning objectives related to CRDs and controller architecture, including understanding CRDs' need for controllers and explaining the reconciliation loop." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/crd-controller-architecture-learning-objectives.jpg" />
</Frame>

This is a longer lesson because controllers are one of the most fundamental concepts in Kubernetes. Deployments, StatefulSets, Services, and every custom operator all depend on controllers to make the desired state real.

Scenario: a shiny Database CRD

A platform team builds a Database CRD with enum validations, printer columns, and a solid OpenAPI schema. They tell developers: create a database with `kubectl apply`.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/platform-team-database-crd-interaction.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=c9979ac8e7db6d8ad8ed5155f755d144" alt="The image illustrates a concept where a platform team interacts with a database CRD (Custom Resource Definition) through processes like enum validation, printer columns, and schema management." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/platform-team-database-crd-interaction.jpg" />
</Frame>

The developer applies a YAML:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/platform-team-developers-kubectl-diagram.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=8223b07f24b55cad4e2e315199abfee2" alt="The image shows a diagram with a &#x22;Platform Team&#x22; informing &#x22;Developers&#x22; that they can now create databases using kubectl apply." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/platform-team-developers-kubectl-diagram.jpg" />
</Frame>

```yaml theme={null}
apiVersion: platform.acme.io/v1
kind: Database
metadata:
  name: orders-db
spec:
  size: medium
  engine: postgresql
```

Command:

```bash theme={null}
kubectl apply -f orders-db.yaml
# Output
database.platform.acme.io/orders-db created
```

It appears successful, but there is still no PostgreSQL instance, no network configuration, and no status updates. Without a controller, the CR is only persisted in etcd — nothing reconciles it to reality.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/crd-limitations-etcd-data-diagram.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=d4a1f3e98d598dde107249058a459956" alt="The image is a diagram highlighting the limitations of CRDs as etcd data, including issues like no database creation, lack of status updates, absence of drift correction, and no cleanup on delete." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/crd-limitations-etcd-data-diagram.jpg" />
</Frame>

Controller = the software that makes desired state real. A helpful analogy:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/kubernetes-roles-crd-cr-controller.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=f90c85f0e799f580329edd0739d5de98" alt="The image explains the roles of CRD (Menu), CR (Order), and Controller (Kitchen) in a Kubernetes context, using a restaurant analogy." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/kubernetes-roles-crd-cr-controller.jpg" />
</Frame>

* CRD is the menu: it defines what can be requested.
* CR (the object you apply) is the order you place.
* Controller is the kitchen: it actually prepares the meal and reports progress.

The Reconciliation Loop

Reconciliation is the core control logic controllers run repeatedly to converge actual state to desired state. Typical phases:

1. Watch / observe — get notifications for creates/updates/deletes.
2. Compare — compute desired state (from `spec`) vs actual state (external resources, cloud APIs, cluster objects).
3. Act — create/update/delete resources to close the gap.
4. Report — update the CR's `status` (conditions, observedGeneration, phase).

Examples for a Database CR:

* If the external DB does not exist, create it.
* If the `size` changed, resize the DB.
* If the CR is deleted, perform cleanup.
* Update `status` to show provisioning progress or readiness.

Follow three guiding principles for safe and robust reconcilers:

* Level-triggered: react to current state, not transient events. If an event is missed, the next reconcile still inspects state and corrects drift.
* Idempotency: running Reconcile multiple times should produce the same final result as running it once—always check current state first.
* Eventual consistency: the system can take multiple reconciles to converge; expect retries and backoff.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/reconciliation-loop-desired-actual-principles.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=99759a6537772900fee4a82752250e51" alt="The image outlines the key principles of &#x22;The Reconciliation Loop: Desired vs Actual,&#x22; emphasizing level-triggered processes, eventual consistency, and idempotent operations. It suggests reconciling based on current state rather than events, allowing systems to converge over time and handle repeated operations with consistent results." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/reconciliation-loop-desired-actual-principles.jpg" />
</Frame>

Use status to inform users: a five-minute provisioning operation can report `Progressing` while the controller continues to reconcile and later update `Ready`.

Controller internals: four collaborating layers

Most operator frameworks implement this architecture for you; your job is to implement the Reconciler (the business logic).

1. API server — source of truth and where desired state is stored.
2. Informer — watches API server and maintains a local cache to reduce API traffic.
3. Work queue — the informer enqueues resource keys; queue deduplicates events so the reconciler runs for the latest state.
4. Reconciler — your implementation: read resource, compare desired vs actual, act, and return a result (requeue/no-requeue/error).

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/controller-architecture-diagram-api-informer-workqueue.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=62cb548c85a4d6dcb5c8477647f19c6f" alt="The image is a diagram of a controller architecture consisting of three components: API Server, Informer, and Work Queue, each with specific roles described alongside." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/controller-architecture-diagram-api-informer-workqueue.jpg" />
</Frame>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/controller-architecture-api-server-informer.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=61aa21a8736e3f82f663da095c0858e2" alt="The image illustrates a controller architecture composed of four components: an API Server, an Informer, a Work Queue, and a Reconciler, each with specific functions outlined." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/controller-architecture-api-server-informer.jpg" />
</Frame>

The framework handles watching, queuing, retries, and worker scaling so you can focus on making the reconciler idempotent and correct.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/controller-architecture-resilience-consistency-scalability.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=fd3b3128e0d186602fffb8ab005266e0" alt="The image illustrates a &#x22;Controller Architecture&#x22; with focus on resilience, consistency, and scalability, explaining the benefits of the design. It highlights that the framework manages watching, queuing, and retries, while the user writes the reconciler." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/controller-architecture-resilience-consistency-scalability.jpg" />
</Frame>

Reconcile function pattern (conceptual Go example)

You don't need to memorize Go—focus on the flow. This example mirrors the signature used by controller-runtime: `Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)`.

```go theme={null}
// Reconcile compares desired and actual state and acts to converge them.
// Signature in controller-runtime: Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
func (r *DBReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the resource
    db := &platformv1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        if apierrors.IsNotFound(err) {
            // Resource deleted after request — nothing to do.
            return ctrl.Result{}, nil
        }
        // Transient error reading the object — requeue.
        return ctrl.Result{}, err
    }

    // 2. Check if being deleted (finalizer handling)
    if db.GetDeletionTimestamp() != nil {
        return r.handleDeletion(ctx, db)
    }

    // 3. Main reconcile logic (idempotent): ensure external DB exists and matches spec
    if err := r.ensureDatabase(ctx, db); err != nil {
        // If transient, requeue with backoff
        return ctrl.Result{}, err
    }

    // 4. Update status: observedGeneration, conditions, phase, etc.
    db.Status.ObservedGeneration = db.Generation
    db.Status.Phase = "Ready"
    // set conditions as appropriate...
    if err := r.Status().Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

Best practices for your reconcile function:

* Always fetch the latest object from the API server or cache.
* Handle deletion via finalizers and short-circuit if the object is gone.
* Make operations idempotent — check external state before making changes.
* Use the `status` subresource to update fields like `observedGeneration` and `conditions`.

Status and conditions

Status is how your controller reports progress and health. Kubernetes conditions are a well-established pattern: each condition includes `type`, `status` (`True`/`False`/`Unknown`), `reason`, and `message`.

Common condition types:

* `Available` / `Ready` — resource is ready to serve.
* `Progressing` — creation/update is in progress.
* `Degraded` — running but not operating correctly.

observedGeneration pattern

When a user modifies the `spec`, Kubernetes increments `metadata.generation`. Your controller sets `status.observedGeneration` to the generation it last reconciled. Comparing `metadata.generation` and `status.observedGeneration` tells users whether the controller has acted on the latest change.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/observedgeneration-metadata-status-comparison.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=cb3499a539291fcf0f3a83943fa5962d" alt="The image explains the &#x22;observedGeneration&#x22; pattern, where you compare metadata.generation with status.observedGeneration to check if changes are processed." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/observedgeneration-metadata-status-comparison.jpg" />
</Frame>

Finalizers and deletion flow

Finalizers allow controllers to perform cleanup before Kubernetes permanently removes a CR. When a user deletes a CR, the API server sets `deletionTimestamp` but keeps the object until finalizers are cleared—this is the controller's opportunity to delete external resources (cloud instances, secrets, networks).

Typical finalizer handling:

```go theme={null}
// This runs when the CR has a deletion timestamp set.
func (r *DBReconciler) handleDeletion(ctx context.Context, db *platformv1.Database) (ctrl.Result, error) {
    if hasFinalizer(db, dbFinalizer) {
        // Perform cleanup of external resources.
        if err := deleteRDSInstance(ctx, db.Spec.Name); err != nil {
            // Retry on transient errors; log and requeue.
            return ctrl.Result{}, err
        }
        if err := deleteSecrets(ctx, db.Namespace, db.Name); err != nil {
            return ctrl.Result{}, err
        }

        // Remove finalizer and update the object to allow deletion.
        removeFinalizer(db, dbFinalizer)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }
    // Nothing else to do; let API server delete the object.
    return ctrl.Result{}, nil
}
```

Deletion flow summary:

1. User runs `kubectl delete`.
2. API server sets `deletionTimestamp` and retains the object.
3. Controller observes the timestamp, performs cleanup, and removes finalizers.
4. When no finalizers remain, the API server deletes the object.

<Callout icon="warning" color="#FF6B6B">
  If cleanup fails (for example, cloud API outages), the object will remain in a Terminating state while finalizers persist. Implement robust retries, exponential backoff, and comprehensive logging so cleanup eventually completes and you avoid orphaned resources.
</Callout>

Operator frameworks

Most teams avoid writing controller boilerplate from scratch. Common frameworks let you focus on business logic:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/qieq8rdZL3ypr7MT/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/operator-frameworks-comparison-chart.jpg?fit=max&auto=format&n=qieq8rdZL3ypr7MT&q=85&s=ba7cc2ffb35431df0e74dcfa2f8d2ddc" alt="The image is a comparison chart of three operator frameworks: Kubebuilder, Operator SDK, and Metacontroller, highlighting their features and best use cases." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/operator-frameworks-comparison-chart.jpg" />
</Frame>

Below is a concise comparison to help you choose:

| Framework      | Language / Style    | Best for                                | Notes & Links                                                                                 |
| -------------- | ------------------- | --------------------------------------- | --------------------------------------------------------------------------------------------- |
| Kubebuilder    | Go (code-first)     | Production Go operators                 | `https://book.kubebuilder.io/` — scaffolds controllers, CRDs, webhooks, testing               |
| Operator SDK   | Go / Helm / Ansible | Multi-language teams, Red Hat ecosystem | Built on Kubebuilder; supports Helm & Ansible operators — `https://sdk.operatorframework.io/` |
| Metacontroller | Any (webhook)       | Lightweight, polyglot operators         | Implement reconcile as an HTTP server — `https://metacontroller.github.io/`                   |

Key takeaways

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/controllers-crds-reconciliation-status-finalizers.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=a117cf77f38342dd2b942056db9ef290" alt="The image outlines four key takeaways about controllers, CRDs, reconciliation, status, and finalizers in a technology context. Each point is numbered and highlighted with a colored icon." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/Operators-Controllers-Reconcile-Like-a-Pro/controllers-crds-reconciliation-status-finalizers.jpg" />
</Frame>

* CRDs without controllers are just stored data — controllers make the desired state become real.
* Design your Reconcile to be idempotent and level-triggered: always read current state and act only when needed.
* Use `status`, conditions, and `observedGeneration` to transparently communicate what the controller has processed.
* Implement finalizers to clean up external resources; handle errors with retries and backoff to avoid stuck Terminating objects.

Additional resources

* Kubernetes controllers & reconciliation concepts: [https://kubernetes.io/docs/concepts/architecture/controller/](https://kubernetes.io/docs/concepts/architecture/controller/)
* controller-runtime (Reconcile signature and helpers): [https://github.com/kubernetes-sigs/controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)
* Kubebuilder book: [https://book.kubebuilder.io/](https://book.kubebuilder.io/)
* Operator SDK docs: [https://sdk.operatorframework.io/](https://sdk.operatorframework.io/)
* Metacontroller: [https://metacontroller.github.io/](https://metacontroller.github.io/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/756ffaae-767b-4743-9724-c05d3fbf9a18/lesson/4c8d6b72-361f-4c1c-8b62-cf84a251000f" />
</CardGroup>
