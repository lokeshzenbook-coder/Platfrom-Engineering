> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# APIs as Products Designing Your Platform Contract

> Describes designing consumer-centric platform APIs as products to enable self-service, replace ticket-driven provisioning, and use CRDs, controllers, and workflow engines to automate and enforce guardrails.

Welcome to Section Three: Platform APIs and self-service capabilities.

If you are a platform engineer, what is your actual product? It is not the Kubernetes cluster, and it is not a wiki page. Your product is an API — a self-service interface that lets developers get what they need without waiting on your team.

In this lesson you'll learn why ticket-driven infrastructure fails at scale, how to adopt a platform-as-a-product mindset, and how to design consumer-centric platform APIs that hide infrastructure complexity. We’ll also review the building blocks you’ll use: Custom Resource Definitions (CRDs), controllers/operators, and workflow engines. Think of your platform as a product, your developers as customers, and your APIs as the contract between them.

Real-world problem: a platform team of five engineers was receiving \~40 provisioning tickets per week: namespaces, databases, SSL certs, service accounts. Tickets took between 30 minutes and two days to resolve. The backlog grew to more than 200 tickets.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/platform-team-40-tickets-per-week.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=8bf444635e93143eadcadb69f7ca78d5" alt="The image highlights that a platform team processes 40 tickets per week, implying that ticket-driven infrastructure is not scalable." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/platform-team-40-tickets-per-week.jpg" />
</Frame>

When developers get tired of waiting, they bypass the platform team and spin up their own cloud accounts without IT approval. This is shadow IT.

Shadow IT causes three major problems: security blind spots (environments aren’t reviewed), compliance violations (governance gaps), and unmanaged cloud spend.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/aws-accounts-shadow-it-security-blind-spots.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=77e6fc373a59ebf8eabeec62a0a718c6" alt="The image illustrates how different development teams manage multiple AWS accounts under the concept of &#x22;Shadow IT,&#x22; highlighting security blind spots and compliance violations." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/aws-accounts-shadow-it-security-blind-spots.jpg" />
</Frame>

<Callout icon="warning" color="#FF6B6B">
  Shadow IT is not just an organizational annoyance — it introduces real security, compliance, and cost risks. Preventing it requires replacing slow manual processes with fast, trustworthy self-service APIs.
</Callout>

The platform team itself isn’t the root cause — the ticket-driven process is. People need self-service, not more tickets.

Reality check: here’s the common workflow at many enterprises. A developer needs a PostgreSQL instance and files a ticket (or a Jira request). The ticket joins a queue. Days or weeks later it’s assigned. An operations engineer provisions and configures the database manually, then emails the requester.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/ticket-driven-infrastructure-request-flow.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=8eeb932362699fd60d2bf83511eb49bc" alt="The image illustrates the typical request flow in a ticket-driven infrastructure, highlighting a process from submission to confirmation with potential delays. It suggests that this approach doesn't scale effectively." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/ticket-driven-infrastructure-request-flow.jpg" />
</Frame>

This workflow creates predictable problems: long provisioning times, inconsistent handling (different engineers, different approaches), poor visibility into deployments/cost/ownership, and a platform team that becomes a bottleneck.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/ticket-infrastructure-challenges-provisioning-issues.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=5d47f495c7bd7b62cce53f61e7986a9a" alt="The image outlines challenges with ticket-driven infrastructure, highlighting issues such as long provisioning times, inconsistency in handling requests, and lack of visibility into deployments." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/ticket-infrastructure-challenges-provisioning-issues.jpg" />
</Frame>

Every change that requires a human in the middle does not scale. The predictable result is developers will find workarounds — leading to shadow IT. This is not a people problem; it’s a system design problem.

Compare two mindsets:

* Ticket-driven: Developer files a ticket or emails platform; a human acts as a middleman and performs manual provisioning.
* Platform-as-product: Developer calls a platform API that declares intent; automated systems provision resources without human intervention.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/infrastructure-team-vs-platform-mindset.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=48b5d945c3f53cc2c0bbd1967cc258fa" alt="The image compares two mindsets: the &#x22;Infrastructure Team Mindset&#x22; involves manual communication and work via tickets/emails, while the &#x22;Platform as Product Mindset&#x22; emphasizes automation through API calls." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/infrastructure-team-vs-platform-mindset.jpg" />
</Frame>

One human plus an API call beats three humans and a ticket queue. This mindset shift is the lesson’s core.

Three guiding principles to internalize:

* Developers are your customers: treat the developer experience as a product. If a developer must wait three days for a namespace, your product is broken.
* APIs are contracts: a clear contract sets expectations — developers know what they can request and what they will receive; platform teams know what they must support and guarantee.
* Automation needs interfaces: humans do not scale; APIs do. You can’t effectively automate a ticket queue, but you can automate an API endpoint.

The solution: define platform APIs that convert manual ticket work into a self-service portal.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/platform-without-api-importance-graphic.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=347d35921e4060b06c1755f29a81b155" alt="The image is a graphic titled &#x22;A Platform Without an API Is Just Another Silo,&#x22; emphasizing the importance of APIs through points on developers, contracts, and automation. The solution suggested is defining platform APIs to automate self-service." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/platform-without-api-importance-graphic.jpg" />
</Frame>

Reference architecture for a platform API

* Consumer layer: Developers request resources via platform APIs and only declare intent — they don’t need to know implementation details.
* Contract layer: The platform API itself. CRDs define schema; validation enforces guardrails; defaults fill unspecified values. This layer separates intent (what to create) from execution (how to create it).
* Platform logic layer: Reconciliation implemented by controllers/operators and workflow engines. These components translate the declared desired state into real infrastructure.

The contract layer should follow four principles:

* Consumer-centric: design from the developer’s perspective.
* Declarative: users declare what they want, not how to implement it.
* Guardrails: enforce validation and policy at the API layer.
* Self-documenting: schema should double as documentation.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/platform-contract-architecture-principles.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=53961be3cfae808a92a9c7cff5e68935" alt="The image illustrates design principles for a &#x22;Platform Contract Architecture,&#x22; which include being consumer-centric, declarative, having built-in guardrails, and being self-documenting. Each principle is represented with a unique icon and brief description." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/platform-contract-architecture-principles.jpg" />
</Frame>

This architecture maps directly to real implementations: CRDs define the contract, controllers/operators encode reconciliation logic, and tooling such as Crossplane handles cloud provisioning and compositions. For hands-on practice, see Learn By Doing: Crossplane.

API design approaches: comparing infrastructure-centric vs consumer-centric

* Infrastructure-centric API: Forces developers to specify cloud-specific settings (instance class, storage class, AZs). This raises the risk of improper configuration, increases cognitive load, and leaks implementation details.

Example (infrastructure-centric):

```yaml theme={null}
apiVersion: infra.internal/v1
kind: PostgresCluster
spec:
  nodeSelector: db-pool-az1
  storageClass: gp3-xfs-encrypted
  pgVersion: "15.2-alpine3.17"
  cpu: "4000m"
  memory: "16Gi"
  s3Bucket: prod-db-bkp-us-e1
```

* Consumer-centric API: Exposes intent (e.g., “medium Postgres”), hides cloud specifics, and maps intent to platform-defined defaults (instance class, backups, networking, security). This reduces mistakes and simplifies the developer experience.

Good APIs ask “What do you need?” not “How should you configure it?” The platform should handle the underlying cloud-specific lines of configuration.

Use this principle when designing CRD schemas, Crossplane compositions, and operator logic. Always evaluate whether the API exposes intent or implementation.

Core building blocks

* Custom Resource Definitions (CRDs): define your API schema and extend Kubernetes with domain-specific types (Database, Application, Team).
* Operators and Controllers: watch custom resources and reconcile actual state to desired state; when a Database CR is created, the controller provisions the database.
* Workflow Engines: orchestrate multi-step provisioning flows requiring retries, parallelism, and dependencies (e.g., create VPC → create DB → create Secret → notify team).

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/platform-api-building-blocks-illustration.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=a363c21623c48a16e2fb4b035873168c" alt="The image illustrates the building blocks of a platform API, covering Custom Resource Definitions, Operators and Controllers, and Workflow Engines, with examples and brief descriptions for each." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/platform-api-building-blocks-illustration.jpg" />
</Frame>

How these components fit together (self-service pipeline)

1. Developer creates a Custom Resource (CR) via the platform API.
2. The CRD validates input against the schema and policies (guardrails).
3. A controller/operator reconciles the CR and triggers provisioning tasks.
4. Infrastructure and related artifacts are created; the CR’s status is updated and notifications are sent to the developer.

This is the complete self-service pipeline: declarative intent → automated reconciliation → realized infrastructure.

Quick comparison table

| Concern                          | Ticket-Driven / Infrastructure-Centric | Platform-as-Product / Consumer-Centric    |
| -------------------------------- | -------------------------------------- | ----------------------------------------- |
| How developers request resources | Tickets, emails, manual forms          | Declarative API / CRs                     |
| Required knowledge               | Deep cloud-specific config             | Intent-level (size, lifecycle, purpose)   |
| Speed                            | Days to weeks                          | Minutes to hours                          |
| Consistency                      | High human variability                 | Enforced by schema, policies, controllers |
| Visibility & governance          | Fragmented                             | Centralized via API & status fields       |

<Callout icon="lightbulb" color="#1CB2FE">
  Design platform APIs with the developer experience in mind: provide concise intent-driven fields, sensible defaults, and clear validation errors. Think of the CRD schema as both the contract and the documentation.
</Callout>

Key takeaways

* Treat your platform as a product: measure developer experience and iterate.
* APIs enable self-service at scale: they prevent bottlenecks and reduce shadow IT.
* Design consumer-centric contracts that hide infrastructure complexity and enforce guardrails.
* Kubernetes gives you the building blocks: CRDs, controllers/operators, and workflow engines to implement self-service.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/key-takeaways-platforms-apis-kubernetes.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=77ccbe3d290f2d21c66fab10e7f63004" alt="The image is a slide titled &#x22;Key Takeaways,&#x22; listing four points on treating platforms as products, using APIs for scalability, designing consumer-focused interfaces, and utilizing Kubernetes building blocks." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Platform-APIs-and-Self-Service-Capabilities/APIs-as-Products-Designing-Your-Platform-Contract/key-takeaways-platforms-apis-kubernetes.jpg" />
</Frame>

Platform engineering is about making the right thing the easy thing. If filing a ticket is easier than using the platform, developers will keep filing tickets — so make the right thing the easy thing.

References and further reading

* Learn By Doing: Crossplane — [https://learn.kodekloud.com/user/courses/learn-by-doing-crossplane](https://learn.kodekloud.com/user/courses/learn-by-doing-crossplane)
* Kubernetes CRD documentation — [https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
* Operator pattern — [https://kubernetes.io/docs/concepts/extend-kubernetes/operator/](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
* Consider workflow engines such as Argo Workflows or Temporal for orchestrating multi-step provisioning flows.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/756ffaae-767b-4743-9724-c05d3fbf9a18/lesson/79029d22-eae3-479b-835b-ba6f83bc0004" />
</CardGroup>
