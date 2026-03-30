# 05 — Controllers: The Brain Behind CRDs

> **Series**: CRD & Operators on AKS | 

---

## What is a Controller?

A **controller** is a control loop that watches the state of your resources and makes changes to move the current state towards the desired state. Every built-in Kubernetes component (Deployment controller, ReplicaSet controller, etc.) is a controller.

For CRDs, **you write the controller** — it contains your domain-specific business logic.

---

## The Reconciliation Loop

The core of every controller is the **reconcile loop** — called whenever desired state (spec) might differ from actual state.

```
              ┌─────────────────────────────────────────┐
              │           Reconcile Loop                 │
              │                                          │
  Event ────► │  1. Get current state (from API/cluster) │
  (CR changed)│  2. Get desired state (from spec)        │
              │  3. Diff: desired vs actual              │
              │  4. Act: create/update/delete resources  │
              │  5. Update status                        │
              │  6. Return: Done / RequeueAfter(duration)│
              └─────────────────────────────────────────┘
```

**Key principle**: Reconcile must be **idempotent** — calling it N times must produce the same result as calling it once. Never assume a previous run succeeded.

---

## Informers & Watch Mechanism

Controllers do **not** poll the API server. They use **Informers** — a watch-based mechanism with a local cache.

```
                    ┌──────────────┐
                    │  API Server  │
                    └──────┬───────┘
                           │ Watch (persistent HTTP/2 stream)
                    ┌──────▼───────┐
                    │   Reflector  │ ← Receives ADD / UPDATE / DELETE events
                    └──────┬───────┘
                           │ Enqueues delta events
                    ┌──────▼───────┐
                    │  Delta FIFO  │ ← Ordered event queue
                    │    Queue     │
                    └──────┬───────┘
                           │ Processes events, populates cache
                    ┌──────▼───────┐
                    │   Indexer    │ ← Thread-safe in-memory object store
                    │  (Store)     │   Supports field/label indexing
                    └──────┬───────┘
                           │ Triggers handlers
                    ┌──────▼───────┐
                    │  Work Queue  │ ← Rate-limited, deduplicated
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Reconciler  │ ← Your business logic
                    └──────────────┘
```

### Why the Cache Matters (Critical Interview Topic)

- Controllers **always read from the local cache** (not the API server) for performance
- The cache is **eventually consistent** — it can lag by milliseconds to seconds
- `cache.WaitForCacheSync()` at startup ensures the cache is fully populated before reconciling begins
- After a `Create` or `Update`, the cache may not reflect it immediately — this is safe because the write itself triggers a new watch event → new reconcile

**Common bug pattern**: Reading from cache, assuming it reflects a change you just made in the same reconcile loop — it won't. Design your reconcile to handle this gracefully.

---

## Work Queue Properties

The work queue between the Informer and Reconciler has three critical properties:

### 1. Deduplication

If 10 events fire for the same object before the reconciler processes even one, only **one reconcile call** is made. This is why reconcile must be level-triggered (react to current state, not events).

### 2. Rate Limiting — Exponential Backoff

On error, items are re-queued with exponential backoff:

```
Error 1 → retry after 5ms
Error 2 → retry after 10ms
Error 3 → retry after 20ms
...
Error N → retry after max 1000s
```

### 3. Re-queue Semantics

```go
// Success — wait for next event (no requeue)
return ctrl.Result{}, nil

// Success — but recheck after 30s (drift detection)
return ctrl.Result{RequeueAfter: 30 * time.Second}, nil

// Immediate requeue (rare — prefer letting events drive reconcile)
return ctrl.Result{Requeue: true}, nil

// Error — automatic exponential backoff requeue
return ctrl.Result{}, fmt.Errorf("failed to reconcile: %w", err)
```

---

## Level-Triggered vs Edge-Triggered

This is one of the most important conceptual topics for senior interviews.

| | Edge-Triggered | Level-Triggered (Kubernetes) |
|---|---|---|
| Reacts to | The change event | The current state |
| Idempotent required | No | Yes |
| Misses events | Broken state | Self-heals on next reconcile |
| Example | Webhook that fires once | Kubernetes controller |

**Why level-triggered is better for self-healing systems:**

If a network partition causes your controller to miss an event, it will still reconcile correctly on the next trigger because it looks at the current state of the world — not what changed.

**Interview answer**: _"Why is the Kubernetes controller model level-triggered?"_
> Because it provides **self-healing**. If the controller crashes, restarts, or misses events, the next reconcile will compare desired vs actual state and fix any drift — no event replay needed.

---

## Controller Anatomy in Go (controller-runtime)

```go
package controllers

import (
    "context"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    dbv1 "github.com/mycompany/database-operator/api/v1"
)

const databaseFinalizer = "databases.mycompany.io/finalizer"

type DatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// Kubebuilder RBAC markers — used to generate ClusterRole YAML
// +kubebuilder:rbac:groups=databases.mycompany.io,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=databases.mycompany.io,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=databases.mycompany.io,resources=databases/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // Step 1: Fetch the resource (from cache)
    db := &dbv1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        if errors.IsNotFound(err) {
            // Deleted before we reconciled — nothing to do
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    // Step 2: Handle deletion
    if !db.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, db)
    }

    // Step 3: Add finalizer if missing
    if !controllerutil.ContainsFinalizer(db, databaseFinalizer) {
        controllerutil.AddFinalizer(db, databaseFinalizer)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{}, nil
    }

    // Step 4: Reconcile child resources
    if err := r.reconcileStatefulSet(ctx, db); err != nil {
        log.Error(err, "Failed to reconcile StatefulSet")
        r.setCondition(db, "Ready", "False", "StatefulSetFailed", err.Error())
        _ = r.Status().Update(ctx, db)
        return ctrl.Result{}, err
    }

    if err := r.reconcileService(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    // Step 5: Update status
    db.Status.Phase = "Running"
    r.setCondition(db, "Ready", "True", "Reconciled", "Database is ready")
    if err := r.Status().Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    // Step 6: Periodic drift detection requeue
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *dbv1.Database) error {
    desired := r.buildStatefulSet(db)

    // Owner reference for automatic garbage collection
    if err := ctrl.SetControllerReference(db, desired, r.Scheme); err != nil {
        return err
    }

    existing := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKeyFromObject(desired), existing)

    if errors.IsNotFound(err) {
        return r.Create(ctx, desired)
    }
    if err != nil {
        return err
    }

    // Only update if something changed
    if *existing.Spec.Replicas != *desired.Spec.Replicas {
        existing.Spec.Replicas = desired.Spec.Replicas
        return r.Update(ctx, existing)
    }

    return nil
}

func (r *DatabaseReconciler) handleDeletion(ctx context.Context, db *dbv1.Database) (ctrl.Result, error) {
    if controllerutil.ContainsFinalizer(db, databaseFinalizer) {
        if err := r.cleanupExternalResources(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
        controllerutil.RemoveFinalizer(db, databaseFinalizer)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }
    return ctrl.Result{}, nil
}

// SetupWithManager — registers the controller with the manager
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&dbv1.Database{}).                   // Watch primary resource
        Owns(&appsv1.StatefulSet{}).              // Watch owned StatefulSets (requeue parent on change)
        Owns(&corev1.Service{}).                  // Watch owned Services
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 3,           // Process up to 3 objects in parallel
        }).
        Complete(r)
}
```

---

## Watching External Resources

Sometimes you need to requeue your CR when an unowned resource changes (e.g., a shared ConfigMap):

```go
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&dbv1.Database{}).
        Watches(
            &source.Kind{Type: &corev1.ConfigMap{}},
            handler.EnqueueRequestsFromMapFunc(func(obj client.Object) []reconcile.Request {
                // Map ConfigMap → list of Database objects that use it
                dbList := &dbv1.DatabaseList{}
                mgr.GetClient().List(context.Background(), dbList,
                    client.InNamespace(obj.GetNamespace()),
                    client.MatchingLabels{"config": obj.GetName()},
                )
                requests := make([]reconcile.Request, len(dbList.Items))
                for i, db := range dbList.Items {
                    requests[i] = reconcile.Request{
                        NamespacedName: types.NamespacedName{
                            Name:      db.Name,
                            Namespace: db.Namespace,
                        },
                    }
                }
                return requests
            }),
        ).
        Complete(r)
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
| **05** | Controllers ← you are here |
| [06](./06-operator-pattern.md) | Operator Pattern |
| [07](./07-operator-frameworks.md) | Operator Frameworks |
| [08](./08-operator-lifecycle-manager.md) | OLM |
| [09](./09-advanced-operator-patterns.md) | Advanced Patterns |
| [10](./10-aks-crd-operator-focus.md) | AKS Focus |
| [11](./11-hands-on-examples.md) | Hands-On Examples |
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
