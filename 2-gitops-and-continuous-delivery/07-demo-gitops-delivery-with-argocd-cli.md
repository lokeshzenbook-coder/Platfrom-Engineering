> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo GitOps Delivery with ArgoCD CLI

> Demonstrates using the Argo CD CLI to create and sync Git based applications, enable automated sync with self‑heal and auto‑prune, and validate automatic reconciliation

This lesson demonstrates how to perform Argo CD application lifecycle operations using the Argo CD CLI instead of the web UI. You'll learn how to:

* Log in to the Argo CD server from the CLI
* Create an application from a Git repository
* Perform a manual sync and inspect status
* Enable automated sync, self-heal, and auto-prune
* Verify self-healing by making an intentional out-of-band change

This is a lab environment: TLS is not configured and plaintext credentials are used for demonstration only. Do not use these options in production.

<Callout icon="warning" color="#FF6B6B">
  This lesson uses an insecure CLI login with plaintext credentials for demonstration only. Do not use `--insecure` or `--plaintext` in production.
</Callout>

<Callout icon="lightbulb" color="#1CB2FE">
  Prerequisites: the `argocd` CLI and `kubectl` must be installed and configured to talk to the target cluster where Argo CD is running. The sample app repository used in this lesson is `https://github.com/argoproj/argocd-example-apps.git`.
</Callout>

## Quick reference: key commands

| Action                            | Command                                                                                                                                                                                 |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Log in to Argo CD (insecure demo) | `argocd login localhost:3000 --username admin --password admin123 --insecure --plaintext`                                                                                               |
| Create application from Git       | `argocd app create web-frontend --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace applications` |
| Get app status                    | `argocd app get web-frontend`                                                                                                                                                           |
| Sync app (manual)                 | `argocd app sync web-frontend`                                                                                                                                                          |
| Set automated sync                | `argocd app set web-frontend --sync-policy automated`                                                                                                                                   |
| Enable self-heal                  | `argocd app set web-frontend --self-heal`                                                                                                                                               |
| Enable auto-prune                 | `argocd app set web-frontend --auto-prune`                                                                                                                                              |
| Wait for sync to complete         | `argocd app wait web-frontend --timeout 120`                                                                                                                                            |
| Scale deployment (test self-heal) | `kubectl scale deployment guestbook-ui -n applications --replicas=5`                                                                                                                    |
| Describe deployment events        | `kubectl describe deployment guestbook-ui -n applications`                                                                                                                              |

***

## 1) Log in to Argo CD via CLI

In this lab Argo CD is exposed on a NodePort at `localhost:3000`. Log in using the `argocd` CLI:

```bash theme={null}
argocd login localhost:3000 \
  --username admin \
  --password admin123 \
  --insecure \
  --plaintext
```

Example successful output:

```text theme={null}
'admin' logged in successfully
Context 'localhost:3000' updated
```

Tip: In production use TLS and a secure authentication method (SSO, OIDC, or client certificates). See the Argo CD docs for recommended authentication setups.

## 2) Create the application from the CLI

Create the `web-frontend` application using the sample guestbook app repository:

```bash theme={null}
argocd app create web-frontend \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace applications
```

Example confirmation:

```text theme={null}
application 'web-frontend' created
```

By default Argo CD creates the application with Manual sync (no automatic sync). Check the application status:

```bash theme={null}
argocd app get web-frontend
```

Representative output (trimmed):

```text theme={null}
Name:               argocd/web-frontend
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          applications
Source:
  Repo:             https://github.com/argoproj/argocd-example-apps.git
  Path:             guestbook
Sync Policy:        Manual
Sync Status:        OutOfSync
Health Status:      Unknown
```

At this point the `applications` namespace has no resources managed by Argo CD:

```bash theme={null}
kubectl get pods -n applications
```

Output:

```text theme={null}
No resources found in applications namespace.
```

## 3) Sync the application

Perform a manual sync to apply the manifests from Git into the cluster:

```bash theme={null}
argocd app sync web-frontend
```

Example (trimmed) sync output:

```text theme={null}
Operation: Sync
Phase:     Succeeded
Message:   successfully synced (all tasks run)

GROUP  KIND        NAMESPACE     NAME           STATUS   HEALTH    MESSAGE
apps   Service     applications  guestbook-ui   Synced   Progressing  service/guestbook-ui created
apps   Deployment  applications  guestbook-ui   Synced   Healthy      deployment.apps/guestbook-ui created
```

Re-check the application status:

```bash theme={null}
argocd app get web-frontend
```

Now the app should report Synced and Healthy (or briefly Progressing while the Deployment becomes ready):

```text theme={null}
Sync Status:       Synced to (rev)
Health Status:     Healthy
```

Confirm pods are running in the `applications` namespace:

```bash theme={null}
kubectl get pods -n applications
```

Example:

```text theme={null}
NAME                             READY   STATUS    RESTARTS   AGE
guestbook-ui-659f948bd-5ndrw     1/1     Running   0          35s
```

## 4) Enable automated sync and self-heal

To make the application automatically reconcile differences and prune deleted resources, switch the sync policy to Automated and enable self-heal and auto-prune:

Set automated sync policy:

```bash theme={null}
argocd app set web-frontend --sync-policy automated
```

Enable self-heal (revert out-of-band cluster changes) and auto-prune (remove resources deleted from Git):

```bash theme={null}
argocd app set web-frontend --self-heal
argocd app set web-frontend --auto-prune
```

Verify the application sync policy and sync options:

```bash theme={null}
argocd app get web-frontend
```

Representative output:

```text theme={null}
Sync Allowed:     Automatic (Prune)
Sync Policy:      Automated
Sync Status:      Synced to (rev)
Health Status:    Healthy
```

You can view available sync flags with `argocd app set --help`. Example sync-related flags include:

```text theme={null}
--sync-option Prune=false                   Add or remove a sync option, e.g., add Prune=false. Remove using '!' prefix, e.g. '!Prune=false'
--sync-policy string                        Sync policy (manual|automated)
--sync-retry-backoff-duration duration      Sync retry backoff duration (e.g. 2m, 1h) (default 5s)
--sync-retry-backoff-factor int             Sync retry backoff factor (default 2)
--sync-retry-limit int                      Max number of allowed sync retries
--sync-retry-refresh                         Use the latest revision when retrying instead of the initial one
...
```

## 5) Demonstrate self-heal by scaling the Deployment

To verify self-healing works, simulate an out-of-band change by scaling the `guestbook-ui` Deployment to 5 replicas:

```bash theme={null}
kubectl scale deployment guestbook-ui -n applications --replicas=5
```

Then observe app status and pods:

```bash theme={null}
argocd app get web-frontend
kubectl get pods -n applications
```

Argo CD will detect the divergence (cluster state != Git desired state) and automatically reconcile the cluster to the Git-declared state. After reconciliation the Deployment should revert to the desired replica count defined in Git.

Example status after reconciliation:

```text theme={null}
Sync Status:      Synced to (rev)
Health Status:    Healthy

GROUP      KIND        NAMESPACE     NAME          STATUS   HEALTH   MESSAGE
apps       Deployment  applications  guestbook-ui  Synced   Healthy  deployment.apps/guestbook-ui configured
```

Pod list after auto-heal:

```bash theme={null}
kubectl get pods -n applications
```

Example:

```text theme={null}
NAME                             READY   STATUS    RESTARTS   AGE
guestbook-ui-695f948db-5ndrv     1/1     Running   0          2m54s
```

## 6) Inspect Deployment events

Describe the Deployment to see events showing scale up and scale down actions. Note: when rendering MDX, literal XML/HTML-like tokens such as `<none>` must be written as inline code to avoid being parsed as JSX. The excerpt below uses the backticked escapes shown to ensure proper rendering.

```bash theme={null}
kubectl describe deployment guestbook-ui -n applications
```

Example (trimmed):

```text theme={null}
Name:                   guestbook-ui
Namespace:              applications
Annotations:            argocd.argoproj.io/tracking-id: web-frontend:apps/Deployment:applications/guestbook-ui
Selector:               app=guestbook-ui
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
Pod Template:
  Labels:  app=guestbook-ui
  Containers:
   guestbook-ui:
    Image:      gcr.io/google-samples/gb-frontend:v5
    Port:       80/TCP
    Host Port:  0/TCP
    Environment: `\`<none>\``
    Mounts: `\`<none>\``
  Volumes:                `\`<none>\``
Conditions:
  Type              Status  Reason
  Progressing       True    NewReplicaSetAvailable

Events:
  Type    Reason                  Age                 From                       Message
  ----    ------                  ----                ----                       -------
  Normal  ScalingReplicaSet       3m18s               deployment-controller      Scaled up replica set guestbook-ui-659f948db from 0 to 1
  Normal  ScalingReplicaSet       52s                 deployment-controller      Scaled up replica set guestbook-ui-659f948db from 1 to 5
  Normal  ScalingReplicaSet       52s                 deployment-controller      Scaled down replica set guestbook-ui-659f948db from 5 to 1
```

(Backticked `<none>` entries ensure MDX does not try to interpret angle-bracket content as JSX.)

## 7) Waiting for syncs

If a sync is in progress and you want to block until it finishes, use `argocd app wait`:

```bash theme={null}
argocd app wait web-frontend --timeout 120
```

This command waits until the application reaches a syncable and healthy state or the timeout is reached. It exits non-zero on timeout or failure.

***

This completes a CLI-based workflow for creating Argo CD applications, syncing them, enabling automated reconciliation (self-heal and auto-prune), and validating self-healing behavior. For more detail and production best practices, see:

* [Argo CD Documentation](https://argo-cd.readthedocs.io/)
* [Argo CD GitHub: argocd-example-apps](https://github.com/argoproj/argocd-example-apps)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/96c7a748-3848-424d-9933-e90d4f8cd5dd" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/efa3002a-4ec7-4dde-abad-486ec7e5bcd9" />
</CardGroup>
