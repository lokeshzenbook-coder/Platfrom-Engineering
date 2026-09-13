# Platform Engineering

Notes and study material for the Certified Cloud Native Platform Engineer (CNPE) course.

## Table of Contents

### Platform Architecture and Infrastructure

1. [Your Platform Blueprint Architecture That Scales](./1-platform-architecture-and-infrastructure/01-your-platform-blueprint-architecture-that-scales.md)
2. [Right Sizing 101: Requests, Limits, QoS and Scheduling — Part 1](./1-platform-architecture-and-infrastructure/02-right-sizing-101-requests-limits-qos-and-scheduling-part-1.md)
3. [Right Sizing 101: Requests, Limits, QoS and Scheduling — Part 2](./1-platform-architecture-and-infrastructure/03-right-sizing-101-requests-limits-qos-and-scheduling-part-2.md)
4. [Multi Tenancy Made Practical: Models, Tradeoffs, Guardrails — Part 1](./1-platform-architecture-and-infrastructure/04-multi-tenancy-made-practical-models-tradeoffs-guardrails-part-1.md)
5. [Resource Governance: Defaults, Limits and Quotas](./1-platform-architecture-and-infrastructure/05-resource-governance-defaults-limits-and-quotas.md)
6. [Defaults Budgets: Designing LimitRanges and Quotas](./1-platform-architecture-and-infrastructure/06-defaults-budgets-designing-limitranges-and-quotas.md)
7. [Persistent Storage Concepts: Volumes, Claims and Provisioners](./1-platform-architecture-and-infrastructure/07-persistent-storage-concepts-volumes-claims-and-provisioners.md)
8. [Demo: Storage Classes in Action](./1-platform-architecture-and-infrastructure/08-demo-storage-classes-in-action.md)
9. [Platform Networking Concepts](./1-platform-architecture-and-infrastructure/09-platform-networking-concepts.md)
10. [Demo: Traffic Flow with Services and Ingress](./1-platform-architecture-and-infrastructure/10-demo-traffic-flow-with-services-and-ingress.md)
11. [Cost for Platforms: What Drives Spend and How to Reduce It](./1-platform-architecture-and-infrastructure/11-cost-for-platforms-what-drives-spend-and-how-to-reduce-it.md)
12. [Demo: Cost Visibility with OpenCost](./1-platform-architecture-and-infrastructure/12-demo-cost-visibility-with-opencost.md)

### GitOps and Continuous Delivery

1. [GitOps Explained: Desired State, Drift and Reconciliation](./2-gitops-and-continuous-delivery/01-gitops-explained-desired-state-drift-and-reconciliation.md)
2. [Repo Design That Works: Apps, Infra, Environments, Promotion](./2-gitops-and-continuous-delivery/02-repo-design-that-works-apps-infra-environments-promotion.md)
3. [Configuration Templating: Parameterization and Environment Management](./2-gitops-and-continuous-delivery/03-configuration-templating-parameterization-and-environment-management.md)
4. [GitOps Tool Landscape: ArgoCD vs Flux](./2-gitops-and-continuous-delivery/04-gitops-tool-landscape-argocd-vs-flux.md)
5. [ArgoCD Overview: Applications, Sync Policies and Helm Integration](./2-gitops-and-continuous-delivery/05-argocd-overview-applications-sync-policies-and-helm-integration.md)
6. [Demo: GitOps Delivery with ArgoCD UI](./2-gitops-and-continuous-delivery/06-demo-gitops-delivery-with-argocd-ui.md)
7. [Demo: GitOps Delivery with ArgoCD CLI](./2-gitops-and-continuous-delivery/07-demo-gitops-delivery-with-argocd-cli.md)
8. [CICD on Kubernetes: Pipelines, Tasks, Artifacts and Flow](./2-gitops-and-continuous-delivery/08-cicd-on-kubernetes-pipelines-tasks-artifacts-and-flow.md)
9. [Demo: Tekton Pipelines](./2-gitops-and-continuous-delivery/09-demo-tekton-pipelines.md)
10. [Progressive Delivery: Canary, Blue Green and Safe Rollbacks](./2-gitops-and-continuous-delivery/10-progressive-delivery-canary-blue-green-and-safe-rollbacks.md)
11. [Service Mesh Integration for Progressive Delivery](./2-gitops-and-continuous-delivery/11-service-mesh-integration-for-progressive-delivery.md)
12. [Demo: Progressive Delivery with Argo Rollouts](./2-gitops-and-continuous-delivery/12-demo-progressive-delivery-with-argo-rollouts.md)
13. [Delivery Troubleshooting: Drift, Permissions and Bad Configs](./2-gitops-and-continuous-delivery/13-delivery-troubleshooting-drift-permissions-and-bad-configs.md)

### Platform APIs and Self-Service Capabilities

1. [APIs as Products: Designing Your Platform Contract](./3-platform-apis-and-self-service-capabilities/01-apis-as-products-designing-your-platform-contract.md)
2. [Extending Kubernetes: Custom Resources and API Extensions](./3-platform-apis-and-self-service-capabilities/02-extending-kubernetes-custom-resources-and-api-extensions.md)
3. [CRD Design Patterns: Versioning, Status and Printer Columns](./3-platform-apis-and-self-service-capabilities/03-crd-design-patterns-versioning-status-and-printer-columns.md)
4. [Demo: Building a Custom Resource Definition](./3-platform-apis-and-self-service-capabilities/04-demo-building-a-custom-resource-definition.md)
5. [Operators Controllers: Reconcile Like a Pro](./3-platform-apis-and-self-service-capabilities/05-operators-controllers-reconcile-like-a-pro.md)
6. [Demo: Reading Operator Status and Conditions](./3-platform-apis-and-self-service-capabilities/06-demo-reading-operator-status-and-conditions.md)
7. [Workflow Orchestration: DAGs, Steps and Event Driven Automation](./3-platform-apis-and-self-service-capabilities/07-workflow-orchestration-dags-steps-and-event-driven-automation.md)
8. [Demo: Workflow Automation with Argo Workflows](./3-platform-apis-and-self-service-capabilities/08-demo-workflow-automation-with-argo-workflows.md)
9. [Platform Building Blocks: Crossplane XRs, Compositions and Functions](./3-platform-apis-and-self-service-capabilities/09-platform-building-blocks-crossplane-xrs-compositions-and-functions.md)
10. [Demo: Compositions with Crossplane](./3-platform-apis-and-self-service-capabilities/10-demo-compositions-with-crossplane.md)
11. [Crossplane Functions](./3-platform-apis-and-self-service-capabilities/11-crossplane-functions.md)
12. [Demo: Crossplane Functions](./3-platform-apis-and-self-service-capabilities/12-demo-crossplane-functions.md)
13. [Choosing the Right Engine: Operators vs Workflows vs Pipelines](./3-platform-apis-and-self-service-capabilities/13-choosing-the-right-engine-operators-vs-workflows-vs-pipelines.md)

### Observability and Operations

1. [Observability for Platforms: What to Measure and Why](./4-observability-and-operations/01-observability-for-platforms-what-to-measure-and-why.md)
2. [Metrics Architecture: Collection, Storage and Querying](./4-observability-and-operations/02-metrics-architecture-collection-storage-and-querying.md)
3. [Demo: Metrics Collection with Prometheus](./4-observability-and-operations/03-demo-metrics-collection-with-prometheus.md)
4. [Prometheus Alerting: Rules, AlertManager and Notification Routing](./4-observability-and-operations/04-prometheus-alerting-rules-alertmanager-and-notification-routing.md)
5. [Visualization and Dashboards: Turning Data into Insight](./4-observability-and-operations/05-visualization-and-dashboards-turning-data-into-insight.md)
6. [Demo: Connecting a Datasource in Grafana](./4-observability-and-operations/06-demo-connecting-a-datasource-in-grafana.md)
7. [Demo: Building Dashboards in Grafana](./4-observability-and-operations/07-demo-building-dashboards-in-grafana.md)
8. [Distributed Tracing: Context Propagation and Trace Analysis](./4-observability-and-operations/08-distributed-tracing-context-propagation-and-trace-analysis.md)
9. [Trace Visualization and Root Cause Analysis](./4-observability-and-operations/09-trace-visualization-and-root-cause-analysis.md)
10. [Demo: Tracing with OpenTelemetry and Jaeger](./4-observability-and-operations/10-demo-tracing-with-opentelemetry-and-jaeger.md)
11. [Logging for Platforms: Patterns That Scale](./4-observability-and-operations/11-logging-for-platforms-patterns-that-scale.md)
12. [Incident Playbook: Triage, Fix, Validate, Repeat](./4-observability-and-operations/12-incident-playbook-triage-fix-validate-repeat.md)

### Security and Policy Enforcement