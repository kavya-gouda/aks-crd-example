# 09 — Advanced Operator Patterns

> **Series**: CRD & Operators on AKS | 

---

## Pattern 1: Composable Operators (Operator of Operators)

Build higher-level CRDs that create lower-level CRDs — allowing operators to compose:

```yaml
# DatabaseCluster CR creates multiple Database CRs + a LoadBalancer CR
apiVersion: mycompany.io/v1
kind: DatabaseCluster
metadata:
  name: orders-cluster
  namespace: production
spec:
  primaryDatabase:
    engine: postgres
    replicas: 1
    version: "15"
  replicaDatabases:
    count: 2
    engine: postgres
  loadBalancer:
    type: azure                       # Creates AzureLoadBalancer CR via ASO
  backup:
    enabled: true
    storageAccount: mybackupaccount
```

The `DatabaseCluster` controller creates child `Database` CRs, watches their status conditions, and aggregates them into the cluster's own status.

---

## Pattern 2: Saga Pattern for Distributed Transactions

Use multiple CRs and controllers to implement long-running distributed workflows:

```
DatabaseProvision (orchestrator CR)
├── Creates: CloudDatabaseRequest CR  → handled by cloud-db controller
├── Creates: SecretsSetup CR          → handled by secrets controller
├── Creates: DNSRecord CR             → handled by dns controller
└── Monitors: All child CRs → aggregates status → marks itself Ready
```

```go
// Orchestrator controller — watches child CRs
func (r *DatabaseProvisionReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    provision := &v1.DatabaseProvision{}
    r.Get(ctx, req.NamespacedName, provision)

    // Create CloudDatabaseRequest if not exists
    r.ensureCloudDatabaseRequest(ctx, provision)

    // Check CloudDatabaseRequest status
    cdr := &v1.CloudDatabaseRequest{}
    r.Get(ctx, client.ObjectKey{Name: provision.Name + "-cloud", Namespace: provision.Namespace}, cdr)

    if !isReady(cdr) {
        provision.Status.Phase = "WaitingForCloudDB"
        r.Status().Update(ctx, provision)
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }

    // Cloud DB ready — create SecretsSetup
    r.ensureSecretsSetup(ctx, provision, cdr.Status.ConnectionString)

    // ... continue saga steps
    return ctrl.Result{}, nil
}
```

---

## Pattern 3: Status Conditions (Kubernetes Convention)

Always model status with **conditions** — they are machine-readable, composable, and follow the Kubernetes API convention.

```go
const (
    ConditionReady       = "Ready"
    ConditionProgressing = "Progressing"
    ConditionDegraded    = "Degraded"
    ConditionAvailable   = "Available"
)

func setCondition(db *v1.Database, condType, status, reason, message string) {
    now := metav1.Now()
    for i, c := range db.Status.Conditions {
        if c.Type == condType {
            if string(c.Status) != status {
                db.Status.Conditions[i].LastTransitionTime = now
            }
            db.Status.Conditions[i].Status = metav1.ConditionStatus(status)
            db.Status.Conditions[i].Reason = reason
            db.Status.Conditions[i].Message = message
            return
        }
    }
    db.Status.Conditions = append(db.Status.Conditions, metav1.Condition{
        Type:               condType,
        Status:             metav1.ConditionStatus(status),
        Reason:             reason,
        Message:            message,
        LastTransitionTime: now,
        ObservedGeneration: db.Generation,
    })
}
```

### Status Condition Rules

- `status` is always `"True"`, `"False"`, or `"Unknown"` — never custom strings
- `reason` is CamelCase machine-readable string (no spaces)
- `message` is human-readable (can have spaces/details)
- Only update `lastTransitionTime` when `status` actually changes
- Set `observedGeneration` to track which spec version the condition refers to

```bash
# Query conditions in kubectl
kubectl get database mydb -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
kubectl wait database mydb --for=condition=Ready --timeout=120s
```

---

## Pattern 4: Generation-Based Reconcile

Avoid unnecessary reconcile work when only status changed (not spec):

```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    db := &v1.Database{}
    r.Get(ctx, req.NamespacedName, db)

    // Skip full reconcile if spec hasn't changed
    if db.Generation == db.Status.ObservedGeneration {
        // Spec unchanged — only run lightweight health/drift check
        return r.performHealthCheck(ctx, db)
    }

    // Full reconcile — spec has changed
    if err := r.reconcileAll(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    // Record which spec generation we just reconciled
    patch := client.MergeFrom(db.DeepCopy())
    db.Status.ObservedGeneration = db.Generation
    return ctrl.Result{}, r.Status().Patch(ctx, db, patch)
}
```

**Why this matters**: Status updates (from your own controller) also trigger reconcile calls. Without generation checks, you enter infinite reconcile loops.

---

## Pattern 5: Patch vs Update — Always Prefer Patch

Full `Update` replaces the entire object and can clobber concurrent changes. Use `Patch` instead.

```go
// BAD — full Update overwrites all fields, can cause conflicts
err = r.Update(ctx, db)

// GOOD — MergePatch: only sends changed fields
patch := client.MergeFrom(db.DeepCopy())
db.Status.Phase = "Running"
db.Status.ReadyReplicas = 3
err = r.Status().Patch(ctx, db, patch)

// GOOD — JSONPatch: explicit operations
patch := client.RawPatch(types.JSONPatchType, []byte(`[
  {"op": "replace", "path": "/status/phase", "value": "Running"},
  {"op": "replace", "path": "/status/readyReplicas", "value": 3}
]`))
err = r.Status().Patch(ctx, db, patch)
```

---

## Pattern 6: Server-Side Apply (SSA) — Modern Best Practice

Server-Side Apply is the modern replacement for `kubectl apply` and controller `Update`. The API server tracks **field ownership** per manager.

```go
// SSA — only declare fields your controller OWNS
statefulSet := &appsv1.StatefulSet{
    TypeMeta: metav1.TypeMeta{
        APIVersion: "apps/v1",
        Kind:       "StatefulSet",
    },
    ObjectMeta: metav1.ObjectMeta{
        Name:      db.Name,
        Namespace: db.Namespace,
    },
    Spec: appsv1.StatefulSetSpec{
        Replicas: &db.Spec.Replicas,
        // Only include fields YOU manage — leave others (managed by HPA, etc.) alone
    },
}

err = r.Patch(ctx, statefulSet, client.Apply,
    client.FieldOwner("database-controller"),  // Your manager name
    client.ForceOwnership,                     // Take ownership of conflicting fields
)
```

**Benefits of SSA:**
- No conflict errors when multiple controllers manage different fields
- Automatic detection of fields you own but removed (they get cleaned up)
- Works naturally with drift detection (apply desired state repeatedly)

---

## Pattern 7: Pause Reconciliation

Allow users to pause reconciliation for maintenance, debugging, or emergency stop:

```go
// Check for pause annotation at the top of Reconcile
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    db := &v1.Database{}
    r.Get(ctx, req.NamespacedName, db)

    // Support both annotation and spec field for pausing
    if db.Annotations["mycompany.io/reconcile-paused"] == "true" || db.Spec.Paused {
        r.Recorder.Event(db, corev1.EventTypeNormal, "Paused",
            "Reconciliation paused — remove annotation to resume")
        setCondition(db, "Paused", "True", "PausedByAnnotation", "Reconciliation is paused")
        r.Status().Update(ctx, db)
        return ctrl.Result{}, nil                   // No requeue — wait for annotation removal
    }

    // ... normal reconcile ...
}
```

```bash
# Pause
kubectl annotate database mydb mycompany.io/reconcile-paused=true

# Resume
kubectl annotate database mydb mycompany.io/reconcile-paused-
```

---

## Pattern 8: Multi-Resource Reconcile with Requeue Chaining

For complex operators that manage many child resources, chain reconcile steps with explicit phase tracking:

```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    db := &v1.Database{}
    r.Get(ctx, req.NamespacedName, db)

    switch db.Status.Phase {
    case "":
        return r.initialize(ctx, db)             // Sets phase to "Initializing"

    case "Initializing":
        return r.provision(ctx, db)              // Creates PVCs, Secrets

    case "Provisioning":
        return r.waitForProvisioning(ctx, db)    // Checks if StatefulSet is ready

    case "Running":
        return r.runningReconcile(ctx, db)       // Drift detection

    case "Upgrading":
        return r.handleUpgrade(ctx, db)          // Rolling upgrade logic

    case "Failed":
        return r.handleFailure(ctx, db)          // Retry / alert

    default:
        return ctrl.Result{}, fmt.Errorf("unknown phase: %s", db.Status.Phase)
    }
}
```

---

## Pattern 9: Circuit Breaker

Prevent a buggy operator from hammering the API server or external services on repeated failures:

```go
type DatabaseReconciler struct {
    client.Client
    Scheme         *runtime.Scheme
    failureTracker map[string]int     // namespace/name → consecutive failure count
    mu             sync.Mutex
}

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    key := req.NamespacedName.String()

    r.mu.Lock()
    failures := r.failureTracker[key]
    r.mu.Unlock()

    // Circuit open — skip reconcile for this object
    if failures > 10 {
        log.FromContext(ctx).Info("Circuit breaker open — too many failures", "key", key)
        r.Recorder.Event(db, corev1.EventTypeWarning, "CircuitOpen",
            "Too many consecutive failures — pausing reconciliation for this resource")
        return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
    }

    _, err := r.reconcile(ctx, req)
    if err != nil {
        r.mu.Lock()
        r.failureTracker[key]++
        r.mu.Unlock()
        return ctrl.Result{}, err
    }

    // Reset on success
    r.mu.Lock()
    delete(r.failureTracker, key)
    r.mu.Unlock()
    return ctrl.Result{}, nil
}
```

---

## Navigation

| # | Topic |
|---|---|
| [01](./01-kubernetes-extension-model.md) | Kubernetes Extension Model |
| [02](./02-crd-deep-dive.md) | CRD Deep Dive |
| [03](./03-custom-resources.md) | Custom Resources |
| [04](./04-crd-versioning-schema-validation.md) | Versioning & Validation |
| [05](./05-controllers.md) | Controllers |
| [06](./06-operator-pattern.md) | Operator Pattern |
| [07](./07-operator-frameworks.md) | Operator Frameworks |
| [08](./08-operator-lifecycle-manager.md) | OLM |
| **09** | Advanced Patterns ← you are here |
| [10](./10-aks-crd-operator-focus.md) | AKS Focus |
| [11](./11-hands-on-examples.md) | Hands-On Examples |
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
