# 04 — CRD Versioning & Schema Validation

> **Series**: CRD & Operators on AKS | 

---

## Multiple Versions in a CRD

A CRD can serve multiple API versions simultaneously. Only **one version** can be the storage version (what gets written to etcd). All others are converted on-the-fly.

```yaml
spec:
  versions:
    - name: v1alpha1
      served: true
      storage: false              # Old version, still served but not stored
      deprecated: true
      deprecationWarning: "v1alpha1 is deprecated, migrate to v1"
      schema:
        openAPIV3Schema: { ... }

    - name: v1
      served: true
      storage: true               # Current storage version
      schema:
        openAPIV3Schema: { ... }

    - name: v2
      served: true
      storage: false              # Future version, preview
      schema:
        openAPIV3Schema: { ... }
```

### Version Lifecycle Rules

```
v1alpha1 (unstable)  →  v1beta1 (feature complete)  →  v1 (stable/GA)
   served: true          served: true                    served: true
   storage: false        storage: false                  storage: true
   deprecated: true
```

**Rules you must follow:**
- Only ONE version can have `storage: true`
- A version with `served: false` is effectively hidden from the API (but data still exists in etcd if it was once the storage version)
- Never remove a version that still has live objects stored — migrate first

---

## Conversion Webhooks

When multiple versions have **different field structures**, the API server cannot convert them automatically. You need a **conversion webhook**.

```yaml
spec:
  conversion:
    strategy: Webhook             # None (default, only for identical schemas) or Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          name: database-operator-webhook
          namespace: operators
          path: /convert
          port: 443
        caBundle: <base64-encoded-CA>
```

### Conversion Webhook Flow

```
User requests v1alpha1 object
        │
        ▼
API Server: "I have this stored as v1 in etcd"
        │
        ▼
API Server calls conversion webhook:
  POST /convert
  Body: ConversionReview { desired: "v1alpha1", objects: [<v1 object>] }
        │
        ▼
Webhook converts v1 → v1alpha1 and returns ConversionReview response
        │
        ▼
User receives v1alpha1 response
```

### Hub-Spoke Conversion Model (Best Practice)

Always designate one version (usually `v1`) as the **hub**. All conversions go through it:

```
v1alpha1 ──► v1 (hub) ──► v2
v1alpha1 ◄── v1 (hub) ◄── v2
```

This means you only need N-1 conversion paths instead of N*(N-1)/2.

```go
// Conversion webhook handler
func (h *ConversionHandler) Handle(ctx context.Context, req webhook.ConversionRequest) webhook.ConversionResponse {
    convertedObjects := make([]runtime.RawExtension, 0, len(req.Objects))

    for _, obj := range req.Objects {
        cr := &unstructured.Unstructured{}
        if err := cr.UnmarshalJSON(obj.Raw); err != nil {
            return webhook.ConversionResponse{Result: metav1.Status{Status: "Failure"}}
        }

        switch req.DesiredAPIVersion {
        case "mycompany.io/v1":
            converted, err := convertToV1(cr)
            if err != nil {
                return errorResponse(err)
            }
            convertedObjects = append(convertedObjects, runtime.RawExtension{Object: converted})

        case "mycompany.io/v2":
            converted, err := convertToV2(cr)
            if err != nil {
                return errorResponse(err)
            }
            convertedObjects = append(convertedObjects, runtime.RawExtension{Object: converted})
        }
    }

    return webhook.ConversionResponse{
        Result:           metav1.Status{Status: "Success"},
        ConvertedObjects: convertedObjects,
    }
}
```

**Interview Tip**: _"How do you do zero-downtime CRD schema migrations?"_
Answer: Deploy conversion webhook **before** updating the CRD. The webhook must be live and healthy before the new version is added to the CRD spec. A broken webhook blocks all access to that resource type.

---

## Validation Rules — CEL (Common Expression Language)

Since Kubernetes 1.25, **CEL rules** (`x-kubernetes-validations`) allow complex cross-field validation directly in the CRD schema — without a separate admission webhook.

```yaml
schema:
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        properties:
          minReplicas:
            type: integer
          maxReplicas:
            type: integer
          engine:
            type: string
        x-kubernetes-validations:
          # Cross-field validation
          - rule: "self.maxReplicas >= self.minReplicas"
            message: "maxReplicas must be >= minReplicas"

          # Upper bound validation
          - rule: "self.maxReplicas <= 100"
            message: "maxReplicas cannot exceed 100"

          # Immutability — engine cannot change after creation
          - rule: "!has(oldSelf.engine) || self.engine == oldSelf.engine"
            message: "engine is immutable once set"

          # Conditional requirement
          - rule: "self.minReplicas > 1 ? has(self.storageClass) : true"
            message: "storageClass is required when minReplicas > 1"
```

### CEL Rule Reference

| CEL Expression | Meaning |
|---|---|
| `self.field` | Access current object field |
| `oldSelf.field` | Access the old value (for UPDATE validation) |
| `has(self.field)` | Check if optional field is present |
| `self.field == "value"` | Equality check |
| `self.list.all(x, x > 0)` | All items in list satisfy condition |
| `self.list.exists(x, x == "foo")` | At least one item satisfies condition |
| `size(self.list) <= 10` | List size constraint |
| `self.field.matches("^v[0-9]+$")` | Regex match |

### CEL vs Admission Webhook — When to Use Which

| Use CEL rules when | Use Admission Webhook when |
|---|---|
| Cross-field validation within one resource | Validating against external systems (DB, APIs) |
| Immutability constraints | Cross-resource validation |
| Simple computed checks | Complex business logic |
| No external calls needed | Need to mutate/inject fields |

---

## Defaulting with CRDs

Field defaults are applied server-side when a field is omitted in the submitted object.

```yaml
properties:
  spec:
    type: object
    properties:
      replicas:
        type: integer
        default: 1                        # Applied when field is omitted

      storageClass:
        type: string
        default: "managed-premium"        # AKS default: Azure Premium SSD

      resources:
        type: object
        default:                          # Object-level default
          requests:
            cpu: "100m"
            memory: "128Mi"

      backup:
        type: object
        default:
          enabled: false
          schedule: "0 2 * * *"
```

**Important**: Defaults are applied by the API server on admission — they are stored in etcd. The controller always sees the defaulted values.

---

## CRD Schema Migration Checklist

```
Adding a new field (safe):
  ✅ Make it optional (not in required[])
  ✅ Provide a default value
  ✅ No conversion webhook needed

Renaming a field (breaking):
  ✅ Add new field (optional with default)
  ✅ Deploy conversion webhook (old → new field mapping)
  ✅ Update CRD with new version
  ✅ Migrate existing objects (batch job)
  ✅ Deprecate old field (keep it, mark deprecated)
  ❌ Never: remove old field while objects still use it

Tightening validation (breaking):
  ✅ New validation only on CREATE (not UPDATE) if possible
  ✅ Use CEL oldSelf checks to grandfather existing values
  ❌ Never: apply stricter validation to fields already stored in etcd

Changing storage version:
  ✅ New version served: true, storage: false first
  ✅ Deploy conversion webhook
  ✅ Set new version storage: true
  ✅ Run storage migration: kubectl get <resource> -A | kubectl apply
  ✅ Old version can be set served: false after migration
```

---

## Navigation

| # | Topic |
|---|---|
| [01](./01-kubernetes-extension-model.md) | Kubernetes Extension Model |
| [02](./02-crd-deep-dive.md) | CRD Deep Dive |
| [03](./03-custom-resources.md) | Custom Resources |
| **04** | Versioning & Validation ← you are here |
| [05](./05-controllers.md) | Controllers |
| [06](./06-operator-pattern.md) | Operator Pattern |
| [07](./07-operator-frameworks.md) | Operator Frameworks |
| [08](./08-operator-lifecycle-manager.md) | OLM |
| [09](./09-advanced-operator-patterns.md) | Advanced Patterns |
| [10](./10-aks-crd-operator-focus.md) | AKS Focus |
| [11](./11-hands-on-examples.md) | Hands-On Examples |
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
