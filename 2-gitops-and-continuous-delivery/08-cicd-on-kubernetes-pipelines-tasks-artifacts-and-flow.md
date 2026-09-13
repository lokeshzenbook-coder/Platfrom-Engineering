> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# CICD on Kubernetes Pipelines Tasks Artifacts and Flow

> Kubernetes-native CI practices, comparing Tekton and Argo Workflows for in-cluster builds, testing, artifact handling, security, and pipeline orchestration.

This lesson focuses on continuous integration (CI) inside Kubernetes: building container images, running tests, scanning for vulnerabilities, and producing artifacts used for automated deployments. We’ll explain why running CI natively on Kubernetes is often preferable, introduce two CNCF projects—Tekton and Argo Workflows—and compare their architectures, execution models, and best-fit use cases.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/kubernetes-ci-cd-learnings-teknon-argo.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=9f790d029b6f96011f0f4c1ae12f5169" alt="The image lists four learning objectives related to Kubernetes-native CI/CD, Tekton, and Argo Workflows, including understanding advantages and defining tasks." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/kubernetes-ci-cd-learnings-teknon-argo.jpg" />
</Frame>

Why run CI inside Kubernetes?
Traditionally, CI systems (Jenkins, GitHub Actions, GitLab CI) run outside your cluster. That model works, but it introduces operational and security trade-offs. Consider this real-world example:

Example story
A company ran Jenkins on a dedicated EC2 instance with cluster-admin access to three clusters, push rights to the container registry, and secrets for seven environments. After a Jenkins plugin vulnerability was disclosed, the organization had to rotate cluster credentials, regenerate registry tokens, and replace deployment secrets across environments. The emergency took two weeks.

<Callout icon="warning" color="#FF6B6B">
  Centralized CI servers with broad privileges create a high blast radius. When one service is compromised or vulnerable, all stored credentials and access can be exposed. Moving CI into the cluster reduces external credential sprawl and improves isolation.
</Callout>

The core problems in the external-CI model:

* Separate infrastructure to manage: patching, scaling, and securing another server or cluster.
* Credential sprawl: the CI server stores cluster, registry, and environment secrets.
* Resource inefficiency: always-on static runners sit idle between builds.
* Limited Kubernetes integration: external CI cannot leverage Kubernetes RBAC, service accounts, pod-level resource limits, or pod security policies.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/ci-systems-challenges-non-kubernetes.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=ca902ae09f1695695b23767e3bd29d78" alt="The image lists challenges of CI systems outside Kubernetes, including separate infrastructure, credential sprawl, resource inefficiency, and lack of native Kubernetes integration." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/ci-systems-challenges-non-kubernetes.jpg" />
</Frame>

Kubernetes-native CI: four core benefits
When CI runs in-cluster, every build step runs as a pod. This provides significant advantages:

* Pods as runners: Kubernetes schedules and scales each task as a pod—no always-on VM required.
* Native RBAC: Use Kubernetes service accounts and Role/ClusterRole bindings to limit each pipeline’s privileges.
* Secret integration: Mount Kubernetes Secrets directly into pods; credentials remain inside the cluster.
* Declarative pipelines: Pipelines live as CRDs (Custom Resource Definitions). Store them in Git and manage them with GitOps.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/kubernetes-cicd-infographic-declarative-pipelines.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=f66c9c06fe803da0403b3259f0b73f99" alt="The image is an infographic about Kubernetes-Native CI/CD, highlighting features like declarative pipelines, pods as runners, secrets integration, and native RBAC." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/kubernetes-cicd-infographic-declarative-pipelines.jpg" />
</Frame>

CNCF projects for Kubernetes-native CI/CD
Two CNCF projects commonly used for Kubernetes-native CI/CD:

* Tekton — modular, CI/CD-focused primitives (Tasks, Pipelines, TaskRuns, PipelineRuns, Workspaces).
* Argo Workflows — a general-purpose workflow engine (Workflows, Templates, DAGs, Artifacts).

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/tekton-vs-argo-kubernetes-ci-cd.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=108eb22215f9d739ca0ebcde8107ed81" alt="The image compares two Kubernetes-native CI/CD tools: Tekton and Argo Workflows, highlighting their features as CNCF projects." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/tekton-vs-argo-kubernetes-ci-cd.jpg" />
</Frame>

Tekton: CI/CD primitives and shared-workspace efficiency
Tekton is purpose-built for CI/CD and is designed around a small set of CRDs:

* Task: A sequence of steps that execute in the same pod. Steps are containers that share the pod filesystem.
* Pipeline: A composition of Tasks with ordering/dependency semantics. Independent Tasks can run in parallel.
* TaskRun / PipelineRun: Execution instances of Tasks or Pipelines.
* Workspace: Shared storage (for example a PVC) mounted into Tasks for passing files and artifacts.

Because multiple steps run in one pod and share a Workspace, Tekton avoids continuous artifact upload/download between steps—this is efficient for build-and-test pipelines.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/tekton-building-blocks-task-pipeline.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=2ffe3335be05bcb8065158093af7e83c" alt="The image explains the building blocks of Tekton, highlighting Task, Pipeline, and TaskRun/PipelineRun with brief descriptions for each component." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/tekton-building-blocks-task-pipeline.jpg" />
</Frame>

Example Tekton Pipeline
This example pipeline includes three Tasks: `fetch-source`, `build-image`, and `test`. `runAfter` enforces ordering so `build-image` and `test` run only after `fetch-source` completes; they may run in parallel after that point.

```yaml theme={null}
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-test
spec:
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone

    - name: build-image
      taskRef:
        name: kaniko
      runAfter:
        - fetch-source

    - name: test
      taskRef:
        name: test
      runAfter:
        - fetch-source
```

Note: Without `runAfter`, Tekton will attempt to start tasks concurrently when dependencies are not defined.

Argo Workflows: DAG orchestration and artifact-driven steps
Argo Workflows is a general-purpose workflow engine suited to CI/CD, data pipelines, ML workflows, and other complex DAG-based orchestrations.

Key Argo concepts:

* Workflow: The top-level resource representing a run.
* Template: Reusable step definitions (container, script, DAG, or resource operation).
* DAG: Directed acyclic graph that defines dependencies between tasks.
* Artifacts: Files passed between steps; commonly stored in object stores like S3 or GCS.

Argo runs each step in a separate pod—this enforces stronger per-step isolation but typically requires artifact storage to pass data between steps.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/argo-workflows-dag-engine-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=5c8f9650776be92dec0ff883d3300215" alt="The image explains the core concepts of Argo Workflows' DAG engine, highlighting Workflow, Template, DAG, and Artifacts in a colorful semicircular diagram." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/argo-workflows-dag-engine-diagram.jpg" />
</Frame>

Example Argo Workflow
This Workflow demonstrates a DAG with three tasks: `build`, `test`, and `deploy`. `build` and `test` run in parallel, while `deploy` depends on both.

```yaml theme={null}
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: ci-pipeline-
spec:
  entrypoint: ci-pipeline
  templates:
    - name: ci-pipeline
      dag:
        tasks:
          - name: build
            template: build-image

          - name: test
            template: run-tests

          - name: deploy
            template: deploy-app
            dependencies:
              - build
              - test
```

Argo’s `dependencies` in a DAG play the same role as Tekton’s `runAfter`—they express ordering constraints using different syntax.

Comparison: Tekton vs Argo Workflows
Both projects are Kubernetes-native and integrate well with GitOps practices, but they serve different primary use cases and have distinct execution models.

| Aspect          | Tekton                                                                  | Argo Workflows                                                             |
| --------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Primary focus   | CI/CD primitives optimized for builds/tests                             | General DAG engine for CI, CD, data and ML workflows                       |
| Execution model | Multiple steps share a single pod per Task; shared files via Workspaces | Each step runs in its own pod; data passed via artifacts or object storage |
| Data passing    | PVC-backed `Workspace` for fast local file sharing                      | Artifact storage (S3/GCS) or artifact repositories                         |
| UI & ecosystem  | Lightweight dashboard, Tekton Hub for tasks                             | Rich UI, ecosystem: `Argo CD`, `Argo Events`, `Argo Rollouts`              |
| Best fit        | Standard CI pipelines with shared filesystem needs                      | Complex DAGs, data/ML pipelines, strong isolation between steps            |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/tekton-vs-argo-workflows-comparison-chart.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=1ff4c8f595458b77558dca0c681e6f77" alt="The image is a comparison chart between Tekton and Argo Workflows, highlighting differences in aspects such as primary focus, execution model, data passing, UI, ecosystem, and best use cases." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/CICD-on-Kubernetes-Pipelines-Tasks-Artifacts-and-Flow/tekton-vs-argo-workflows-comparison-chart.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Choose Tekton when you want CI/CD primitives with shared-workspace efficiency and tight integration for builds/tests. Choose Argo Workflows when you need a general DAG engine for complex orchestration, strong isolation between steps, or advanced artifact handling.
</Callout>

Best practices for Kubernetes-native CI

* Limit privileges: Give pipelines the minimum service account permissions required.
* Use ephemeral pods: Prefer short-lived pods for build steps to reduce blast radius.
* Store pipeline definitions in Git: Keep CRDs/pipeline YAML under version control and deploy via GitOps.
* Secure artifact storage: Use encrypted object stores and restrict access to artifact repositories.
* Reuse tasks/templates: Publish common Tasks (Tekton) or Templates (Argo) to a hub/catalog to avoid duplication.
* Monitor and autoscale: Use Kubernetes metrics and HPA/Cluster Autoscaler to scale workers and storage.

Wrap-up
Kubernetes-native CI replaces external runners with pod-based execution and offers native RBAC, better secret handling, scalable ephemeral runners, and declarative pipelines stored in Git. Tekton is tailored to CI/CD with efficient shared workspaces and Task/Pipeline primitives; Argo Workflows excels as a DAG engine for complex orchestration where per-step isolation and artifact-driven workflows are preferred. Both are CRD-based, support Git-driven workflows, and help build secure, scalable, maintainable CI systems inside your cluster.

Links and references

* Tekton: [https://tekton.dev/](https://tekton.dev/)
* Argo Workflows: [https://argoproj.github.io/argo-workflows/](https://argoproj.github.io/argo-workflows/)
* Argo CD: [https://argo-cd.readthedocs.io/](https://argo-cd.readthedocs.io/)
* Kaniko (example image builder used with Tekton): [https://github.com/GoogleContainerTools/kaniko](https://github.com/GoogleContainerTools/kaniko)
* Kubernetes RBAC concept: [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/3a285de6-56db-4118-8f22-a8fc032d2731" />
</CardGroup>
