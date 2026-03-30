# 03 — Custom Resources (CR): Working with CRDs

> **Series**: CRD & Operators on AKS 

---

## What is a Custom Resource?

Once a CRD is registered, you create **Custom Resources (CRs)** — instances of that type — just like you create Pods or Deployments. The API server validates them against the CRD schema, stores them in etcd, and your controller watches and acts on them.

---

## Creating a Custom Resource

```yaml
apiVersion: mycompany.io/v1          # group/version from CRD
kind: Database                        # kind from CRD
metadata:
  name: my-postgres-db
  namespace: production
  labels:
    app: backend
    tier: database
  annotations:
    mycompany.io/backup-schedule: "0 2 * * *"
spec:
  engine: postgres
  replicas: 3
  version: "14.5"
  storageSize: "50Gi"
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
```

### Common kubectl Operations

```bash
# Apply / create
kubectl apply -f mydb.yaml

# List (uses additionalPrinterColumns from CRD)
kubectl get databases -n production
kubectl get databases -A                          # all namespaces
kubectl get db                                    # using shortName

# Describe (shows events, spec, status)
kubectl describe database my-postgres-db -n production

# Get raw JSON/YAML
kubectl get database my-postgres-db -o yaml
kubectl get database my-postgres-db -o json | jq .spec

# Edit live
kubectl edit database my-postgres-db -n production

# Delete
kubectl delete database my-postgres-db -n production

# Watch status in real-time
kubectl get database my-postgres-db -w -n production

# Explain fields (uses CRD schema descriptions)
kubectl explain database.spec
kubectl explain database.spec.engine
```

---

## RBAC for Custom Resources

RBAC for CRDs uses the same `ClusterRole` / `Role` mechanism, but references your custom `apiGroups` and `resources`.

```yaml
# Role for an application to read databases
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-reader
rules:
  - apiGroups: ["mycompany.io"]
    resources: ["databases"]
    verbs: ["get", "list", "watch"]

---
# Role for the operator/controller
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-operator
rules:
  - apiGroups: ["mycompany.io"]
    resources: ["databases"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["mycompany.io"]
    resources: ["databases/status"]          # Separate verb for status sub-resource
    verbs: ["get", "update", "patch"]
  - apiGroups: ["mycompany.io"]
    resources: ["databases/finalizers"]      # Separate verb for finalizers
    verbs: ["update"]
  - apiGroups: [""]                          # Core API group
    resources: ["pods", "services", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Bind the Role to the Operator's Service Account

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: database-operator-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: database-operator
subjects:
  - kind: ServiceAccount
    name: database-operator
    namespace: operators
```

### Verify RBAC with kubectl

```bash
# Can the operator list databases?
kubectl auth can-i list databases \
  --as=system:serviceaccount:operators:database-operator

# Can it update status?
kubectl auth can-i update databases/status \
  --as=system:serviceaccount:operators:database-operator -n production

# Can a developer create databases in their namespace?
kubectl auth can-i create databases \
  --as=system:serviceaccount:dev-team:developer -n dev-namespace
```

---

## Finalizers — Critical for Cleanup

Finalizers are **strings in `metadata.finalizers`** that block deletion until your controller removes them. They are the mechanism that lets operators run cleanup logic before a resource is fully deleted.

```yaml
metadata:
  finalizers:
    - databases.mycompany.io/cleanup        # Your controller must remove this
```

### Lifecycle with Finalizers

```
User: kubectl delete database mydb
        │
        ▼
API Server sets deletionTimestamp        ← resource is NOT deleted yet
        │
        ▼
Controller sees deletionTimestamp != nil
        │
        ▼
Controller runs cleanup:
  - Delete cloud database
  - Remove DNS records
  - Release licenses
  - Notify monitoring systems
        │
        ▼
Controller removes finalizer from metadata.finalizers
        │
        ▼
API Server: finalizers list is empty → permanently deletes the resource
```

### Go Code Pattern for Finalizers

```go
const myFinalizer = "databases.mycompany.io/finalizer"

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    db := &dbv1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Handle deletion
    if !db.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(db, myFinalizer) {
            // Run cleanup logic
            if err := r.cleanupExternalResources(ctx, db); err != nil {
                return ctrl.Result{}, err
            }
            // Remove finalizer — triggers actual deletion
            controllerutil.RemoveFinalizer(db, myFinalizer)
            return ctrl.Result{}, r.Update(ctx, db)
        }
        return ctrl.Result{}, nil
    }

    // Add finalizer on first reconcile
    if !controllerutil.ContainsFinalizer(db, myFinalizer) {
        controllerutil.AddFinalizer(db, myFinalizer)
        return ctrl.Result{}, r.Update(ctx, db)
    }

    // ... normal reconcile logic ...
    return ctrl.Result{}, nil
}
```

### Emergency Finalizer Removal (Use with Caution)

```bash
# If a controller is broken and finalizer is stuck, force-remove it
kubectl patch database mydb \
  -p '{"metadata":{"finalizers":[]}}' \
  --type=merge

# WARNING: This skips cleanup — only use when you've manually cleaned up
```

---

## Owner References — Automatic Garbage Collection

Owner references link **child resources** to a **parent custom resource**. When the parent is deleted, Kubernetes automatically garbage-collects all children.

```go
// In your controller — set owner reference when creating child resources
func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *dbv1.Database) error {
    sts := r.buildStatefulSet(db)

    // This sets ownerReferences on the StatefulSet pointing to the Database
    if err := ctrl.SetControllerReference(db, sts, r.Scheme); err != nil {
        return err
    }

    return r.Create(ctx, sts)
}
```

The resulting owner reference on the child resource:

```yaml
metadata:
  ownerReferences:
    - apiVersion: mycompany.io/v1
      kind: Database
      name: my-postgres-db
      uid: "abc-123-def-456"
      controller: true            # Only one controller per resource
      blockOwnerDeletion: true    # Block deletion until child is deleted
```

### Finalizer vs OwnerReference — Key Difference

| | OwnerReference | Finalizer |
|---|---|---|
| Mechanism | Declarative GC | Imperative hook |
| Custom logic | None | Full control |
| In-cluster only | Yes | No (can clean external resources) |
| Ordering | Kubernetes handles | You control |
| Use for | K8s child resources | External resources, notifications |

**Best practice**: Use both together — OwnerReferences for in-cluster children, Finalizers for external resource cleanup.

---

## Status Conventions

Follow the Kubernetes API conventions for status fields. Controllers write status; users should not.

```yaml
status:
  # Phase — simple string summary
  phase: Running                    # Pending | Running | Failed | Terminating

  # ObservedGeneration — tracks which spec version status reflects
  observedGeneration: 3

  # Conditions — detailed machine-readable status (preferred pattern)
  conditions:
    - type: Ready
      status: "True"                # "True" | "False" | "Unknown"
      reason: ReconcileSucceeded    # CamelCase, machine-readable
      message: "All replicas are ready"
      lastTransitionTime: "2026-03-30T10:00:00Z"

    - type: Progressing
      status: "False"
      reason: Idle
      message: "No rollout in progress"
      lastTransitionTime: "2026-03-30T09:55:00Z"
```

**Standard condition types** (match these from the Kubernetes conventions):
- `Ready` — is the resource fully operational?
- `Available` — is the resource available to serve requests?
- `Progressing` — is a rollout or change in progress?
- `Degraded` — is it running but with reduced capacity?

---

## Navigation

| # | Topic |
|---|---|
| [01](./01-kubernetes-extension-model.md) | Kubernetes Extension Model |
| [02](./02-crd-deep-dive.md) | CRD Deep Dive |
| **03** | Custom Resources ← you are here |
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
