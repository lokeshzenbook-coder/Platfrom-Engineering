> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Trace Visualization and Root Cause Analysis

> Practical guide to using Jaeger traces and OpenTelemetry to read waterfalls, identify latency patterns, follow a triage workflow, inspect spans, and correlate traces with logs and metrics.

This lesson assumes you already understand traces, spans, OpenTelemetry, and Jaeger. Here we make those concepts practical: how to read a Jaeger waterfall, spot latency patterns, and follow a repeatable triage workflow that takes you from “something is slow” to “this is exactly why.”

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/jaeger-learning-objectives-latency-traces.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=63cf86d3999f88653989d535c7e4ff5f" alt="The image lists five learning objectives related to using Jaeger for identifying critical paths, recognizing latency patterns, following a triage workflow, connecting traces to metrics and logs, and using Jaeger effectively." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/jaeger-learning-objectives-latency-traces.jpg" />
</Frame>

## Core skill: reading the waterfall

The waterfall (timeline) view is your primary diagnostic tool. Each row is a span; each bar shows when the operation started and how long it ran. Read it like this:

* Start at the root span (top).
* Find the widest child span — that’s the current critical path.
* Drill into that child and repeat until you find the operation that dominates latency.

Example: an API gateway span lasts 100 ms. Inside it, `auth-service` takes 12 ms while `order-service` takes 82 ms — clearly most time is spent in the order service.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/jaeger-trace-waterfall-chart-latencies.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=273d98b2bbd90a9df554fac6e7f6e74c" alt="The image is a waterfall chart illustrating the timeline of different services in a Jaeger trace, highlighting API-gateway, Auth-service, Order-service, Inventory-svc, Payment-svc, and Payment-gv with their respective latencies. A note indicates that the payment-svc is a critical path, making up 70% of the total latency." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/jaeger-trace-waterfall-chart-latencies.jpg" />
</Frame>

## Five common latency patterns

Most latency incidents fit into one of these patterns. Commit these to memory — they speed up triage.

| Pattern               | Symptoms (what you see in the waterfall)                                                    | Typical fixes                                                              |
| --------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Serial bottleneck     | One span dominates total request time (e.g., single DB query is 800 ms of a 900 ms request) | Cache results, optimize query, offload to async jobs                       |
| Fan-out amplification | A parent calls many children; total equals the slowest child                                | Add timeouts, degrade non-essential calls, use hedging or fallbacks        |
| Retry storm           | Repeated identical spans; retries add up to large total latency                             | Fix underlying failure, tune retry backoff/limits, add circuit breakers    |
| Missing spans (gaps)  | Parent duration > sum(child durations) — unexplained interval                               | Add instrumentation, capture queue/wait events, log connection/dns timings |
| Error cascade         | Failures propagate and cause upstream retries or failures                                   | Fix root cause, add timeouts/circuit breakers, graceful degradation        |

After the next section on the first three patterns, see the illustrated example below.

1. Serial bottleneck\
   One span dominates the timeline. Example: a DB call is 800 ms in a 900 ms request. Fixes: caching, asynchronous processing, or optimizing the slow operation.

2. Fan-out amplification\
   A service issues many parallel calls. The slowest child determines total latency. Fixes: set timeouts, degrade functionality, add fallbacks.

3. Retry storm\
   A failed call is retried multiple times. Each retry consumes its timeout — three retries with 2 s timeouts add 6 s to latency. Fixes: address the root failure, implement exponential backoff, limit retries, add circuit breakers.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/latency-patterns-serial-bottleneck-fixes.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=5d2edddc130a131d6e14eba6a8153c0d" alt="The image lists five common latency patterns: Serial Bottleneck, Fan-Out Amplification, Retry Storm, Missing Spans (Gaps), and Error Cascade, with a focus on Retry Storm and suggestions for fixes." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/latency-patterns-serial-bottleneck-fixes.jpg" />
</Frame>

4. Missing spans (gaps)\
   The parent span duration is longer than the sum of child spans. Example: parent shows 500 ms but children add to 200 ms — the remaining 300 ms is usually uninstrumented work (queue wait, DNS lookup, connection pool wait). Fix: add instrumentation or capture timing for the missing operations.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/latency-patterns-serial-bottleneck-error-cascade.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=030444853174de866bec64c316d18626" alt="The image lists five common latency patterns: Serial Bottleneck, Fan-Out Amplification, Retry Storm, Missing Spans (Gaps), and Error Cascade, with a note about unexplained gaps in the timeline and a suggested fix." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/latency-patterns-serial-bottleneck-error-cascade.jpg" />
</Frame>

5. Error cascade\
   One service fails, its caller retries, and upstream callers retry — a tree of failures. Fix the root error and add circuit breakers, timeouts, and graceful degradation to stop the cascade.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/five-latency-patterns-with-corrections.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=037227ea37c50c29f42d7bce729308a8" alt="The image illustrates &#x22;Five Common Latency Patterns,&#x22; including Serial Bottleneck, Fan-Out Amplification, Retry Storm, Missing Spans (Gaps), and Error Cascade, with corrective actions suggested below." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/five-latency-patterns-with-corrections.jpg" />
</Frame>

## A systematic triage workflow

Use a repeatable process rather than guessing. This workflow takes you from a symptom to a concrete root cause.

1. Search and filter
   * Open Jaeger, select the relevant service and operation.
   * Set a minimum duration filter so you only see slow traces (e.g., `> 1s`).
   * Use tags to narrow results, for example: `http.status_code = 500`.

2. Sort and pick
   * Sort traces by duration and pick the slowest trace.
   * When possible, find a corresponding fast trace for the same operation and compare them.

3. Identify and drill down
   * In the waterfall, find the widest span and click it.
   * Inspect the span’s tags, logs/events, and process info.
   * Follow the critical path: at each level, pick the widest child.

4. Correlate with metrics and logs
   * Use the service name, pod/host, and timestamp from the span to search metrics (Prometheus/Grafana) and logs (ELK/Loki) for that instance.
   * Look for CPU, memory, thread/connection pool saturation, GC pauses, or network errors at the same timestamp.

Pro tip: always compare a slow and a fast trace of the same operation — the delta usually points to the root cause.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/systematic-triage-workflow-process-diagram.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=d7d9a047ec3201cf4005560fcf466417" alt="The image illustrates a &#x22;Systematic Triage Workflow&#x22; process, detailing steps such as Search, Identify, Correlate, Filter, and Drill Down for analyzing and troubleshooting service requests. It includes a pro tip about comparing slow and fast traces to identify root causes." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/systematic-triage-workflow-process-diagram.jpg" />
</Frame>

## Span details: what to look for

When you click a span in Jaeger, examine these sections — they tell you “why,” not just “where”:

* Tags — e.g., `http.method`, `http.status_code`, `db.statement`. These describe the request and its outcome.
* Logs / events — timestamped internal events like `connection acquired at 0 ms` or `query started at 5 ms`. These reveal timing inside the span.
* Process information — hostname, pod name, namespace; these identify which instance emitted the span so you can correlate logs/metrics.

The waterfall shows where the problem is; span details show why it happened.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/span-data-tags-logs-process-info.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=6a5032b019a50ba6967c6272dcb4643b" alt="The image provides details of what to look for in span data, including tags, logs/events, and process information related to an operation, such as HTTP method, status code, timestamps of key events, and process identifiers." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/span-data-tags-logs-process-info.jpg" />
</Frame>

## Practical tips that save time

* Set a minimum duration when searching to avoid triaging healthy traces. Example: `duration > 1s` or `> 500ms` depending on your SLA.
* Use tag filters to focus on errors: `http.status_code = 500`.
* At each level of the waterfall, follow the widest child (the critical path).
* Watch for gaps: if children don’t add up to the parent, instrument the missing interval (queue, DNS, connection pool).
* Repeated adjacent spans usually indicate retries; inspect the first failure event and the retry policy.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/jaeger-tips-tricks-search-duration.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=119df956952195d9dcb6a5db2b6f3485" alt="The image provides tips and tricks for using Jaeger, focusing on setting minimum duration, using tags in search, adjusting time range, and using trace compare." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/jaeger-tips-tricks-search-duration.jpg" />
</Frame>

<Callout icon="lightbulb" color="#1CB2FE">
  Compare a slow trace with a fast trace of the same operation — the difference is often the quickest path to the root cause.
</Callout>

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/gQpDYNH1QRx_0eCB/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/jaeger-tips-retries-critical-path-brain.jpg?fit=max&auto=format&n=gQpDYNH1QRx_0eCB&q=85&s=f6e40df891a46a1666eb8cc2783c8c03" alt="The image provides tips and tricks for using Jaeger in practice, including noting span count, looking for retries, following the critical path, and checking for gaps, all centered around a graphic of a stylized brain." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Observability-and-Operations/Trace-Visualization-and-Root-Cause-Analysis/jaeger-tips-retries-critical-path-brain.jpg" />
</Frame>

<Callout icon="warning" color="#FF6B6B">
  If spans are missing (gaps), don’t assume the slow child is the cause — the uninstrumented interval might be the true source. Add instrumentation before concluding.
</Callout>

## Quick checklist for live triage

* Did you filter by duration and tags?
* Did you compare slow vs fast traces?
* Did you follow the critical path to the widest spans?
* Did you inspect tags, events, and the process that emitted the span?
* Did you correlate with metrics/logs for the same timestamp and instance?

## Wrap-up

* The waterfall is your primary diagnostic tool: find the widest bar and follow it.
* Remember the five latency patterns — they cover almost every issue you’ll encounter.
* Use the triage workflow (search, filter, identify, drill down, correlate) instead of ad hoc guessing.
* Span tags, logs/events, and process details point you to the next action and the fix.

## Links and references

* Jaeger documentation: [https://www.jaegertracing.io/docs/](https://www.jaegertracing.io/docs/)
* OpenTelemetry: [https://opentelemetry.io/](https://opentelemetry.io/)
* Distributed tracing best practices: [https://opentelemetry.io/docs/concepts/best-practices/](https://opentelemetry.io/docs/concepts/best-practices/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/9bd090c8-8d99-4742-b50c-ae63e516e6b9/lesson/1b7e7dc8-f5b0-4683-b739-8aa9d0a7ebc1" />
</CardGroup>
