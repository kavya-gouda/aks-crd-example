# 02 — Custom Resource Definitions (CRD): Deep Dive

> **Series**: CRD & Operators on AKS | Senior Engineer (12 YOE) Interview Prep

---

## What is a CRD?

A **Custom Resource Definition (CRD)** is itself a Kubernetes resource (`kind: CustomResourceDefinition`) that tells the Kubernetes API server about a **new resource type** you want to introduce into the cluster.

Think of it this way:
- `Deployment` is a built-in resource type → its schema is baked into `kube-apiserver`
- `Database` (your custom resource) → its schema is registered dynamically via a CRD

---

## CRD Anatomy — Full YAML Breakdown

```yaml
apiVersion: apiextensions.k8s.io/v1          # API group for CRDs
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.io               # MUST be <plural>.<group>
spec:
  group: mycompany.io                        # API Group (like apps, networking.k8s.io)
  scope: Namespaced                          # Namespaced OR Cluster
  names:
    plural: databases                        # URL path: /apis/mycompany.io/v1/databases
    singular: database                       # Used in kubectl
    kind: Database                           # PascalCase type name
    shortNames:                              # kubectl shortcuts
      - db
    categories:                              # Groups for `kubectl get all`
      - mycompany
  versions:
    - name: v1
      served: true                           # API version actively served
      storage: true                          # Version stored in etcd (only one can be true)
      deprecated: false                      # Mark old versions deprecated
      deprecationWarning: "Use v2 instead"
      schema:
        openAPIV3Schema:                     # Validation schema (structural schema)
          type: object
          properties:
            spec:
              type: object
              required: ["engine", "replicas"]
              properties:
                engine:
                  type: string
                  enum: ["postgres", "mysql", "mongodb"]
                  description: "Database engine type"
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
                version:
                  type: string
                  pattern: '^[0-9]+\.[0-9]+$'
                storageSize:
                  type: string
                  default: "10Gi"
                resources:
                  type: object
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Running", "Failed", "Terminating"]
                readyReplicas:
                  type: integer
                conditions:
                  type: array
                  items:
                    type: object
                    required: ["type", "status"]
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                        enum: ["True", "False", "Unknown"]
                      reason:
                        type: string
                      message:
                        type: string
                      lastTransitionTime:
                        type: string
                        format: date-time
      # Sub-resources
      subresources:
        status: {}                           # Enables /status sub-resource (separate RBAC)
        scale:                               # Enables /scale sub-resource (HPA support)
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
          labelSelectorPath: .status.labelSelector
      # Printer columns (kubectl get output)
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
```

---

## Key CRD Concepts to Master

### Scope: Namespaced vs Cluster-scoped

```yaml
# Namespaced — lives within a namespace
scope: Namespaced
# Access URL: /apis/mycompany.io/v1/namespaces/{namespace}/databases

# Cluster-scoped — cluster-wide resource (like Nodes, PersistentVolumes)
scope: Cluster
# Access URL: /apis/mycompany.io/v1/databases
```

**When to use Cluster-scoped**: Global configuration, cross-namespace resources (ClusterIssuer in cert-manager), infrastructure-level resources.

---

### Structural Schema (Required since k8s 1.15+)

Every CRD in `apiextensions.k8s.io/v1` MUST have a **structural schema**. Rules:
1. Must define `type` at every level
2. Cannot use `x-kubernetes-preserve-unknown-fields: true` at top level (except for extensibility)
3. Enables **pruning** (unknown fields are dropped)
4. Enables **defaulting**
5. Enables **nullable** fields

```yaml
# Preserve unknown fields in a specific nested object (escape hatch)
spec:
  type: object
  x-kubernetes-preserve-unknown-fields: true  # Only at nested level
```

---

### Sub-resources

**Status sub-resource** (`.status`):
- Separates spec writes from status writes
- Different RBAC: controllers need `update` on `databases/status`, users update `databases`
- Prevents update loops (spec write != status write)

```bash
# Update status via sub-resource (used by controllers)
kubectl patch database mydb --subresource=status --type=merge \
  -p '{"status":{"phase":"Running"}}'
```

**Scale sub-resource**:
- Allows HPA (Horizontal Pod Autoscaler) to scale your custom resource
- Required if you want HPA to work with your operator

---

### CRD Categories and Short Names

```bash
# With categories: [mycompany], this returns your CRs too:
kubectl get mycompany

# With shortNames: [db]
kubectl get db                  # same as: kubectl get databases
kubectl get db -n production
```

---

### CRD Printer Columns in Detail

```yaml
additionalPrinterColumns:
  - name: Engine
    type: string                # string, integer, number, boolean, date
    jsonPath: .spec.engine
    priority: 0                 # 0 = always shown, 1+ = shown with -o wide
  - name: Age
    type: date
    jsonPath: .metadata.creationTimestamp
    format: date-time           # date, date-time, byte, int64, float64
```

```bash
kubectl get databases                    # Shows default columns
kubectl get databases -o wide            # Shows priority > 0 columns too
```

---

## Navigation

| # | Topic |
|---|---|
| [01](./01-kubernetes-extension-model.md) | Kubernetes Extension Model |
| **02** | CRD Deep Dive ← you are here |
| [03](./03-custom-resources.md) | Custom Resources |
| [04](./04-crd-versioning-schema-validation.md) | Versioning & Validation |
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
