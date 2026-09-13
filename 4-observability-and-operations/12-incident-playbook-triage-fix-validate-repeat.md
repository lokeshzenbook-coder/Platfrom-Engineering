> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Incident Playbook Triage Fix Validate Repeat

> A five-phase incident response playbook guiding teams from alert to validation using observability tools and safe reversible fixes to reduce mean time to recovery

This capstone ties together the observability and incident-response concepts covered earlier into a single, repeatable flow. Use this five-phase playbook to go from “something’s broken” to “it's fixed and confirmed” and to map each phase to the right tools and actions.

A logistics company had all the right tools — Prometheus, Grafana, Jaeger, Loki, and Alertmanager — but no playbook. At 2:00 a.m. their shipping API went down. The on-call engineer opened Grafana, saw red everywhere, panicked, and started restarting pods at random. Forty-five minutes later they discovered the root cause: an HPA had scaled the API to zero because a custom metric was broken.

After implementing a simple playbook, the next similar incident took four minutes instead of forty-five. The tools were the same; the outcome changed because they had a process.

Use this playbook for every incident, every time.

* Alert: Something triggered (Alertmanager, PagerDuty, Slack, or a custom report). It tells you which metric crossed a threshold and when.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/five-phase-incident-playbook-steps.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=fe049e3b2c028bb40ad4a6a9e0c974f4" alt="The image outlines a &#x22;Five-Phase Incident Playbook&#x22; consisting of steps: Alert, Assess, Investigate, Fix, and Validate. A description under &#x22;Alert&#x22; mentions triggers like PagerDuty and Slack, detailing what and when a metric threshold was crossed." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/five-phase-incident-playbook-steps.jpg" />
</Frame>

* Assess: Before you touch anything, spend a short, focused time (aim \~60 seconds) to understand the scope: how bad it is, who is affected, and whether it's partial or total.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/five-phase-incident-playbook-assess.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=f08a39203d0d27c770c7660556b3fe08" alt="The image depicts a five-phase incident playbook with steps: Alert, Assess, Investigate, Fix, and Validate, highlighting &#x22;Assess&#x22; with a note on evaluating scope, impact, and urgency." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/five-phase-incident-playbook-assess.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Spend a minute to assess before you act. Skipping assess is the most common cause of wasted time and harmful changes.
</Callout>

* Investigate: Dig into the root cause in a focused, methodical order: metrics first, then traces, then logs. Follow a triage workflow to narrow scope gradually.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/five-phase-incident-playbook-diagram.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=d0bcedd46392d4e19246d654e18e0a89" alt="The image illustrates &#x22;The Five-Phase Incident Playbook,&#x22; detailing phases: Alert, Assess, Investigate, Fix, and Validate. There is additional text guiding the investigation process with steps like searching and correlating." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/five-phase-incident-playbook-diagram.jpg" />
</Frame>

* Fix: Apply the minimum-change necessary to restore service: rollback, scale up, adjust configuration, or restart. Prefer a reversible, conservative change that buys time to investigate further.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/five-phase-incident-playbook-steps-2.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=9d2df0f4d83eddaf93529514ee918fb3" alt="The image presents a &#x22;Five-Phase Incident Playbook&#x22; consisting of five steps: Alert, Assess, Investigate, Fix, and Validate, with detailed instructions under the &#x22;Fix&#x22; phase." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/five-phase-incident-playbook-steps-2.jpg" />
</Frame>

* Validate: Don't close the incident until you confirm the fix worked. Watch dashboards and verify stability for a few minutes.

Tools map naturally to the phases:

* Alertmanager, PagerDuty, Slack — alerts and notification (Alert, Validate)
* Grafana — dashboards and runbooks (Assess, Validate)
* Prometheus — metrics and PromQL (Investigate)
* Jaeger — distributed traces (Investigate)
* Loki / kubectl logs — logs and pod-level details (Investigate)
* kubectl, GitOps, Helm — apply safe, reversible fixes (Fix)

Now let’s dig into each phase, starting with Assess — the step most engineers skip or rush.

Assess: four quick questions

* What’s broken? Read the alert and identify the affected service, endpoint, or pod.
* How bad is it? Is the outage partial (a single endpoint or region) or total?
* Who’s affected? One customer, a region, or everyone?
* What changed recently? A deployment or config change within the alert window is a prime suspect.

Investigation: use the observability funnel (metrics → traces → logs)

* Metrics first: Inspect the alert’s PromQL expression and open Grafana. Confirm the alert timestamps, pattern (sudden vs gradual), and compare to baseline (same time yesterday, same weekday).
* Traces next: Search Jaeger for slow or error traces. Use minimal-duration filters, inspect the waterfall, and find the longest-duration span or failing service.
* Logs last: Query logs by service and timestamp, or by trace ID to pinpoint the exact error message.

Each pillar narrows the search: metrics detect and scope, traces locate the problematic service or interaction, and logs explain the root cause.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/phase-3-investigate-observability-pillars.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=a92f5ac4059fe4604141b426cfd11af0" alt="The image outlines &#x22;Phase 3: Investigate With ALL Three Pillars&#x22; in observability, featuring metrics, traces, and logs using tools like Prometheus, Grafana, Jaeger, Loki, and Kubectl logs." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/phase-3-investigate-observability-pillars.jpg" />
</Frame>

Measure platform performance over time with DORA metrics. These give you objective targets to reduce MTTR and improve reliability.

| DORA Metric                  | What it measures                          | Example target |
| ---------------------------- | ----------------------------------------- | -------------- |
| Deployment Frequency         | How often you ship to production          | Daily or more  |
| Lead Time for Changes        | Time from commit to production            | Under 1 hour   |
| Change Failure Rate          | % of deployments causing failures         | Under 5%       |
| Mean Time to Recovery (MTTR) | Time to restore service after an incident | Under 1 hour   |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/platform-efficiency-dora-metrics-summary.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=b97bade7c5684dff8e5944308bafe143" alt="The image lists key platform efficiency metrics for measuring platform health, featuring four DORA metrics: Deployment Frequency, Lead Time for Changes, Change Failure Rate, and MTTR. Each metric includes a brief explanation and a target goal." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Incident-Playbook-Triage-Fix-Validate-Repeat/platform-efficiency-dora-metrics-summary.jpg" />
</Frame>

Phase 4 — Fix: four safe strategies

* Rollback
  * When to use: a recent deployment caused the incident and assessment/investigation confirm it.
  * Why: fastest, lowest-risk path to restore service.
  * Example:

```bash theme={null}
kubectl rollout undo deployment app
```

* Scale up
  * When to use: load exceeds capacity (CPU/memory/requests throttled) and you need breathing room.
  * Why: buys time to investigate whether load is legitimate or anomalous.
  * Example:

```bash theme={null}
kubectl scale deployment app --replicas=5
```

* Config fixes
  * When to use: misconfigured DB URL, expired secret, or bad environment variable.
  * How: update ConfigMap/Secret and trigger a restart or rolling update.
  * Example:

```bash theme={null}
kubectl rollout restart deployment app
```

* Resource adjustments (scheduling failures)
  * When to use: pods Pending because nodes lack allocatable resources.
  * How: free resources, tune requests/limits, or adjust quotas for targeted workloads.
  * Note: avoid broad, cluster-wide changes during an active incident; prefer targeted, minimal changes.

<Callout icon="warning" color="#FF6B6B">
  During an incident prefer reversible, minimal changes. Large, unreviewed edits or cluster-wide operations increase risk and can create cascading failures.
</Callout>

Quick phase-to-tool reference

| Phase       | Common tools / actions                                             |
| ----------- | ------------------------------------------------------------------ |
| Alert       | Alertmanager, PagerDuty, Slack; check alert details and timestamps |
| Assess      | Grafana dashboards, runbooks, deployment history                   |
| Investigate | Prometheus (PromQL), Jaeger traces, Loki/kubectl logs              |
| Fix         | kubectl, Helm, GitOps rollbacks, targeted scaling or config edits  |
| Validate    | Grafana dashboards, synthetic tests, user-facing checks            |

Key takeaways

* Follow the playbook every time: Alert → Assess → Investigate → Fix → Validate.
* Never skip the Assess phase; the 60-second check prevents wasted effort.
* Use the three observability pillars as a funnel: metrics detect, traces locate, logs explain.
* Prefer minimal, reversible fixes that restore service quickly and enable deeper post-incident investigation.
* Track DORA metrics to measure platform health and quantify improvements to MTTR.

Links and references

* [Prometheus Documentation](https://prometheus.io/docs/)
* [Grafana Docs](https://grafana.com/docs/)
* [Jaeger Tracing](https://www.jaegertracing.io/docs/)
* [Loki Logging](https://grafana.com/oss/loki/)
* [Kubernetes kubectl reference](https://kubernetes.io/docs/reference/kubectl/)

Use this playbook as your default incident response flow. With consistent practice and post-incident learning, the same tools will yield much faster, safer outcomes.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/9bd090c8-8d99-4742-b50c-ae63e516e6b9/lesson/a32b40e5-e639-40ca-bd3a-00bb8cf752cf" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/9bd090c8-8d99-4742-b50c-ae63e516e6b9/lesson/e88d8670-511b-4f7a-b1ba-5fa0421e2bbe" />
</CardGroup>
