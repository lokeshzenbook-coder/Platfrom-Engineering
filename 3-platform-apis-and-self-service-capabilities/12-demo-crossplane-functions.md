> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Crossplane Functions

> Demo showing how to use Crossplane Patch-and-Transform functions to map composite resource fields into provider configuration using Composition transforms like map, combine, convert, and status writeback

This demo shows how to use Crossplane's Patch-and-Transform function to translate high-level platform inputs (for example, `tier: large`) into concrete provider values (for example, `db.r5.large`). We'll walk through each of the four transform types supported by the Patch-and-Transform function and add them to a Composition so you can observe their effects.

We use a simple WebApp composite resource with the following spec fields:

* `tier` (free | standard | premium)
* `environment` (string)
* `replicas` (integer)

The platform (Composition) decides the actual resource configuration that will be created from those inputs — in this example the Composition constructs a Kubernetes ConfigMap.

<Callout icon="lightbulb" color="#1CB2FE">
  When you modify an existing Composition, append new patches/transforms to the existing `patches` array — do not replace the existing entries. Appending preserves prior behavior while adding new transformations.
</Callout>

***

## 1) CompositeResourceDefinition (XRD)

Create an XRD for the WebApp composite resource. Note the `spec` fields and the `status.configName` property — the latter is required if you intend to write back values into the XR status using `ToCompositeFieldPath`.

```yaml theme={null}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: webapps.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: WebApp
    plural: webapps
  scope: Namespaced
  versions:
    - name: v1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                tier:
                  type: string
                  enum: ["free", "standard", "premium"]
                environment:
                  type: string
                replicas:
                  type: integer
              required: ["tier", "environment", "replicas"]
            status:
              type: object
              properties:
                configName:
                  type: string
          x-kubernetes-preserve-unknown-fields: true
```

Apply and verify the XRD:

```bash theme={null}
kubectl apply -f webapp-xrd.yaml
kubectl get compositeresourcedefinition webapps.platform.example.com -o yaml
```

***

## 2) Base Composition (ConfigMap resource)

This base Composition creates a ConfigMap named `webapp-config` in the `function-lab` namespace. It uses the Patch-and-Transform function (`pt.fn.crossplane.io/v1beta1`, function name `function-patch-and-transform`) with `Resources` input. Start by mapping `spec.environment` from the XR into the ConfigMap `data.environment`.

```yaml theme={null}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: webapp-kubernetes
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1
    kind: WebApp
  mode: Pipeline
  pipeline:
    steps:
      - name: patch-and-transform
        functionRef:
          name: function-patch-and-transform
          apiVersion: pt.fn.crossplane.io/v1beta1
        input:
          apiVersion: pt.fn.crossplane.io/v1beta1
          kind: Resources
          resources:
            - name: webapp-config
              base:
                apiVersion: kubernetes.m.crossplane.io/v1alpha1
                kind: Object
                metadata:
                  name: webapp-config
                  namespace: function-lab
                spec:
                  providerConfigRef:
                    name: ClusterProviderConfig
                  forProvider:
                    manifest:
                      apiVersion: v1
                      kind: ConfigMap
                      metadata:
                        name: webapp-config
                        namespace: function-lab
                      data: {}
              patches:
                - type: FromCompositeFieldPath
                  fromFieldPath: spec.environment
                  toFieldPath: spec.forProvider.manifest.data.environment
```

Apply the Composition:

```bash theme={null}
kubectl apply -f webapp-composition.yaml
kubectl get composition webapp-kubernetes -o yaml
```

***

## 3) Transform type: map (value mapping)

Use the `map` transform to translate user-friendly values into concrete provider values. For example, map `spec.tier` to `data.cpuLimit` in the ConfigMap.

Append this patch to `resources[].patches` in your Composition:

```yaml theme={null}
- type: FromCompositeFieldPath
  fromFieldPath: spec.tier
  toFieldPath: spec.forProvider.manifest.data.cpuLimit
  transforms:
    - type: map
      map:
        free: "100m"
        standard: "500m"
        premium: "2000m"
```

Apply the updated Composition:

```bash theme={null}
kubectl apply -f webapp-composition.yaml
```

Create an example WebApp XR to test mapping:

```yaml theme={null}
# xr-example.yaml
apiVersion: platform.example.com/v1
kind: WebApp
metadata:
  name: my-app
  namespace: default
spec:
  environment: prod
  tier: premium
  replicas: 3
```

```bash theme={null}
kubectl apply -f xr-example.yaml
# wait for composition to reconcile, then:
kubectl get configmap -n function-lab webapp-config -o yaml
```

Expected ConfigMap (partial):

```yaml theme={null}
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: function-lab
data:
  environment: "prod"
  cpuLimit: "2000m"
```

***

## 4) Transform type: CombineFromComposite (combine fields into one)

`CombineFromComposite` merges multiple XR fields into a single value. For example, concatenate `metadata.name` and `spec.environment` into `data.appId` formatted as `<name>-<environment>`.

Append this patch to `resources[].patches`:

```yaml theme={null}
- type: CombineFromComposite
  combine:
    variables:
      - fromFieldPath: metadata.name
      - fromFieldPath: spec.environment
    strategy: string
    fmt: "%s-%s"
  toFieldPath: spec.forProvider.manifest.data.appId
```

Apply the Composition, then (optionally) recreate the XR to force a new reconciliation:

```bash theme={null}
kubectl apply -f webapp-composition.yaml
kubectl delete -f xr-example.yaml || true
kubectl apply -f xr-example.yaml
kubectl get configmap -n function-lab webapp-config -o yaml
```

Expected ConfigMap (partial):

```yaml theme={null}
data:
  appId: "my-app-prod"
  cpuLimit: "2000m"
  environment: "prod"
```

***

## 5) Transform type: ToCompositeFieldPath (write back to XR status)

`ToCompositeFieldPath` writes a value from the composed resource back into the XR status. This is helpful to expose generated resource names or IDs to platform users.

Append this patch to `resources[].patches` to copy the composed resource's `metadata.name` into `status.configName` of the XR:

```yaml theme={null}
- type: ToCompositeFieldPath
  fromFieldPath: metadata.name
  toFieldPath: status.configName
```

<Callout icon="warning" color="#FF6B6B">
  The XRD must declare `status.configName` (or whatever status field you write to). If the XRD does not include the named status property, Crossplane will reject the status write. Reapply the XRD after updating it.
</Callout>

Apply the XRD and Composition, then recreate the XR:

```bash theme={null}
kubectl apply -f webapp-xrd.yaml
kubectl apply -f webapp-composition.yaml
kubectl delete -f xr-example.yaml || true
kubectl apply -f xr-example.yaml
kubectl get webapp my-app -o yaml
```

You should see `status.configName` populated with the composed resource name (for example `my-app-97c0b06087b`).

***

## 6) Transform type: convert (type conversion)

The `convert` transform converts values between types (for example, integer → string). This is required for ConfigMap `data` entries which must be strings.

Append this patch to `resources[].patches`:

```yaml theme={null}
- type: FromCompositeFieldPath
  fromFieldPath: spec.replicas
  toFieldPath: spec.forProvider.manifest.data.replicas
  transforms:
    - type: convert
      convert:
        toType: string
```

Apply the updated Composition and recreate the XR to observe the converted value in the ConfigMap:

```bash theme={null}
kubectl apply -f webapp-composition.yaml
kubectl delete -f xr-example.yaml || true
kubectl apply -f xr-example.yaml
kubectl get configmap -n function-lab webapp-config -o yaml
```

Expected ConfigMap (partial):

```yaml theme={null}
data:
  appId: "my-app-prod"
  cpuLimit: "2000m"
  environment: "prod"
  replicas: "3"    # replicas is now a string
```

***

## 7) Summary & quick reference

The four transforms covered here let you implement flexible, platform-driven mappings and derive provider configuration from high-level inputs.

| Transform type         | Purpose                                                   | Example                                                    |
| ---------------------- | --------------------------------------------------------- | ---------------------------------------------------------- |
| `map`                  | Map a set of input values to specific output values       | `tier` → `cpuLimit` (`free` → `100m`, `premium` → `2000m`) |
| `CombineFromComposite` | Combine multiple XR fields into one formatted value       | `metadata.name + spec.environment` → `appId` (`%s-%s`)     |
| `ToCompositeFieldPath` | Write a composed resource value back into the XR `status` | composed `metadata.name` → `status.configName`             |
| `convert`              | Convert between types (e.g., integer → string)            | `spec.replicas` (int) → `data.replicas` (string)           |

These transforms cover common composition needs: mapping platform-friendly inputs to provider values, aggregating identifiers, surfacing generated names back to users, and converting types for compatibility (e.g., ConfigMap data).

Next steps: experiment with these transforms in your own Composition to implement policy-driven defaults, computed values, and richer platform APIs.

## Links and references

* Crossplane docs: [https://crossplane.io/docs/](https://crossplane.io/docs/)
* Crossplane Functions: [https://doc.crds.example.org/](https://doc.crds.example.org/) (replace with your function docs or internal reference)
* Kubernetes ConfigMap: [https://kubernetes.io/docs/concepts/configuration/configmap/](https://kubernetes.io/docs/concepts/configuration/configmap/)
* Kubernetes API Extensions (CRDs/XRD): [https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/756ffaae-767b-4743-9724-c05d3fbf9a18/lesson/248dcf28-f81b-417f-ab7a-c9fd53237e28" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/756ffaae-767b-4743-9724-c05d3fbf9a18/lesson/5c8eaf8d-b487-45cc-be5d-bb228eea6645" />
</CardGroup>
