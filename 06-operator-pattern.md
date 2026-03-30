# 06 — Kubernetes Operator Pattern: Deep Dive

> **Series**: CRD & Operators on AKS | 

---

## What is an Operator?

An **Operator** = CRD + Controller that encodes **domain-specific operational knowledge** — not just Day 1 provisioning but full Day 2 operations.

| Day 1 Operations | Day 2 Operations |
|---|---|
| Install | Upgrade |
| Configure | Backup & Restore |
| Provision | Scale |
| Bootstrap | Self-heal |
| | Tune performance |
| | Certificate rotation |

A Helm chart is Day 1. An Operator is Day 1 + Day 2.

---

## The Operator Maturity Model

```
Level 1 — Basic Install
├── Automated application provisioning from CRD spec
└── Configuration management via spec fields

Level 2 — Seamless Upgrades
├── Patch and minor version upgrades
└── Configuration changes with rolling restarts

Level 3 — Full Lifecycle
├── App lifecycle (scale up/down, pause/resume)
├── Storage lifecycle (expand PVCs, migrate data)
└── Backup and restore operations

Level 4 — Deep Insights
├── Custom metrics exposed via Prometheus
├── Alerts based on operational knowledge
└── Log aggregation and anomaly detection

Level 5 — Auto Pilot
├── Horizontal/vertical auto-scaling
├── Auto config tuning based on load
└── Anomaly detection and self-remediation
```

**Interview Tip**: Product companies building platforms expect their operators to be at Level 3–4. Level 5 is rare but impressive to discuss.

---

## Operator Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Operator Pod                            │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │  Manager        │    │  Controllers/Reconcilers        │ │
│  │  (controller-   │    │                                  │ │
│  │   runtime)      │───►│  DatabaseReconciler              │ │
│  │                 │    │  BackupReconciler                │ │
│  │  - Leader       │    │  RestoreReconciler               │ │
│  │    Election     │    └─────────────────────────────────┘ │
│  │  - Health       │                                        │
│  │    Probes       │    ┌─────────────────────────────────┐ │
│  │  - Metrics      │    │  Webhooks                       │ │
│  │    Server       │    │  - Validating Webhook           │ │
│  └─────────────────┘    │  - Mutating Webhook             │ │
│                         │  - Conversion Webhook           │ │
│                         └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
         │                              │
         │ Watch/Reconcile              │ Validate/Mutate/Convert
         ▼                              ▼
   ┌──────────┐                  ┌──────────────┐
   │  Custom  │                  │  API Server  │
   │Resources │                  └──────────────┘
   └──────────┘
```

---

## Leader Election in Operators

Operators run **multiple replicas** for HA. But only ONE replica should reconcile at any time — otherwise you get race conditions and duplicate actions.

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    LeaderElection:          true,
    LeaderElectionID:        "database-operator-lock",
    LeaderElectionNamespace: "operators",
    // Uses Lease objects: coordination.k8s.io/v1/leases
    LeaseDuration: &leaseDuration,    // default: 15s
    RenewDeadline: &renewDeadline,    // default: 10s
    RetryPeriod:   &retryPeriod,      // default: 2s
})
```

### Leader Election Mechanism (Lease-based)

```
Replica A, B, C all start
        │
        ▼
All three try to CREATE/UPDATE Lease object:
  coordination.k8s.io/v1
  namespace: operators
  name: database-operator-lock
        │
        ▼
One wins (say Replica A) — becomes leader
  - Lease holderIdentity = "replica-a-pod-name"
  - Lease renewTime = now
        │
        ▼
Replica A renews lease every RenewDeadline (10s)
Replicas B, C watch — if lease not renewed within LeaseDuration (15s)...
        │
        ▼
Leader fails → lease expires after 15s
        │
        ▼
B and C compete again — one wins, becomes new leader
```

```bash
# Check who the current leader is
kubectl get lease database-operator-lock -n operators -o yaml
```

**Interview Tip**: _"What happens if reconcile takes longer than the lease duration?"_
> The leader's lease expires. Another replica becomes leader. Two leaders may reconcile the same resource simultaneously — causing conflicts. Fix: use `ctx.Done()` to cancel long operations when leadership is lost. Increase `LeaseDuration` if reconcile legitimately takes longer.

---

## Admission Webhooks — Validating & Mutating

Webhooks plug into the API server's admission chain — they run on every CREATE/UPDATE/DELETE before the object is persisted to etcd.

### Admission Chain Flow

```
kubectl apply -f db.yaml
        │
        ▼
API Server receives request
        │
        ▼
Authentication & Authorization
        │
        ▼
Mutating Admission Webhooks (in parallel)
  → Your MutatingWebhookConfiguration
  → Can MODIFY the object (inject fields, set defaults)
        │
        ▼
Object schema validation (against CRD openAPIV3Schema)
        │
        ▼
Validating Admission Webhooks (in parallel)
  → Your ValidatingWebhookConfiguration
  → Can REJECT the object (return error)
        │
        ▼
Persisted to etcd ✅
```

### Mutating Webhook — Example

```go
func (r *DatabaseWebhook) Default(ctx context.Context, obj runtime.Object) error {
    db := obj.(*v1.Database)

    // Set defaults that CRD defaulting can't express
    if db.Spec.BackupSchedule == "" {
        db.Spec.BackupSchedule = "0 2 * * *"
    }

    // Inject sidecar labels
    if db.Labels == nil {
        db.Labels = map[string]string{}
    }
    db.Labels["managed-by"] = "database-operator"

    // Normalize values
    db.Spec.Engine = strings.ToLower(db.Spec.Engine)

    return nil
}
```

### Validating Webhook — Example

```go
func (r *DatabaseWebhook) ValidateCreate(ctx context.Context, obj runtime.Object) error {
    db := obj.(*v1.Database)

    // Cross-field validation (can't do in CEL)
    if db.Spec.Replicas > 1 && db.Spec.StorageClass == "standard" {
        return fmt.Errorf("HA databases (replicas > 1) require premium storage class")
    }

    // External validation — check quota against an external system
    if err := r.quotaClient.CheckQuota(ctx, db.Namespace, "databases"); err != nil {
        return fmt.Errorf("quota exceeded: %w", err)
    }

    return nil
}

func (r *DatabaseWebhook) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) error {
    oldDB := oldObj.(*v1.Database)
    newDB := newObj.(*v1.Database)

    // Immutability check on update
    if oldDB.Spec.Engine != newDB.Spec.Engine {
        return fmt.Errorf("engine is immutable: cannot change from %s to %s",
            oldDB.Spec.Engine, newDB.Spec.Engine)
    }

    return nil
}

func (r *DatabaseWebhook) ValidateDelete(ctx context.Context, obj runtime.Object) error {
    db := obj.(*v1.Database)

    // Prevent deletion of production databases with active connections
    if db.Labels["environment"] == "production" && db.Status.ActiveConnections > 0 {
        return fmt.Errorf("cannot delete database with %d active connections",
            db.Status.ActiveConnections)
    }

    return nil
}
```

### Webhook Registration YAML

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: database-validator
  annotations:
    cert-manager.io/inject-ca-from: operators/database-operator-webhook-cert
webhooks:
  - name: validate.databases.mycompany.io
    admissionReviewVersions: ["v1"]
    sideEffects: None              # None | NoneOnDryRun | Some | Unknown
    failurePolicy: Fail            # Fail (reject on webhook error) or Ignore
    matchPolicy: Equivalent        # Equivalent matches all versions of the resource
    timeoutSeconds: 10
    rules:
      - apiGroups: ["mycompany.io"]
        apiVersions: ["v1"]
        resources: ["databases"]
        operations: ["CREATE", "UPDATE", "DELETE"]
    clientConfig:
      service:
        name: database-operator-webhook-service
        namespace: operators
        path: /validate-mycompany-io-v1-database
        port: 443
      caBundle: <base64-ca>        # Or use cert-manager annotation above
    namespaceSelector:             # Only validate in namespaces with this label
      matchLabels:
        enable-database-validation: "true"
    objectSelector:                # Only validate objects with this label
      matchExpressions:
        - key: mycompany.io/skip-validation
          operator: NotIn
          values: ["true"]
```

**Critical AKS consideration**: Webhooks must be **reachable from the API server**. In AKS:
- Use `cert-manager` to issue TLS certificates for webhook servers
- Network policies must allow traffic from `kube-apiserver` to webhook pod on port 443
- `failurePolicy: Fail` + unreachable webhook = **cluster-wide API outage for that resource type**

---

## Navigation

| # | Topic |
|---|---|
| [01](./01-kubernetes-extension-model.md) | Kubernetes Extension Model |
| [02](./02-crd-deep-dive.md) | CRD Deep Dive |
| [03](./03-custom-resources.md) | Custom Resources |
| [04](./04-crd-versioning-schema-validation.md) | Versioning & Validation |
| [05](./05-controllers.md) | Controllers |
| **06** | Operator Pattern ← you are here |
| [07](./07-operator-frameworks.md) | Operator Frameworks |
| [08](./08-operator-lifecycle-manager.md) | OLM |
| [09](./09-advanced-operator-patterns.md) | Advanced Patterns |
| [10](./10-aks-crd-operator-focus.md) | AKS Focus |
| [11](./11-hands-on-examples.md) | Hands-On Examples |
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
