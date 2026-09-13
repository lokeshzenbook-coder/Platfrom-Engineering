> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Demo Building a Custom Resource Definition

> Guide to creating a namespaced Kubernetes CustomResourceDefinition for a Database type with openAPIV3Schema validation, enums, required fields, example YAML, and troubleshooting

Kubernetes includes built-in types like Deployments, Services, and ConfigMaps, but platform teams often need higher-level, domain-specific abstractions—such as `Database` or `TenantEnvironment`—to simplify user workflows. CustomResourceDefinitions (CRDs) let you extend the Kubernetes API with these custom types and enforce server-side validation using `openAPIV3Schema`.

This guide walks through creating a namespaced `Database` CRD that:

* Serves a single API version: `v1`.
* Requires `spec.engine` and `spec.size`.
* Validates `engine` and `size` using enums.
* Allows optional `version` and `storage` fields.

<Callout icon="lightbulb" color="#1CB2FE">
  Only one version entry in `spec.versions` may have `storage: true`. This version is used as the persisted storage version in etcd.
</Callout>

## CRD structure — key fields and meaning

Below are the important CRD fields you will set. Use the links for more details:

* Kubernetes CRD docs: [https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

| Field                    | Purpose                                                                     | Notes / Example                                                              |
| ------------------------ | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `metadata.name`          | The fully-qualified name of the CRD                                         | Must be `<plural>.<group>` — e.g. `databases.platform.example.com`           |
| `spec.group`             | API domain (group) for your custom types                                    | e.g. `platform.example.com`                                                  |
| `spec.scope`             | Resource scope                                                              | `Namespaced` or `Cluster`                                                    |
| `spec.names`             | How the resource is referenced (plural, singular, kind, shortNames)         | e.g. `plural: databases`, `kind: Database`                                   |
| `spec.versions`          | List of served versions and optional validation schema                      | Each version may include `schema.openAPIV3Schema` for server-side validation |
| `schema.openAPIV3Schema` | OpenAPI v3 schema used by the API server to validate custom resources (CRs) | Define `type`, `properties`, `required`, `enum`, etc.                        |

Notes:

* The string ` <plural>.<group>` is used as an example for CRD names; when writing docs, wrap such placeholders in backticks: `<plural>.<group>`.
* Use `openAPIV3Schema` inside `spec.versions[*]` to perform server-side validation for fields, types, required keys, and enums.

## Example CRD YAML: Database

Below is a complete CRD definition for a namespaced `Database` custom resource. Save this as `database-crd.yaml` and apply it to your cluster.

```yaml theme={null}
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # Must be the fully qualified name: <plural>.<group>
  name: databases.platform.example.com
spec:
  group: platform.example.com
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
      - db
      - dbs
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - engine
                - size
              properties:
                engine:
                  type: string
                  enum:
                    - postgresql
                    - mysql
                    - mariadb
                size:
                  type: string
                  enum:
                    - small
                    - medium
                    - large
                version:
                  type: string
                storage:
                  type: string
```

Schema highlights:

* The `spec` object is validated using `openAPIV3Schema`.
* `engine` and `size` are required and constrained to specific enum values.
* `version` and `storage` are optional string fields.

## Apply the CRD

Create the `database-crd.yaml` file (shown above), then apply it:

```bash theme={null}
kubectl apply -f database-crd.yaml
```

Expected output:

```bash theme={null}
customresourcedefinition.apiextensions.k8s.io/databases.platform.example.com created
```

Verify the API resource is available:

```bash theme={null}
kubectl api-resources | grep platform
```

Example output:

```bash theme={null}
databases  db,dbs    platform.example.com/v1   true   Database
```

Once the CRD is present, you can create `Database` custom resources. The API server will validate them according to the schema.

## Creating Custom Resources (CRs) — examples and validation behaviour

The examples below show common validation errors and a final valid CR. Save each YAML to the noted filename and apply with `kubectl apply -f <file>`.

1. Missing `metadata.name` (invalid)

`cr-invalid-name.yaml`:

```yaml theme={null}
apiVersion: platform.example.com/v1
kind: Database
metadata:
  # name intentionally omitted to demonstrate the error
  namespace: platform-apis
spec:
  engine: mongo
  size: medium
```

Apply:

```bash theme={null}
kubectl apply -f cr-invalid-name.yaml
```

Error:

```bash theme={null}
error: error when retrieving current configuration of:
Resource: "platform.example.com/v1, Resource=databases", GroupVersionKind: "platform.example.com/v1, Kind=Database"
Name: "", Namespace: "platform-apis"
from server for: "cr-invalid-name.yaml": resource name may not be empty
```

Explanation: `metadata.name` is required for any Kubernetes resource.

2. Unsupported `engine` (invalid)

`cr-invalid-engine.yaml`:

```yaml theme={null}
apiVersion: platform.example.com/v1
kind: Database
metadata:
  name: db-order
  namespace: platform-apis
spec:
  engine: mongo
  size: medium
```

Apply:

```bash theme={null}
kubectl apply -f cr-invalid-engine.yaml
```

Error:

```bash theme={null}
The Database "db-order" is invalid: spec.engine: Unsupported value: "mongo": supported values: "postgresql", "mysql", "mariadb"
```

Explanation: `spec.engine` must match one of the allowed enum values from the CRD (`postgresql`, `mysql`, `mariadb`).

3. Missing required `size` (invalid)

`cr-missing-size.yaml`:

```yaml theme={null}
apiVersion: platform.example.com/v1
kind: Database
metadata:
  name: db-order
  namespace: platform-apis
spec:
  engine: mariadb
  # size omitted intentionally
```

Apply:

```bash theme={null}
kubectl apply -f cr-missing-size.yaml
```

Error:

```bash theme={null}
The Database "db-order" is invalid: spec.size: Required value
```

Explanation: `spec.size` is required by the CRD schema.

4. Valid CR (successful)

`cr-valid.yaml`:

```yaml theme={null}
apiVersion: platform.example.com/v1
kind: Database
metadata:
  name: db-order
  namespace: platform-apis
spec:
  engine: mariadb
  size: medium
  version: "15"
  storage: 50Gi
```

Apply:

```bash theme={null}
kubectl apply -f cr-valid.yaml
```

Expected output:

```bash theme={null}
database.platform.example.com/db-order created
```

Verify the resource exists (using short name):

```bash theme={null}
kubectl get db -n platform-apis
```

Example output:

```bash theme={null}
NAME       AGE
db-order   15s
```

Inspect the resource:

```bash theme={null}
kubectl describe database db-order -n platform-apis
```

Example (trimmed) output:

```bash theme={null}
Name:         db-order
Namespace:    platform-apis
API Version:  platform.example.com/v1
Kind:         Database
Spec:
  Engine:    mariadb
  Size:      medium
  Version:   15
  Storage:   50Gi
```

This confirms that:

* The API server enforces schema validation.
* The custom resource behaves like native Kubernetes resources with `kubectl get`, `kubectl describe`, etc.

## Troubleshooting tips

<Callout icon="lightbulb" color="#1CB2FE">
  If you don't see your CRD listed after applying, check the API server logs and `kubectl get crd` output. Also ensure `metadata.name` in the CRD follows the required `<plural>.<group>` naming convention and that `spec.versions[*].storage` is set on only one version.
</Callout>

Common checks:

* `kubectl get crd databases.platform.example.com -o yaml` — inspect what was created.
* `kubectl api-resources | grep platform` — confirm the API resource is registered.
* Ensure your cluster version supports `apiextensions.k8s.io/v1` (Kubernetes 1.16+).

## Summary and best practices

* CRDs extend the Kubernetes API to introduce domain-specific object types for your platform.
* Use `openAPIV3Schema` inside `spec.versions` to enable server-side validation (required fields, enums, types).
* Ensure `metadata.name` of the CRD follows the `<plural>.<group>` pattern.
* Only one `spec.versions[*]` entry may have `storage: true`; it determines the persisted version stored in etcd.
* After creating a CRD, `kubectl` and the API server will validate and accept/reject CRs according to the schema you define.

Further reading:

* [Custom Resource Definitions (Kubernetes docs)](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
* [apiextensions.k8s.io API reference](https://kubernetes.io/docs/reference/using-api/api-concepts/#custom-resources)

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/756ffaae-767b-4743-9724-c05d3fbf9a18/lesson/c82aa899-a70b-43ff-b920-da6222c154c1" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/756ffaae-767b-4743-9724-c05d3fbf9a18/lesson/57aa85b4-e21b-4311-8593-6489e31df10a" />
</CardGroup>
