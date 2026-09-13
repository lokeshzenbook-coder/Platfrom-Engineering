> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo GitOps Delivery with ArgoCD UI

> Walkthrough of using Argo CD web UI to create, sync, detect drift, and deploy Kubernetes manifests and Helm charts via GitOps.

Argo CD is a popular GitOps continuous delivery tool. In GitOps you declare the desired cluster state in a Git repository, and Argo CD continuously watches that repository and reconciles the live cluster with the declared state. This walkthrough uses the Argo CD web UI to build a mental model of application creation, syncing, drift detection, and Helm chart deployments before you manage Argo CD from the CLI.

## Prerequisites

* A Kubernetes cluster with Argo CD installed in the `argocd` namespace.
* Network access to the Argo CD server UI.
* For this demo we used the `admin` user with password `admin123` (replace with your secure password in real environments).

<Callout icon="lightbulb" color="#1CB2FE">
  Use a secure admin password in production. You can create dedicated Argo CD users or integrate SSO instead of reusing the `admin` account.
</Callout>

## 1. Verify Argo CD pods

Confirm Argo CD components are running:

```bash theme={null}
kubectl get pods -n argocd
```

Example output:

```bash theme={null}
NAME                                            READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                 1/1     Running   0          7m
argocd-applicationset-controller-0              1/1     Running   0          7m
argocd-dex-server-656ffbf4b5-tcmhp             1/1     Running   0          7m
argocd-notifications-controller-0               1/1     Running   0          7m
argocd-redis-7c767cd79-kf8mf                    1/1     Running   0          7m
argocd-repo-server-779879c89d-w7fxl             1/1     Running   0          7m
argocd-server-5c54c9fdd4-tpqnf                  1/1     Running   0          7m
```

Log in to the Argo CD UI. On a fresh install the Applications view may be empty:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-no-applications-interface.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=9619e855101f7e8c454edf3c04064575" alt="The image shows an Argo CD interface with a message indicating that no applications are available. There's a prompt to create a new application to manage resources in the cluster." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-no-applications-interface.jpg" />
</Frame>

## 2. Create an application from manifests (guestbook example)

Steps to create a new application via the UI:

* Click **New App**.
* Name: `web-frontend`
* Project: `default`
* Sync policy: start with **Manual** (so you can observe actions before enabling auto-sync).
* Source: point to the Argo CD example repository: `https://github.com/argoproj/argocd-example-apps`, path `guestbook`.
* Destination: your in-cluster API server and choose to use the application namespace.

The New Application configuration in the UI looks like this:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-new-application-settings.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=a13d77734fa74535e7402fc04e82cb30" alt="The image shows the Argo CD interface with settings for creating a new application, including sections for configuring the source repository and destination cluster." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-new-application-settings.jpg" />
</Frame>

Ensure the target namespace exists (for example `applications`):

```bash theme={null}
kubectl get namespaces
```

Create the application. After creation Argo CD will compare Git to the cluster and should show the app as OutOfSync since the resources are not yet in-cluster:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-web-frontend-status-outofsync.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=e82e64ce870e55e638ea160d1a48b157" alt="The image shows the Argo CD interface displaying the status of a &#x22;web-frontend&#x22; project, which is marked as &#x22;OutOfSync&#x22; and &#x22;Missing.&#x22; There are options to sync, refresh, or delete the application." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-web-frontend-status-outofsync.jpg" />
</Frame>

Open the application to view the resource tree. "OutOfSync / Missing" entries indicate Argo CD detected resources in Git that are not present in the cluster. Click **Sync** and then **Synchronize** to apply the manifests. Argo CD will apply the manifests and the application will move to `Synced` as the cluster state matches the declared state.

You can inspect the deployed resources in the target namespace:

```bash theme={null}
kubectl get all -n applications
```

Example output after syncing:

```bash theme={null}
NAME                                READY   STATUS    RESTARTS   AGE
pod/guestbook-ui-659f948d8-5xgzh    1/1     Running   0          30s

NAME                        TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/guestbook-ui        ClusterIP  172.20.182.60    <none>        80/TCP    30s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/guestbook-ui        1/1     1            1           30s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/guestbook-ui-659f948d8      1         1         1       30s
```

Refresh the app in the Argo CD UI and you should see `Healthy` and `Synced`.

## 3. Observe drift detection and reconciliation

Make a manual change on the cluster to simulate drift. For example, scale the deployment to 5 replicas:

```bash theme={null}
kubectl scale deployment guestbook-ui -n applications --replicas=5
kubectl get all -n applications
```

Example output after scaling to five replicas:

```bash theme={null}
NAME                                READY   STATUS    RESTARTS   AGE
pod/guestbook-ui-6595f948d-aaaa     1/1     Running   0          10s
pod/guestbook-ui-6595f948d-bbbb     1/1     Running   0          10s
pod/guestbook-ui-6595f948d-cccc     1/1     Running   0          10s
pod/guestbook-ui-6595f948d-dddd     1/1     Running   0          10s
pod/guestbook-ui-6595f948d-eeee     1/1     Running   0          10s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/guestbook-ui        5/5     5            5           10s
```

Refresh the application in Argo CD — it will now show `OutOfSync`. Use the Diff view to inspect differences (cluster on the left, Git desired state on the right). For the `replicas` field you’ll see:

Cluster (actual):

```yaml theme={null}
spec:
  replicas: 5
```

Desired (Git):

```yaml theme={null}
spec:
  replicas: 1
```

Click **Sync** in Argo CD to restore the declared state. Argo CD will scale the deployment back to the desired replica count (1), and the app will return to `Synced` and `Healthy`.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-web-frontend-deployment-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=18ee8a7da4b0b93767556c22b8a8b336" alt="The image shows an Argo CD interface with an application deployment diagram, indicating the health and sync status of a &#x22;web-frontend&#x22; application and its components. The application is &#x22;OutOfSync&#x22; but &#x22;Healthy&#x22;, and includes various services, deployments, and pods." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-web-frontend-deployment-diagram.jpg" />
</Frame>

## 4. Deploying Helm charts from Git

Argo CD can render and deploy Helm charts directly from a Git repo. The same example repository includes a Helm guestbook chart and values file:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argocd-example-apps-helm-guestbook-directory.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=27636f2a0c67b8aabe42cc7cd50fc8eb" alt="This image shows a GitHub repository page for &#x22;argocd-example-apps,&#x22; specifically focusing on the &#x22;helm-guestbook&#x22; directory. It lists files and folders along with their last commit messages and dates." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argocd-example-apps-helm-guestbook-directory.jpg" />
</Frame>

Create a new application for the Helm chart:

* Repository: `https://github.com/argoproj/argocd-example-apps`
* Path: `applications-helm/guestbook` (path to the Helm chart)
* Sync policy: set to **Automatic** if you want Argo CD to auto-reconcile (self-heal).
* Destination namespace: `applications-helm` (optionally enable **Auto Create Namespace**).

Argo CD will detect the chart and display its `values.yaml` in the UI so you can override values before creating the application. For example, change `replicaCount` from `1` to `2` in the UI:

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-helm-configuration-ui.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=5c513babce54c5fa0101e0f49af6b2c6" alt="The image shows an Argo CD user interface displaying a Helm configuration with parameters like containerPort, image.repository, and service.type. There are sync and health status indicators on the left." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-helm-configuration-ui.jpg" />
</Frame>

Click **Create**. Argo CD will render the Helm templates with your overrides and deploy them. Verify with kubectl:

```bash theme={null}
kubectl get pods -n applications-helm
```

You should see the number of pods matching the value you set (for example, two pods if `replicaCount` was set to 2).

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-web-backend-dashboard-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=fa9ec2d982f068124657ce47b566341f" alt="The image shows a dashboard from Argo CD displaying the status of a &#x22;web-backend&#x22; application, with its health marked as &#x22;Healthy&#x22; and sync status as &#x22;Synced.&#x22; It includes a flow diagram depicting the components and relationships within the application architecture." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Demo-GitOps-Delivery-with-ArgoCD-UI/argo-cd-web-backend-dashboard-diagram.jpg" />
</Frame>

<Callout icon="warning" color="#FF6B6B">
  When enabling Automatic sync, Argo CD will immediately reconcile any divergence. Use automatic sync for production only if you’ve validated the repo and policies; otherwise keep Manual sync while testing.
</Callout>

## Quick reference: Commands & concepts

| Topic                               | Command / Action                                                     | Notes                                                                 |
| ----------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Check Argo CD pods                  | `kubectl get pods -n argocd`                                         | Ensure core components are Running.                                   |
| Create namespace                    | `kubectl create namespace applications`                              | Ensure destination namespace exists or enable Auto Create Namespace.  |
| List app resources                  | `kubectl get all -n applications`                                    | Confirm manifests deployed by Argo CD.                                |
| Scale a deployment (simulate drift) | `kubectl scale deployment guestbook-ui -n applications --replicas=5` | Observe OutOfSync in UI and use Diff view.                            |
| Helm chart path example             | `applications-helm/guestbook`                                        | Argo CD will render Helm templates and allow `values.yaml` overrides. |
| GitOps pattern                      | Git repo → Argo CD watches → Reconcile loop                          | Argo CD enforces desired state and reports drift.                     |

## Where to go next

* Argo CD docs: [https://argo-cd.readthedocs.io/](https://argo-cd.readthedocs.io/)
* Example repo used in this demo: [https://github.com/argoproj/argocd-example-apps](https://github.com/argoproj/argocd-example-apps)

This overview demonstrated creating applications from plain manifests and Helm charts, inspecting resource diffs, syncing, and observing Argo CD reconciliation behavior. Next you can explore automating Argo CD via the CLI and managing applications using `argocd` commands and ApplicationSets.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/e3b88945-2390-4aad-9959-16c8512e6322" />
</CardGroup>
