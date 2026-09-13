> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Configuration Templating Parameterization and Environment Management

> Comparison of Helm and Kustomize for templating and patching Kubernetes manifests across environments.

We designed a clean repo layout with `base/` and `overlays/` directories, but the YAML files in `base/` are static and contain hard-coded values. When you need the same application deployed across multiple environments with different replicas, images, or resource limits, copying and pasting manifests is the naive — and fragile — approach.

In this lesson we compare two practical approaches to parameterize and manage environment differences: Helm (template-driven) and Kustomize (patch-driven). Both solve the copy‑paste problem differently, and both integrate with GitOps tools such as Argo CD and Flux.

* Why copy-and-paste YAML does not scale
* How Helm templates and Kustomize overlays address parameterization
* When to choose Helm vs. Kustomize and how they fit into GitOps workflows

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/yaml-helm-kustomize-learning-objectives.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=06678368d6b4d5dca13e4a8658b4c5b9" alt="The image outlines three learning objectives related to YAML, Helm, and Kustomize. It includes points about understanding YAML's limitations, Helm templates, and comparing Helm with Kustomize." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/yaml-helm-kustomize-learning-objectives.jpg" />
</Frame>

## Why copy–paste configuration is a problem

Imagine a healthcare platform team running three microservices across three environments (dev, staging, prod). For each service and environment they maintain separate Deployment, Service, and Ingress YAML files — dozens of nearly identical manifests.

One engineer updates a health-check path in most files but misses three files for the staging payment service. Staging health checks silently fail for two weeks because the alerting channel was archived. The root cause: manual copy-and-paste configuration. This pattern causes:

* Configuration drift across environments
* Tedious, error-prone updates
* Hard-to-audit differences and silent failures

<Callout icon="warning" color="#FF6B6B">
  Avoid maintaining environment-specific manifests by copying files. Copy‑paste leads to drift, missed updates, and increased operational risk.
</Callout>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/infrastructure-configuration-dev-staging-services.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=e1ef05a1af2d293e65ad9f62184015af" alt="The image depicts a platform infrastructure configuration with three environments: Dev, Staging, and another Dev, each containing services for Patient, Appointment, and Payment. Each service consists of multiple microservices labeled &#x22;ms&#x22; with corresponding numbers." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/infrastructure-configuration-dev-staging-services.jpg" />
</Frame>

When manifests are 90% identical, the meaningful 10% difference is hard to spot — like finding a needle in a haystack. Typos, missed updates, and inconsistent naming are common and costly.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/copy-paste-configuration-issues-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=19754fcb1ec4ff7c0972037e1ee1b25b" alt="The image outlines issues related to &#x22;Copy-Paste Configuration,&#x22; including drift between environments, tedious updates, hard-to-audit differences, and human error." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/copy-paste-configuration-issues-diagram.jpg" />
</Frame>

## Why YAML alone isn't enough

YAML is a data serialization format, not a programming language. It lacks variables, conditionals, loops, and cross-file references. Anchors and aliases can reduce duplication inside a single file, but they cannot:

* Reference values from other files or environments
* Provide conditional logic or iteration across manifests

To manage differences between environments, use external tooling that either generates or patches YAML based on parameters.

<Callout icon="lightbulb" color="#1CB2FE">
  YAML anchors and aliases reduce repetition within a single file, but they are not a replacement for templating or overlay tools when managing multiple environments.
</Callout>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/yaml-anchors-aliases-explanation.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=32131ce0b78547fbf9695651619aee28" alt="The image explains that YAML does not have variables but uses anchors and aliases, which only work within a single file and cannot reference external values." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/yaml-anchors-aliases-explanation.jpg" />
</Frame>

## Two pragmatic solutions: Helm (templating) and Kustomize (patching)

Both Helm and Kustomize transform a canonical manifest into environment-specific manifests, but they take different approaches. Both are supported by GitOps platforms:

* Argo CD: [https://learn.kodekloud.com/user/courses/gitops-with-argocd](https://learn.kodekloud.com/user/courses/gitops-with-argocd)
* Flux CD: [https://learn.kodekloud.com/user/courses/gitops-with-fluxcd](https://learn.kodekloud.com/user/courses/gitops-with-fluxcd)

## Helm (template-driven)

* Helm uses Go templates to generate Kubernetes YAML. You author template files with placeholders and logic, and you supply `values.yaml` files for environment-specific values.
* Helm Charts are packageable and versioned. A large ecosystem of community charts is available via Artifact Hub: [https://artifacthub.io/](https://artifacthub.io/)
* Templates support loops, conditionals, and functions, making Helm expressive for complex parameterization.
* Downside: Go template syntax (curly braces, pipelines, dot notation) can be tricky to read and debug in complex charts.

Minimal Helm chart example:

```yaml theme={null}
# Chart.yaml
apiVersion: v2
name: myapp
version: 0.1.0
```

```yaml theme={null}
# values.yaml
replicaCount: 1
image:
  repository: myapp
  tag: v1.0.0
```

```yaml theme={null}
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Rendering the chart with a different values file (for example `values-prod.yaml`) produces plain Kubernetes YAML with substituted values. Helm is particularly useful for:

* Packaging third-party applications (Prometheus, cert-manager)
* Reusing community charts
* Complex manifests that require logic

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/helm-template-engine-strengths-challenges.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=ea5a199bcdf0dd1d6f4b92053155277f" alt="The image describes the strengths and challenges of the Helm template engine. Strengths include rich templating and a large chart ecosystem, while challenges involve learning curve and debugging complexity." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/helm-template-engine-strengths-challenges.jpg" />
</Frame>

## Kustomize (patch-driven)

* Kustomize starts from valid Kubernetes YAML "bases" and applies declarative overlays (patches) per environment.
* Since base manifests are valid YAML, you can `kubectl apply -f base/` directly — bases are not templates.
* Overlays are focused: change the image tag, set replicas, add labels, or merge small diffs via patches. Kustomize is usually easier to read and debug for teams that prefer plain YAML.
* Limitation: no native loops or conditionals. Very complex transformations can become verbose.

Example Kustomize layout and files:

```text theme={null}
my-app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

```yaml theme={null}
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:latest
```

```yaml theme={null}
# base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
```

```yaml theme={null}
# overlays/prod/kustomization.yaml
resources:
  - ../../base
images:
  - name: myapp
    newTag: v2.1.0
patchesStrategicMerge:
  - replicas-patch.yaml
```

```yaml theme={null}
# overlays/prod/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

Render the overlay with:

```bash theme={null}
kubectl kustomize overlays/prod
```

Kustomize emphasizes small, explicit patches to an otherwise valid base. It’s built into `kubectl`, so extra tooling is often unnecessary. For many teams this makes debugging and review simpler.

## Trade-offs and guidance

Use the right tool for the right job:

* Helm
  * Best when you need expressive templating (loops, conditionals)
  * Ideal for packaging and installing third-party apps
  * Use when you want versioned charts and community ecosystem support
* Kustomize
  * Best when you prefer base manifests to remain valid YAML
  * Ideal for in-house services and small environment-specific patches
  * Simpler to read and debug for teams managing their own manifests

Many teams adopt a hybrid approach: use Helm for packaged third-party applications and Kustomize for internal service manifests. Both are supported by GitOps tools (Argo CD, Flux), so workflows can accommodate either or both.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/helm-vs-kustomize-comparison-chart.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=c1f8533bc246f9724ea075b13818c835" alt="The image is a comparison chart between Helm and Kustomize, highlighting their different approaches, base files, conditionals/loops, package ecosystem, and best use cases." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Configuration-Templating-Parameterization-and-Environment-Management/helm-vs-kustomize-comparison-chart.jpg" />
</Frame>

Comparison table: Helm vs Kustomize

| Area              | Helm (templating)                            | Kustomize (patching)                           |
| ----------------- | -------------------------------------------- | ---------------------------------------------- |
| Primary approach  | Generate YAML from Go templates              | Patch valid YAML bases                         |
| Best for          | Complex parameterization, third-party charts | Simple environment patches, in-house manifests |
| Templates / Logic | Loops, conditionals, functions               | No native loops or conditionals                |
| Ecosystem         | Large chart repository (Artifact Hub)        | Lightweight, built-in to kubectl               |
| Debuggability     | Can be harder to read for complex templates  | Easier to inspect resulting YAML and patches   |
| Packaging         | Chart packaging & versioning                 | No chart packaging; use Git structure          |

Further reading and references

* Helm documentation and charts: [https://artifacthub.io/](https://artifacthub.io/)
* Kustomize and `kubectl kustomize`: [https://kubernetes.io/docs/reference/kubectl/overview/](https://kubernetes.io/docs/reference/kubectl/overview/)
* GitOps with Argo CD: [https://learn.kodekloud.com/user/courses/gitops-with-argocd](https://learn.kodekloud.com/user/courses/gitops-with-argocd)
* GitOps with Flux CD: [https://learn.kodekloud.com/user/courses/gitops-with-fluxcd](https://learn.kodekloud.com/user/courses/gitops-with-fluxcd)
* Helm for Beginners course: [https://learn.kodekloud.com/user/courses/helm-for-beginners](https://learn.kodekloud.com/user/courses/helm-for-beginners)
* Kustomize course: [https://learn.kodekloud.com/user/courses/kustomize](https://learn.kodekloud.com/user/courses/kustomize)

## Summary

* Never manage multiple environments by copying YAML files.
* Helm generates YAML from Go templates and excels at expressive parameterization and packaging third-party apps.
* Kustomize patches valid YAML bases and is a good fit for readable overlays and simple environment changes.
* Both integrate with GitOps tools; choose the approach that best fits the team, the application, and operational needs.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/dd65e2cc-6e0c-487f-ac70-31523af5309e" />
</CardGroup>
