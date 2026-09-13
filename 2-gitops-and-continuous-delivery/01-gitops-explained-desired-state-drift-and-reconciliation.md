> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# GitOps Explained Desired State Drift and Reconciliation

> Explains GitOps principles, the declarative desired state model, reconciliation loop, drift detection and self healing, and the four pillars enabling secure auditable continuous delivery.

Welcome to the GitOps and Continuous Delivery lesson.

Scenario: It's 2 a.m. and your pager goes off — production is down. You SSH into the cluster to investigate, but something changed and you have no record of what, when, or who. You're flying blind. This is the world before GitOps.

In this lesson you'll learn:

* The problems GitOps solves,
* The four pillars of GitOps,
* How the reconciliation loop functions, and
* How drift detection and self-healing work.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/gitops-learning-objectives-reconciliation-drifts.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=cc808a3882ab45281b50013b6517880d" alt="The image lists four learning objectives related to GitOps, including understanding its problems, defining its pillars, explaining the reconciliation loop, and understanding drift detection and self-healing strategies." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/gitops-learning-objectives-reconciliation-drifts.jpg" />
</Frame>

Why incidents like this occur

Many Kubernetes environments grant developers direct `kubectl` access to production. That enables ad-hoc, imperative changes from developers' laptops. Those changes are applied directly, often without a centralized audit trail or version history.

Example (imperative action):

```bash theme={null}
# example of an imperative client action
kubectl apply -f ./manifest.yaml
```

When multiple people make ad-hoc changes—especially during high-pressure incidents—conflicting modifications can be introduced simultaneously. With no single source of truth and no formal change history, teams cannot track who changed what, when, or why.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/deployment-roulette-flowchart-teams.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=d6eae91fd96b9c8fedbdc0d5268c6b03" alt="The image illustrates a flowchart titled &#x22;Deployment Roulette,&#x22; showing three uncoordinated teams (Dev, Ops, SRE) sending conflicting changes to a production cluster." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/deployment-roulette-flowchart-teams.jpg" />
</Frame>

Consequences: configuration drift, unpredictable behavior, and prolonged outages

Uncoordinated changes create configuration drift between environments, unpredictable production behavior, and outages caused by incompatible or unintended changes. This makes root cause analysis difficult and puts production stability in the hands of human coordination and memory rather than an auditable system.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/deployment-roulette-issues-outline.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=98dafe62be99c22e255c01e5a9f318c6" alt="The image outlines the concept of &#x22;Deployment Roulette,&#x22; highlighting three issues: configuration drift between environments, unpredictable production behavior, and prolonged outages from incompatible or unintended changes." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/deployment-roulette-issues-outline.jpg" />
</Frame>

Imperative vs. Declarative

Root cause: the imperative model. Running imperative commands (scale, set image, etc.) immediately changes cluster state and leaves only ephemeral terminal history.

Imperative examples:

```bash theme={null}
# Imperative examples (state-changing commands run ad-hoc)
kubectl scale deployment api --replicas=5
kubectl set image deployment/web web=nginx:1.21
```

There is no integrated version history, no central audit trail, and no automatic detection when the same resource changes later.

By contrast, the declarative model records desired state in files (YAML/JSON) committed to Git. Git automatically provides a versioned history so you can see who changed what, when, and why (when commit messages are used).

Example (declarative snippet):

```yaml theme={null}
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
```

The key insight: GitOps replaces imperative “do this now” commands with declarative desired state and automation that converges actual cluster state to that declared state.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/imperative-vs-declarative-commands-comparison.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=b29462c2b23f4a918339b2f84a621481" alt="The image compares imperative commands (before) with declarative commands (after), highlighting a shift from step-by-step instructions to automation and desired state declaration." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/imperative-vs-declarative-commands-comparison.jpg" />
</Frame>

What is GitOps?

GitOps is an operational model where Git repositories store declarative descriptions of infrastructure and applications. Automated controllers running in the cluster ensure the cluster state matches those Git-declared states.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/git-source-of-truth-gitops-yaml-json.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=3da92db67c02024f597a5dd07127eb90" alt="The image illustrates Git as the single source of truth in a GitOps operational model, connecting Git repositories to declarative descriptions in YAML/JSON format." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/git-source-of-truth-gitops-yaml-json.jpg" />
</Frame>

The four pillars of GitOps

| Pillar      | What it means                                  | Why it matters                                                                |
| ----------- | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| Declarative | Desired state defined as files (`YAML`/`JSON`) | Describes the intended system rather than individual commands                 |
| Versioned   | All manifests stored in Git                    | Complete commit history, easy rollbacks and audits                            |
| Pull-based  | Agents inside the cluster pull from Git        | No CI with persistent cluster creds; reduces secret sprawl and attack surface |
| Reconciled  | Controllers continuously converge state        | Detects and corrects drift; enables self-healing                              |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/git-single-source-of-truth-features.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=aea4cca11ec30e5baa598d270bc3506e" alt="The image illustrates four features of Git as a single source of truth: Declarative, Versioned, Pulled, and Reconciled, with brief descriptions of each." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/git-single-source-of-truth-features.jpg" />
</Frame>

If it’s not in Git, it does not exist. If it's in Git, it should be in the cluster.

How the reconciliation loop works

The reconciliation loop is the engine of GitOps. Controllers such as Argo CD and Flux operate continuously in three core phases:

* Watch: The controller detects Git changes via polling, webhooks, or provider notifications.
* Compare: It compares the declared desired state in Git against the actual cluster state resource-by-resource.
* Sync: When differences are found, the controller applies the necessary changes (create/update/delete) to align the cluster with Git.

This loop integrates with developer workflows: Code & Push → Detect & Compare → Sync & Deploy → Continuous Watch.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/reconciliation-loop-cyclical-process-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=02a82f3fab012e10a2e76c55f837ec8c" alt="The image depicts &#x22;The Reconciliation Loop,&#x22; a cyclical process involving four steps: Code & Push, Detect & Compare, Sync & Deploy, and Continuous Watch, each with brief descriptions." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/reconciliation-loop-cyclical-process-diagram.jpg" />
</Frame>

Example: you change a Deployment's image from `1.0` to `2.0` and push the commit. Within the controller's detection window (seconds to minutes), it sees the Git commit, compares the cluster's running `1.0` deployment, and applies the update to `2.0`. The loop is continuous; it’s not a one-time deployment.

Drift detection and self-healing

Drift occurs when the cluster diverges from Git-declared desired state. Controllers handle drift in several ways depending on policy and environment sensitivity.

| Response                 | Behavior                                                                             | Use case                                                       |
| ------------------------ | ------------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| Auto-sync (self-healing) | Controller automatically reverts unintended changes to match Git                     | Standard GitOps for most environments                          |
| Alert + manual sync      | Controller alerts team and waits for approval to apply changes                       | Production systems requiring human checks                      |
| Diff + PR workflow       | Controller generates a diff or PR to update Git when cluster changes are intentional | When cluster-initiated changes must be reflected back into Git |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/drift-detection-self-healing-github-process.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=5306693b7d38bcde849012d9a4255672" alt="The image outlines a drift detection and self-healing process using GitHub, featuring auto-sync for self-healing and alert and manual sync for human approval." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/GitOps-Explained-Desired-State-Drift-and-Reconciliation/drift-detection-self-healing-github-process.jpg" />
</Frame>

Common sources of drift

* Manual `kubectl` edits and ad-hoc changes.
* Autoscalers (HPA/VPA) adjusting replicas or resource requests/limits.
* Other controllers or operators adding annotations/labels or modifying resources.

Principle to follow: Git is authoritative. Either the controller syncs the cluster to Git, or you update Git to reflect an intentional change in the cluster. The cluster is not the source of truth.

<Callout icon="lightbulb" color="#1CB2FE">
  Using a pull-based model (agents inside the cluster pulling from Git) reduces secret sprawl. CI/CD systems do not need persistent cluster credentials to apply changes, improving security and auditability.
</Callout>

Recap — core takeaways

* Git is the single source of truth: desired state belongs in version-controlled repositories.
* Prefer declarative manifests over imperative commands: state is described, automation handles convergence.
* Continuous reconciliation prevents unintentional drift: controllers compare and correct constantly.
* Pull-based deployments are more secure and auditable because the cluster pulls from Git and CI/CD systems avoid persistent credentials.

Further reading and references

* Argo CD: [https://argo-cd.readthedocs.io](https://argo-cd.readthedocs.io)
* Flux: [https://fluxcd.io/](https://fluxcd.io/)
* GitOps principles and best practices: [https://www.gitops.tech/](https://www.gitops.tech/)

This concludes the lesson on GitOps, desired state, drift, and reconciliation.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/c28403c7-4078-4ed2-a273-ec35b204e887" />
</CardGroup>
