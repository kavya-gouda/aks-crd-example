# 02 — Custom Resource Definitions (CRD): Deep Dive

> **Series**: CRD & Operators on AKS 

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

#### What Problem Does It Solve?

Before structural schema (the old `v1beta1` CRDs), you could write a CRD with **no schema at all**. The API server accepted any arbitrary YAML — no validation, no type checking, no defaults. This caused silent bugs where typos in field names were accepted and simply ignored.

```yaml
# OLD way — v1beta1, no schema (BAD, removed in k8s 1.22)
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
spec:
  validation:
    openAPIV3Schema:
      type: object    # That's it — no field definitions at all
```

**Structural schema** forces you to describe every field with a `type`. Once you do that, Kubernetes unlocks powerful features automatically.

---

#### The Core Rule: Every Level Must Have a `type`

Think of it like TypeScript vs JavaScript. In plain JavaScript you can pass any object anywhere. In TypeScript (structural schema), every field must have a declared type.

```yaml
# NON-structural (broken) — missing type on nested object
properties:
  spec:
    properties:          # ❌ ERROR: no "type: object" here
      engine:
        type: string

# STRUCTURAL (correct) — type declared at every level
properties:
  spec:
    type: object         # ✅ type declared
    properties:
      engine:
        type: string     # ✅ type declared
      replicas:
        type: integer    # ✅ type declared
      config:
        type: object     # ✅ type declared
        properties:
          timeout:
            type: string # ✅ type declared
```

**Rule**: Every `properties`, `items` (for arrays), and `additionalProperties` must have a `type` at its own level.

---

#### Feature 1: Pruning (Unknown Fields Are Dropped)

Once you have a structural schema, the API server automatically **removes any field you didn't define** in the schema. This is called **pruning**.

```yaml
# Your CRD schema defines only these fields:
spec:
  type: object
  properties:
    engine:
      type: string
    replicas:
      type: integer
```

```yaml
# User submits this CR with an extra typo'd field:
spec:
  engine: postgres
  replicas: 3
  engnie: postgres     # ← typo — extra unknown field
  myCustomField: "foo" # ← not in schema
```

```yaml
# What actually gets stored in etcd (after pruning):
spec:
  engine: postgres
  replicas: 3
  # engnie and myCustomField are silently DROPPED ✅
```

**Why pruning matters:**
- Prevents garbage accumulating in etcd
- Catches typos immediately (field is dropped → controller doesn't see it → behavior unexpected → you notice and fix it)
- Before pruning, `engnie: postgres` would sit silently in etcd forever and your controller would read `engine: ""` — very hard to debug

---

#### Feature 2: Defaulting (Auto-fill Missing Fields)

Structural schema enables server-side defaults. When a user doesn't provide a field, the API server fills it in automatically **before storing in etcd**.

```yaml
spec:
  type: object
  properties:
    replicas:
      type: integer
      default: 1            # If user omits replicas, it becomes 1

    storageClass:
      type: string
      default: "managed-premium"   # AKS default

    backup:
      type: object
      default:              # Entire object default
        enabled: false
        schedule: "0 2 * * *"
      properties:
        enabled:
          type: boolean
        schedule:
          type: string
```

```yaml
# User submits (minimal):
spec:
  engine: postgres

# What gets stored in etcd (after defaulting):
spec:
  engine: postgres
  replicas: 1                      # ← filled in by default
  storageClass: "managed-premium"  # ← filled in by default
  backup:
    enabled: false                 # ← filled in by default
    schedule: "0 2 * * *"          # ← filled in by default
```

**Why defaulting in schema is better than defaulting in controller:**
- Controller always sees the default values — no nil checks needed
- `kubectl get database mydb -o yaml` shows the actual stored values — no surprises
- Default is applied once at admission, not re-applied every reconcile

---

#### Feature 3: Nullable Fields

Sometimes a field is an object but you want to allow users to explicitly set it to `null` to "unset" it:

```yaml
properties:
  backup:
    type: object
    nullable: true         # Allows: backup: null  (to disable)
    properties:
      enabled:
        type: boolean
```

```yaml
# Without nullable: true, this would be rejected:
spec:
  backup: null    # ❌ error without nullable

# With nullable: true:
spec:
  backup: null    # ✅ accepted — means "no backup config"
```

---

#### Feature 4: x-kubernetes-preserve-unknown-fields (Escape Hatch)

Sometimes you genuinely need to accept arbitrary key-value data — for example, a field that holds user-defined labels, arbitrary config, or raw Helm values. Use `x-kubernetes-preserve-unknown-fields: true` **only on that specific nested field**:

```yaml
spec:
  type: object
  properties:
    engine:
      type: string             # ← still validated normally
    replicas:
      type: integer            # ← still validated normally
    extraConfig:
      type: object
      x-kubernetes-preserve-unknown-fields: true  # ← this field accepts anything
```

```yaml
# Now this is accepted:
spec:
  engine: postgres
  replicas: 3
  extraConfig:                 # Anything goes inside here
    custom_setting_1: "foo"
    any_key: 42
    nested:
      deep: value
```

**Rules for using this escape hatch:**
- NEVER use it at the top level of `spec` — defeats the purpose of structural schema
- Use it only for fields that genuinely need dynamic keys (labels map, annotations map, arbitrary config)
- In AKS environments: Azure Policy / Gatekeeper may flag CRDs that use this broadly

---

#### Full Example: Structural vs Non-Structural Side-by-Side

```yaml
# ❌ NON-STRUCTURAL — will be REJECTED by apiextensions.k8s.io/v1
schema:
  openAPIV3Schema:
    properties:           # Missing: type: object
      spec:
        properties:       # Missing: type: object
          engine:
            type: string
          config:
            properties:   # Missing: type: object
              timeout:
                type: string

# ✅ STRUCTURAL — accepted, all features unlocked
schema:
  openAPIV3Schema:
    type: object          # ← required
    properties:
      spec:
        type: object      # ← required
        properties:
          engine:
            type: string
          config:
            type: object  # ← required
            properties:
              timeout:
                type: string
```

---

#### Summary Table

| Feature | Requires Structural Schema? | What It Does |
|---|---|---|
| **Pruning** | ✅ Yes | Drops unknown fields on admission |
| **Defaulting** | ✅ Yes | Fills in missing fields with defaults |
| **Nullable** | ✅ Yes | Allows `null` value for object fields |
| **CEL validation** | ✅ Yes | `x-kubernetes-validations` rules |
| **Server-Side Apply** | ✅ Yes | Field ownership tracking |
| **kubectl explain** | ✅ Yes (richer) | Shows field descriptions from schema |

**Interview answer for "Why is structural schema required?":**
> Without it, the API server can't safely prune, default, or validate CRs. Structural schema makes the CRD a first-class API citizen — same guarantees as built-in types like Deployments.

---

### Sub-resources

A **sub-resource** is an additional HTTP endpoint hanging off your main resource URL. Think of it as a separate "channel" for a specific type of operation on the same object.

```
Main resource:      /apis/mycompany.io/v1/namespaces/prod/databases/mydb
Status sub-resource: /apis/mycompany.io/v1/namespaces/prod/databases/mydb/status
Scale sub-resource:  /apis/mycompany.io/v1/namespaces/prod/databases/mydb/scale
```

CRDs support exactly two built-in sub-resources: **`/status`** and **`/scale`**.

---

#### Status Sub-resource — Deep Dive

##### What Problem Does It Solve?

Without the status sub-resource, `spec` and `status` are both part of the same object. A single `kubectl apply` or `r.Update()` writes everything — spec AND status — in one call.

This creates an **infinite reconcile loop**:

```
Without status sub-resource:
─────────────────────────────────────────────────────
User applies CR                 → triggers reconcile
Controller updates status       → r.Update(ctx, db)
                                  (writes ENTIRE object including spec)
API server sees object changed  → triggers reconcile AGAIN
Controller updates status       → r.Update(ctx, db)  again
API server sees object changed  → triggers reconcile AGAIN
... infinite loop ...
```

With `subresources: status: {}` enabled, **spec and status are written through separate endpoints**. A status write does NOT trigger a spec-change event — the reconcile loop only fires when spec actually changes:

```
With status sub-resource:
─────────────────────────────────────────────────────
User applies CR (spec change)       → triggers reconcile ✅
Controller writes status            → r.Status().Update(ctx, db)
                                      (writes ONLY /status endpoint)
API server: "spec didn't change"    → NO reconcile triggered ✅
```

##### How to Enable It in CRD

```yaml
versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}          # Just this one line — that's all you need
```

##### What Changes When You Enable It

**Before** (no status sub-resource):
```
GET  /apis/mycompany.io/v1/namespaces/prod/databases/mydb    → reads spec + status together
PUT  /apis/mycompany.io/v1/namespaces/prod/databases/mydb    → writes spec + status together
```

**After** (status sub-resource enabled):
```
GET  /apis/mycompany.io/v1/namespaces/prod/databases/mydb          → reads spec + status
PUT  /apis/mycompany.io/v1/namespaces/prod/databases/mydb          → writes ONLY spec
                                                                      (status field is IGNORED)
PUT  /apis/mycompany.io/v1/namespaces/prod/databases/mydb/status   → writes ONLY status
                                                                      (spec field is IGNORED)
```

This means:
- When a **user** does `kubectl apply -f mydb.yaml` with both spec and status filled in → the status part is silently ignored (only spec is written)
- When a **controller** does `r.Status().Update(ctx, db)` → only the status field is written, spec is untouched

##### Separate RBAC — Critical for Security

Because `/status` is a separate endpoint, you can control who can update spec vs status independently:

```yaml
# Users / GitOps tools — can write spec, cannot write status
- apiGroups: ["mycompany.io"]
  resources: ["databases"]           # the main resource = spec
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Controller / Operator — can write status, but NOT arbitrarily change spec
- apiGroups: ["mycompany.io"]
  resources: ["databases/status"]    # the /status sub-resource
  verbs: ["get", "update", "patch"]
  # No "create" or "delete" — status cannot be independently created or deleted
```

This enforces a clean separation:
- **Humans / CI systems** own `spec` (desired state)
- **Controllers** own `status` (observed state)

##### How Controllers Write Status in Go

```go
// ❌ WRONG — writes entire object including spec, can cause conflicts
err = r.Update(ctx, db)

// ✅ CORRECT — writes ONLY the /status sub-resource
db.Status.Phase = "Running"
db.Status.ReadyReplicas = 3
err = r.Status().Update(ctx, db)

// ✅ ALSO CORRECT — patch only the changed fields (preferred over full update)
patch := client.MergeFrom(db.DeepCopy())
db.Status.Phase = "Running"
err = r.Status().Patch(ctx, db, patch)
```

##### Kubectl Commands for Status Sub-resource

```bash
# Read status (same as normal get — spec+status together)
kubectl get database mydb -o jsonpath='{.status}'

# Update status directly (simulates what a controller does — useful for testing)
kubectl patch database mydb \
  --subresource=status \
  --type=merge \
  -p '{"status":{"phase":"Running","readyReplicas":3}}'

# Watch status changes in real-time
kubectl get database mydb -w

# Wait for a specific status condition (uses /status internally)
kubectl wait database mydb \
  --for=condition=Ready \
  --timeout=120s
```

##### Visual Summary

```
┌──────────────────────────────────────────────────────────────┐
│                    Database CR Object                        │
│                                                              │
│  metadata: { name: mydb, ... }                               │
│                                                              │
│  spec:              ◄── Written via: PUT /databases/mydb     │
│    engine: postgres     Who writes: Users, GitOps, CI/CD     │
│    replicas: 3                                               │
│                                                              │
│  status:            ◄── Written via: PUT /databases/mydb/status│
│    phase: Running       Who writes: Controller/Operator only │
│    readyReplicas: 3                                          │
└──────────────────────────────────────────────────────────────┘
```

---

#### Scale Sub-resource — Deep Dive

##### What Problem Does It Solve?

The **Horizontal Pod Autoscaler (HPA)** is a built-in Kubernetes controller that scales workloads up/down based on CPU, memory, or custom metrics. It works out of the box for `Deployments`, `StatefulSets`, and `ReplicaSets` because they all expose a standard `/scale` endpoint.

Without the scale sub-resource, HPA has **no way to scale your custom resource**. Your `Database` CR would be invisible to HPA. With `subresources: scale`, your CRD gets the same `/scale` endpoint that Deployments have — and HPA can now scale it automatically.

##### How to Enable It in CRD

```yaml
subresources:
  status: {}            # Always enable this alongside scale
  scale:
    specReplicasPath: .spec.replicas         # WHERE to write the desired replica count
    statusReplicasPath: .status.readyReplicas # WHERE to read the current replica count
    labelSelectorPath: .status.labelSelector  # (Optional) pod selector for HPA metrics
```

These three paths tell HPA how to talk to your resource:
- **`specReplicasPath`**: HPA writes the desired replicas here (e.g., "scale to 5")
- **`statusReplicasPath`**: HPA reads the current replicas from here
- **`labelSelectorPath`**: HPA uses this label selector to find the pods for CPU/memory metrics

##### What the /scale Endpoint Looks Like

When you enable the scale sub-resource, the API server automatically creates a `/scale` endpoint that returns a `Scale` object:

```bash
# Read the scale object for your CR
kubectl get --raw \
  "/apis/mycompany.io/v1/namespaces/production/databases/mydb/scale" | jq .
```

```json
{
  "apiVersion": "autoscaling/v1",
  "kind": "Scale",
  "metadata": {
    "name": "mydb",
    "namespace": "production"
  },
  "spec": {
    "replicas": 3           ← current desired replicas (from .spec.replicas)
  },
  "status": {
    "replicas": 3,          ← current ready replicas (from .status.readyReplicas)
    "selector": "app=mydb"  ← pod selector (from .status.labelSelector)
  }
}
```

HPA reads and writes this `/scale` object — it never directly touches your spec or status.

##### kubectl scale — Works Automatically

Once the scale sub-resource is enabled, `kubectl scale` works on your custom resource just like it does on a Deployment:

```bash
# Scale manually using kubectl (writes to .spec.replicas via /scale)
kubectl scale database mydb --replicas=5

# This is equivalent to:
kubectl patch database mydb \
  --subresource=scale \
  --type=merge \
  -p '{"spec":{"replicas":5}}'
```

##### Wiring Up HPA to Your Custom Resource

```yaml
# HPA targeting your Database CR
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mydb-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: mycompany.io/v1
    kind: Database              # ← your CRD kind
    name: mydb
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70     # Scale up when CPU > 70%
```

##### How labelSelectorPath Works (Important Detail)

For HPA to calculate per-pod CPU/memory, it needs to know **which pods belong to your CR**. It uses the `labelSelectorPath` to find them.

Your controller must set `.status.labelSelector` to a label selector string:

```go
// In your reconciler — set the label selector in status
db.Status.LabelSelector = metav1.FormatLabelSelector(&metav1.LabelSelector{
    MatchLabels: map[string]string{
        "app": db.Name,
        "mycompany.io/database": db.Name,
    },
})
r.Status().Update(ctx, db)
```

```yaml
# Resulting status (HPA reads this to find pods):
status:
  readyReplicas: 3
  labelSelector: "app=mydb,mycompany.io/database=mydb"
```

HPA then queries the metrics API for pods matching that selector.

##### Full Scale Sub-resource CRD Configuration

```yaml
versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}
      scale:
        specReplicasPath: .spec.replicas           # HPA writes here
        statusReplicasPath: .status.readyReplicas  # HPA reads from here
        labelSelectorPath: .status.labelSelector   # HPA finds pods via this

    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                default: 1
          status:
            type: object
            properties:
              readyReplicas:
                type: integer
              labelSelector:
                type: string      # ← must exist for labelSelectorPath to work
```

##### Scale Sub-resource vs KEDA

On AKS, you have two options to auto-scale a custom resource:

| | HPA + Scale Sub-resource | KEDA ScaledObject |
|---|---|---|
| Trigger | CPU / Memory / Custom Metrics | Azure events (Service Bus, Queue, Event Hub…) |
| Scale to zero | ❌ No (min 1) | ✅ Yes |
| Setup complexity | Low (just enable sub-resource + HPA) | Medium (install KEDA, write TriggerAuth) |
| AKS integration | Basic | Deep Azure integration |
| Best for | CPU/memory-driven scaling | Event-driven, batch, queue-based workloads |

Use HPA + scale sub-resource for **resource-based scaling** (e.g., "scale DB read replicas when CPU > 70%").
Use KEDA for **event-driven scaling** (e.g., "scale workers based on Service Bus queue depth").

---

#### Sub-resources Summary

| | `/status` | `/scale` |
|---|---|---|
| **Purpose** | Separate spec writes from status writes | Allow HPA and `kubectl scale` to work |
| **Enabled by** | `subresources: status: {}` | `subresources: scale: {paths...}` |
| **Prevents** | Infinite reconcile loops | Nothing (additive feature) |
| **RBAC impact** | Separate `databases/status` permission | Separate `databases/scale` permission |
| **Controller code** | `r.Status().Update(ctx, db)` | `kubectl scale` / HPA writes `.spec.replicas` |
| **Always enable?** | ✅ Yes — always enable this | Only if you need HPA or `kubectl scale` |

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
