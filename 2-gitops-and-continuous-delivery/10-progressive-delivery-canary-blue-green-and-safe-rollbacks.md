> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Progressive Delivery Canary Blue Green and Safe Rollbacks

> Explains progressive delivery strategies like canary and blue green deployments, risks of standard rollouts, and using Argo Rollouts for metric driven gradual rollouts and safe rollbacks.

Traditional Kubernetes deployments are typically all-or-nothing: you push a change, Argo CD (or another delivery tool) syncs, and the new version rolls out to all users. If the change contains a bug or a performance regression, every user can be affected — a 100% blast radius. Progressive delivery reduces that risk by exposing a smaller fraction of real traffic to new code, validating behavior with metrics, and automating safe rollbacks.

In this lesson we will:

* Compare standard, blue-green, and canary strategies.
* Explain when to use each strategy.
* Show how Argo Rollouts enables metric-driven progressive delivery on Kubernetes.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/learning-objectives-deployments-list.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=e7b6960495bfba665e75cd5edb43f1f2" alt="The image displays a list of learning objectives related to deployments, including understanding risks, comparing strategies, and determining usage. It has a colorful sidebar with numbered points." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/learning-objectives-deployments-list.jpg" />
</Frame>

## Why progressive delivery matters — a real-world example

A social-media platform used the default Kubernetes rolling update to push a new recommendation algorithm. The rollout reached 100% of users in \~90 seconds. The algorithm contained a memory leak that only manifested under production load. Latencies rose from \~50 ms to \~800 ms, and the degradation persisted for 15 minutes while the team detected and rolled back the change — impacting every user.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/memory-leak-impact-response-times-traffic.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=97ac5b1c650449bad6eef4d77cc4a49b" alt="The image illustrates the impact of a memory leak on production traffic, resulting in increased response times from 50 ms to 800 ms." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/memory-leak-impact-response-times-traffic.jpg" />
</Frame>

Standard Kubernetes rolling updates expose all users at once and have several limitations:

* 100% blast radius if the deployment is faulty.
* No native traffic control for staged exposure.
* Rollbacks can be slow because pods must terminate, be rescheduled, and become ready.
* No built-in automated checks tied to production metrics.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/kubernetes-all-or-nothing-drawbacks.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=e70e533c3e29425b5699d29a76efc17b" alt="The image illustrates the drawbacks of all-or-nothing deployments in Kubernetes, highlighting issues such as a 100% blast radius, slow rollback, and lack of traffic control." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/kubernetes-all-or-nothing-drawbacks.jpg" />
</Frame>

Progressive delivery strategies (blue-green and canary) reduce exposure and validate changes in production. Tools such as Argo Rollouts and Flagger implement these patterns on Kubernetes and integrate with traffic control layers (service mesh, Gateway API, cloud load balancers).

## Blue-green deployments

Blue-green is the simplest progressive-delivery pattern: maintain two identical environments. Blue is the active environment, and green is the new version running in parallel but receiving no production traffic. Deploy to green, run smoke tests and validations, and when confident, flip the router/load-balancer or Service selector from blue to green. Rollback is an instant switch back to blue.

Strengths

* Instant rollback by switching routing to the previous environment.
* Zero-downtime deployments for stateless services when readiness and routing are handled correctly.
* Full testing of the new version before exposing users.

Challenges

* Requires roughly double the resources during deployment.
* The cutover is all-or-nothing; traffic moves 100% at the switch.
* Database migrations can be complicated because both versions may need compatibility.

Blue-green is a strong choice for stateless services that need instant rollback and where resource overhead is acceptable.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/blue-green-deployments-traffic-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=53fa781a41d2a2a6e935a34ab3837ec1" alt="The image illustrates a &#x22;Blue-Green Deployments&#x22; concept with two environments, showing &#x22;Blue (v1)&#x22; as active with 100% traffic and &#x22;Green (v2)&#x22; as idle with 0% traffic." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/blue-green-deployments-traffic-diagram.jpg" />
</Frame>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/blue-green-deployments-presentation-slide.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=984110ebabf5c54b900b9dd6017c48a3" alt="The image is a presentation slide about Blue-Green Deployments, outlining strengths such as instant rollback and zero-downtime deployment, and challenges like requiring double resources and tricky database migrations." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/blue-green-deployments-presentation-slide.jpg" />
</Frame>

## Canary deployments

Canary deployments gradually shift traffic to the new version while continuously monitoring production metrics. This lets you validate the new release against real user traffic and abort early if something goes wrong.

A typical canary flow:

1. Route a small percentage (e.g., 5%) to the canary.
2. Run automated analysis on latency, error rates, and business metrics.
3. If checks pass, promote to larger percentages (e.g., 25%, 50%).
4. If any analysis step fails, roll the canary back to 0%.

Strengths

* Gradual exposure reduces blast radius.
* Promotions can be automated and metric-driven.
* Real user traffic provides meaningful validation.

Challenges

* Requires traffic splitting support (ingress, service mesh, or load balancer).
* More operational complexity to set up and maintain.
* Requires an observability stack (Prometheus, tracing, logs) to drive decisions.

Canary is ideal for high-traffic applications where validating changes against real user behavior is critical.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/canary-deployments-traffic-shifting-diagram.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=94f900198145f4d1ff8aa85d3c7d0f66" alt="The image explains canary deployments, showing how traffic is gradually shifted from a stable version to a canary version while monitoring metrics. If metrics are okay, traffic increases incrementally; if bad, it automatically rolls back." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/canary-deployments-traffic-shifting-diagram.jpg" />
</Frame>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/canary-deployments-strengths-challenges-comparison.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=e6c2f3c64c7922f3b6e763ae7a024814" alt="The image is a comparison of the strengths and challenges of canary deployments, highlighting gradual risk exposure and metric-driven promotion as strengths, and traffic splitting and complexity as challenges." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/canary-deployments-strengths-challenges-comparison.jpg" />
</Frame>

## Argo Rollouts — Kubernetes-native progressive delivery

Argo Rollouts is a custom controller that replaces the Deployment resource with a Rollout CRD. It supports both blue-green and canary strategies and provides first-class analysis and traffic management integrations.

Key behaviors for canary rollouts:

* Sequential steps: adjust weight, pause, or run analysis.
* `setWeight`: change the percentage of traffic sent to the canary.
* `pause`: wait for a duration or for manual approval.
* `analysis`: run automated checks (Prometheus queries, webhooks) to decide promotion or rollback.

Example Rollout (canary) with steps and an analysis step:

```yaml theme={null}
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v2
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause:
            duration: 10s
        - setWeight: 25
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 100
```

An AnalysisTemplate ties the rollout to real metrics. Example: a Prometheus-based success-rate check that requires >99% success:

```yaml theme={null}
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: request-success-rate
      interval: 30s
      count: 3
      successCondition: result > 0.99
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{job="my-app",status!~"5.."}[1m]))
            /
            sum(rate(http_requests_total{job="my-app"}[1m]))
```

With this configuration, Argo Rollouts will only advance the canary after the analysis passes. If the analysis fails, it can automatically abort and roll back or pause for manual review.

<Callout icon="lightbulb" color="#1CB2FE">
  [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) requires its CRDs and controller to be installed in the cluster. It integrates with traffic control solutions (e.g., [Istio](https://istio.io), Gateway API (`https://gateway-api.sigs.k8s.io/`), and cloud load balancers) to implement traffic splitting, and pairs well with [Argo CD](https://learn.kodekloud.com/user/courses/gitops-with-argocd) for visualizing rollouts in the UI.
</Callout>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/argo-rollouts-canary-blue-green-strategies.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=cb9ef2db9ac28ec995fa335feec78d90" alt="The image highlights key features of &#x22;Argo Rollouts,&#x22; including Canary and Blue-Green strategies, traffic management, analysis templates, and ArgoCD integration." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/argo-rollouts-canary-blue-green-strategies.jpg" />
</Frame>

Argo Rollouts key features:

* Native support for canary and blue-green strategies.
* Integrations with Istio, Gateway API, and cloud load balancers for traffic split.
* AnalysisTemplates to automate checks using Prometheus, webhooks, or custom providers.
* Integration with Argo CD for visibility and GitOps workflows.

## Strategy comparison

| Strategy                |    Risk / Blast radius |                         Rollback speed | Traffic control            | Best for                                           |
| ----------------------- | ---------------------: | -------------------------------------: | -------------------------- | -------------------------------------------------- |
| Standard rolling update |            High (100%) |                       Slow (pod churn) | None                       | Simple apps where rapid rollbacks are not critical |
| Blue-green              |      Low after cutover |               Instant at routing level | Full cutover (100% switch) | Stateless services needing instant rollback        |
| Canary                  | Low (gradual exposure) | Fast (can abort and scale down canary) | Fine-grained (percentages) | High-traffic apps needing metric-driven validation |

## Summary

* Standard Kubernetes rollouts expose all users at once and have a 100% blast radius.
* Blue-green provides an instant switch and rollback at the cost of running duplicate environments and complex migrations.
* Canary provides gradual exposure with metric-driven promotions and safe automatic rollbacks, but requires traffic splitting and observability.
* Argo Rollouts is a Kubernetes-native controller that implements both patterns and adds analysis templates for automated promotion and rollback.

Progressive delivery reduces blast radius by validating changes with real user traffic and automated analysis, allowing teams to ship faster with lower risk.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/fWoje-V05mxPGY9H/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/key-takeaways-deployment-strategies-kubernetes.jpg?fit=max&auto=format&n=fWoje-V05mxPGY9H&q=85&s=cace72b5741cde338c2099d27df77ac8" alt="The image shows &#x22;Key Takeaways&#x22; on deployment strategies, including standard rollouts, blue-green deployments, canary deployments, and using Argo Rollouts with Kubernetes." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/GitOps-and-Continuous-Delivery/Progressive-Delivery-Canary-Blue-Green-and-Safe-Rollbacks/key-takeaways-deployment-strategies-kubernetes.jpg" />
</Frame>

## Links and references

* Argo Rollouts: [https://argoproj.github.io/argo-rollouts/](https://argoproj.github.io/argo-rollouts/)
* Argo CD: [https://learn.kodekloud.com/user/courses/gitops-with-argocd](https://learn.kodekloud.com/user/courses/gitops-with-argocd)
* Flagger: [https://flagger.app/](https://flagger.app/)
* Istio: [https://istio.io](https://istio.io)
* Gateway API: [https://gateway-api.sigs.k8s.io/](https://gateway-api.sigs.k8s.io/)
* Prometheus: [https://prometheus.io](https://prometheus.io)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/dff5382b-dbe7-4cac-bd2b-d5a47028945e/lesson/225de7cf-0ffe-47dc-a96f-cc5af3ffc44a" />
</CardGroup>
