# Kubernetes ResourceQuota & LimitRange --- Study Notes

## 1. Learning Objectives

The material focuses on four main objectives:

1.  Understand the challenges in selecting appropriate **ResourceQuota**
    and **LimitRange** values.
2.  Determine suitable default **CPU and memory requests/limits** for
    different workloads.
3.  Design **environment-specific quota tiers** for:
    -   Development
    -   Staging
    -   Production
4.  Use actual **resource-usage data** to continuously refine quotas and
    limits.

------------------------------------------------------------------------

# 2. ResourceQuota --- What Problem Does It Solve?

A **ResourceQuota** controls the total amount of resources that can be
consumed by objects within a Kubernetes namespace.

### Example

**Resource Quota: 2 CPU per namespace**

Suppose a namespace has:

``` text
CPU quota = 2 CPU
```

### Data Pipeline Team

``` text
Required CPU = 8 CPU
Namespace quota = 2 CPU
```

Result:

``` text
Deployment → FAILED
Reason → Cannot allocate required CPU
```

### Lightweight API Team

``` text
Actual CPU usage = 200m
Namespace quota = 2 CPU
```

Result:

``` text
API → HEALTHY
```

### Key lesson

A quota that works for one workload may be inappropriate for another.

> **ResourceQuota must be designed according to workload requirements
> rather than using one universal value.**

------------------------------------------------------------------------

# 3. One Size Doesn't Fit All

Different applications have very different resource profiles.

  Workload            CPU   Memory
  --------------- ------- --------
  Web/API            100m    128Mi
  Data Pipeline     2 CPU      4Gi
  ML Training       8 CPU     32Gi

A single default may be:

-   Too high for a lightweight API
-   Too low for a data-processing application
-   Extremely low for ML training

Therefore:

> **Resource configuration should be based on workload
> characteristics.**

------------------------------------------------------------------------

# 4. Important Questions

## Question 1 --- What should the default CPU request be?

There is **no universal answer**.

It depends on:

-   Application type
-   Traffic
-   CPU intensity
-   Environment
-   Number of replicas
-   Scaling behavior

Example:

``` text
API Gateway → 50m CPU
Data Pipeline → 2 CPU
ML Training → 8 CPU
```

Using 2 CPU as the default for everything wastes resources for small
APIs.

Using 50m for everything can cause performance problems for
compute-heavy applications.

------------------------------------------------------------------------

## Question 2 --- How much memory quota per namespace is fair?

The answer depends on:

-   Services running in the namespace
-   Memory requirements
-   Number of pods
-   Environment
-   Application intensity
-   Expected workload

Development environments may need lower memory quotas, while production
environments generally require more capacity.

------------------------------------------------------------------------

## Question 3 --- Should we limit pods to 10 or 100 per namespace?

There is no universal answer.

It depends on the **deployment pattern**.

### Single-service team

Example:

``` text
3 pods
```

A relatively small pod limit may be sufficient.

### Microservices team

A microservices environment may have many services, each with multiple
replicas.

If autoscaling is enabled:

``` text
3 pods → 10 pods → 20 pods → ...
```

A limit of 10 pods could unnecessarily prevent legitimate scaling.

### Key principle

> **Pod limits should consider both the number of services and
> autoscaling behavior.**

------------------------------------------------------------------------

# 5. What Happens When Defaults Are Too Restrictive?

This is an important governance problem.

## Default too low → Friction

``` text
Developer deploys application
        ↓
Resource limit reached
        ↓
Deployment fails
        ↓
Developer needs exception
        ↓
Development slows down
```

This creates **developer friction**.

## Default too high → Governance Defeated

``` text
Large quota
     ↓
Applications consume more resources
     ↓
Little resource control
     ↓
Over-allocation
     ↓
Governance becomes ineffective
```

### Goal

Find a balanced resource policy:

``` text
Too Low
  ↓
Friction

     OPTIMAL RANGE

Too High
  ↓
Governance Defeated
```

------------------------------------------------------------------------

# 6. Workload Mix Matters

Organizations often run a mixture of:

-   Light services
-   Medium workloads
-   Heavy compute workloads

Example:

  Workload            CPU   Memory
  --------------- ------- --------
  Web API            100m    128Mi
  Data Pipeline     2 CPU      4Gi
  ML Training       8 CPU     32Gi

Therefore:

> **Resource policies should recognize workload differences.**

------------------------------------------------------------------------

# 7. Environment-Specific Quotas

A major recommendation is to create **different quota tiers for
different environments**.

## Development --- Low Quotas

Development environments usually have:

-   Many experiments
-   Temporary workloads
-   Lower resource requirements

``` text
DEV
 ↓
Lower quotas
 ↓
Many experiments
```

The goal is to prevent unnecessary resource consumption while still
allowing developers to experiment.

------------------------------------------------------------------------

## Staging --- Moderate Quotas

Staging should resemble production but generally doesn't need the same
resource allocation.

``` text
STAGING
 ↓
Production-like environment
 ↓
Moderate quotas
```

This allows realistic testing without allocating full production
capacity.

------------------------------------------------------------------------

## Production --- Higher Quotas + Strict Limits

Production workloads generally require more resources.

``` text
PRODUCTION
 ↓
Higher quotas
 ↓
Strict limits
```

The purpose is to provide enough capacity while maintaining strong
governance and preventing uncontrolled consumption.

------------------------------------------------------------------------

# 8. Recommended Environment Model

  Environment      Quota      Purpose
  ---------------- ---------- ------------------------------------
  **Dev**          Low        Experiments and development
  **Staging**      Moderate   Production-like testing
  **Production**   Higher     Real workloads + strong governance

### Easy memory trick

``` text
DEV       → Low
STAGING   → Medium
PROD      → High + Strict
```

------------------------------------------------------------------------

# 9. ResourceQuota vs LimitRange

These Kubernetes objects solve different problems.

## ResourceQuota

Controls the **total resource consumption of a namespace**.

Think:

> **"How much can this namespace consume in total?"**

Example:

``` yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

This establishes a namespace-level boundary.

------------------------------------------------------------------------

## LimitRange

Controls resource requests/limits at the **individual container/pod
level**.

Think:

> **"How much can one container/pod request or use?"**

Example:

``` yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: 500m
      memory: 512Mi
```

Here:

``` text
defaultRequest → CPU/memory requested if not specified
default        → CPU/memory limit used if not specified
```

------------------------------------------------------------------------

# 10. How ResourceQuota and LimitRange Work Together

Think of the architecture like this:

``` text
                 Kubernetes Cluster
                        │
                 ┌──────┴──────┐
                 │  Namespace  │
                 └──────┬──────┘
                        │
             ┌──────────┴──────────┐
             │                     │
       ResourceQuota          LimitRange
             │                     │
       Namespace total       Per-container
       resource budget       defaults/limits
```

### ResourceQuota

Controls:

``` text
TOTAL namespace consumption
```

### LimitRange

Controls:

``` text
INDIVIDUAL container/pod resources
```

Together they provide stronger resource governance.

------------------------------------------------------------------------

# 11. Framework for Choosing Values

Instead of randomly selecting:

``` text
CPU = 500m
Memory = 512Mi
Pods = 10
```

use a systematic approach.

## Step 1 --- Classify workloads

Identify:

``` text
Light
Medium
Heavy
```

Example:

``` text
Light → Web/API
Medium → Data processing
Heavy → ML/compute
```

## Step 2 --- Understand the environment

Classify the namespace:

``` text
DEV
STAGING
PRODUCTION
```

## Step 3 --- Analyze usage

Look at actual resource consumption:

``` text
CPU usage
Memory usage
Pod count
Peak usage
Average usage
Scaling behavior
```

## Step 4 --- Set initial quotas

Create reasonable starting values:

``` text
Dev        → Low
Staging    → Moderate
Production → Higher
```

## Step 5 --- Monitor

Observe whether applications:

-   Frequently hit limits
-   Consistently underuse resources
-   Need exceptions
-   Experience OOMKills
-   Fail to scale

## Step 6 --- Refine

Adjust quotas based on real data.

``` text
Set
 ↓
Monitor
 ↓
Measure
 ↓
Analyze
 ↓
Adjust
 ↓
Repeat
```

This is better than setting quotas once and never changing them.

------------------------------------------------------------------------

# 12. Practical Decision Matrix

  Factor                 Lower Requirement    Higher Requirement
  ---------------------- -------------------- -----------------------
  CPU intensity          Lower CPU quota      Higher CPU quota
  Memory intensity       Lower memory quota   Higher memory quota
  Number of services     Smaller pod limit    Larger pod limit
  Autoscaling            Limited scaling      More scaling headroom
  Environment            Dev                  Production
  Workload criticality   Lower allocation     Higher allocation
  Historical usage       Conservative         More capacity

------------------------------------------------------------------------

# 13. Real-World Example

Suppose an organization has three namespaces.

## Dev

``` text
Purpose: Development
Workload: API + experiments

CPU quota: Low
Memory quota: Low
Pod limit: Moderate
```

## Staging

``` text
Purpose: Production testing

CPU quota: Moderate
Memory quota: Moderate
Pod limit: Moderate
```

## Production

``` text
Purpose: Customer-facing services

CPU quota: High
Memory quota: High
Pod limit: Higher
Limits: Strict
```

The exact numbers should be determined using **actual workload and usage
data**, rather than blindly copying examples.

------------------------------------------------------------------------

# 14. Important Takeaways

### 1. Don't use one quota for everything

Different workloads have different resource requirements.

### 2. ResourceQuota is namespace-level

It controls the **total resources available to a namespace**.

### 3. LimitRange is workload/container-level

It establishes **defaults and constraints for individual resources**.

### 4. Environment matters

``` text
Dev → Low
Staging → Moderate
Production → Higher + Strict
```

### 5. Autoscaling matters

Microservices with HPA may need higher pod limits than a single-service
application.

### 6. Too restrictive is also bad

Low limits can cause:

``` text
Deployment failures
Developer friction
Exception requests
```

### 7. Too generous is also bad

High limits can lead to:

``` text
Resource waste
Over-allocation
Poor governance
```

### 8. Use actual usage data

Resource policies should evolve based on:

``` text
Observed usage
+
Peak usage
+
Growth
+
Scaling behavior
```

------------------------------------------------------------------------

# 15. Interview-Friendly Answer

**Question:** How would you design ResourceQuota and LimitRange for a
Kubernetes cluster?

**Answer:**

> "I wouldn't use a single universal quota. First, I would classify
> workloads based on CPU, memory, scaling behavior, and number of
> services. Then I would create environment-specific tiers for
> development, staging, and production. ResourceQuota would control the
> total resource consumption at the namespace level, while LimitRange
> would provide per-container defaults and constraints. I would start
> with conservative values, monitor actual CPU, memory, and pod usage,
> and continuously tune the quotas based on historical and peak usage.
> The goal is to avoid both developer friction from overly restrictive
> limits and loss of governance from overly generous limits."

------------------------------------------------------------------------

# 16. Quick Revision Sheet

## ResourceQuota

``` text
ResourceQuota
      ↓
Namespace-level resource budget
      ↓
Controls total CPU / Memory / Pods
```

## LimitRange

``` text
LimitRange
      ↓
Container/Pod-level policy
      ↓
Sets default requests & limits
```

## Workloads

``` text
Web API       → Light
Data Pipeline → Medium
ML Training   → Heavy
```

## Environments

``` text
DEV       → Low quotas
STAGING   → Moderate quotas
PROD      → Higher quotas + strict limits
```

## Governance Loop

``` text
Set reasonable defaults
        ↓
Monitor usage
        ↓
Analyze actual demand
        ↓
Tune quotas
        ↓
Repeat
```

### One-line concept to remember

> **Right-size resources according to workload + environment + usage
> data, then continuously refine.**
