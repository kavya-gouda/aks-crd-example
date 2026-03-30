# 12 — Production Best Practices

> **Series**: CRD & Operators on AKS 
---

## Operator Deployment — Production-Grade YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-operator
  namespace: operators
spec:
  replicas: 2                              # HA — leader election handles active/standby
  selector:
    matchLabels:
      app: database-operator
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0                    # Never take both replicas down simultaneously
      maxSurge: 1
  template:
    metadata:
      labels:
        app: database-operator
        azure.workload.identity/use: "true"  # AKS Workload Identity
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: database-operator

      # Security context — run as non-root
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        runAsGroup: 65532
        fsGroup: 65532
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: manager
          image: mycompany/database-operator:v1.0.0
          imagePullPolicy: Always

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          ports:
            - containerPort: 9443    # Webhook server (TLS)
              name: webhook-server
            - containerPort: 8080    # Prometheus metrics
              name: metrics
            - containerPort: 8081    # Health probes
              name: health

          livenessProbe:
            httpGet:
              path: /healthz
              port: 8081
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /readyz
              port: 8081
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3

          args:
            - --leader-elect
            - --health-probe-bind-address=:8081
            - --metrics-bind-address=:8080
            - --zap-log-level=info
            - --max-concurrent-reconciles=5

          volumeMounts:
            - mountPath: /tmp/k8s-webhook-server/serving-certs
              name: webhook-cert
              readOnly: true
            - mountPath: /tmp
              name: tmp-dir              # Needed for readOnlyRootFilesystem

      volumes:
        - name: webhook-cert
          secret:
            secretName: database-operator-webhook-tls
        - name: tmp-dir
          emptyDir: {}

      # AKS-specific placement
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
      nodeSelector:
        kubernetes.azure.com/mode: system

      # Spread replicas across availability zones
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: database-operator
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: database-operator

      terminationGracePeriodSeconds: 60   # Allow in-flight reconciles to finish

---
# Pod Disruption Budget — prevent both replicas going down at once
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: database-operator-pdb
  namespace: operators
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: database-operator
```

---

## Observability — Metrics, Logs, Traces

### Custom Prometheus Metrics

```go
package controllers

import (
    "github.com/prometheus/client_golang/prometheus"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
    // Gauge: current number of databases by phase and engine
    databasesTotal = prometheus.NewGaugeVec(prometheus.GaugeOpts{
        Namespace: "database_operator",
        Name:      "databases_total",
        Help:      "Total number of Database resources by phase and engine",
    }, []string{"phase", "engine", "namespace"})

    // Histogram: how long each reconcile takes
    reconcileDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
        Namespace: "database_operator",
        Name:      "reconcile_duration_seconds",
        Help:      "Duration of reconcile operations in seconds",
        Buckets:   []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5, 10},
    }, []string{"result", "controller"})

    // Counter: total reconcile errors
    reconcileErrors = prometheus.NewCounterVec(prometheus.CounterOpts{
        Namespace: "database_operator",
        Name:      "reconcile_errors_total",
        Help:      "Total number of reconcile errors",
    }, []string{"reason", "controller"})

    // Gauge: leader election state
    isLeader = prometheus.NewGauge(prometheus.GaugeOpts{
        Namespace: "database_operator",
        Name:      "is_leader",
        Help:      "1 if this instance is the leader, 0 otherwise",
    })
)

func init() {
    metrics.Registry.MustRegister(
        databasesTotal,
        reconcileDuration,
        reconcileErrors,
        isLeader,
    )
}

// Wrap reconcile with metrics
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    start := time.Now()

    result, err := r.reconcile(ctx, req)

    duration := time.Since(start).Seconds()
    resultLabel := "success"
    if err != nil {
        resultLabel = "error"
        reconcileErrors.WithLabelValues(getErrorType(err), "database").Inc()
    }
    reconcileDuration.WithLabelValues(resultLabel, "database").Observe(duration)

    return result, err
}
```

### Structured Logging

```go
// Use structured logging — key-value pairs for queryability
log := log.FromContext(ctx).WithValues(
    "database", req.NamespacedName,
    "generation", db.Generation,
    "phase", db.Status.Phase,
)

log.Info("Starting reconcile")
log.Info("StatefulSet reconciled", "replicas", *sts.Spec.Replicas)
log.Error(err, "Failed to update status", "error_type", reflect.TypeOf(err).String())
```

### Kubernetes Events (for `kubectl describe`)

```go
// Emit events — shows up in `kubectl describe database mydb`
r.Recorder.Event(db, corev1.EventTypeNormal, "Provisioning", "Creating StatefulSet")
r.Recorder.Event(db, corev1.EventTypeNormal, "Ready", "Database is ready")
r.Recorder.Eventf(db, corev1.EventTypeWarning, "BackupFailed",
    "Backup failed: %s — retrying in 1 hour", err.Error())
```

---

## CRD Upgrade Safety Checklist

```
═══ Before upgrading CRD schema ═══════════════════════════════════════════
□ New fields are optional (not in required[]) OR have default values
□ Validation rules are not tightened for values already stored in etcd
□ Removed fields are handled by conversion webhook
□ Storage migration plan exists if changing storage version
□ Conversion webhook is deployed and healthy BEFORE CRD update
□ Rollback plan: old CRD version ready to re-apply
□ Tested on staging with production-like data volume

═══ Before AKS cluster version upgrade ════════════════════════════════════
□ Run: pluto detect-all-in-cluster --target-versions k8s=<new-version>
□ All CRDs use apiextensions.k8s.io/v1 (not v1beta1 — removed in 1.22)
□ All webhook configs use admissionregistration.k8s.io/v1
□ All RBAC uses rbac.authorization.k8s.io/v1
□ Operator's release notes confirm support for target k8s version
□ cert-manager, KEDA, ASO compatibility verified
□ Tested in staging AKS cluster at target version
□ Node pool upgrade strategy: upgrade user pools first, then system pool
□ PodDisruptionBudgets set correctly so operator stays available during upgrade

═══ After CRD or operator upgrade ════════════════════════════════════════
□ All CRs still accessible: kubectl get <resource> -A
□ Status conditions are healthy: kubectl get <resource> -A -o wide
□ Controller logs show no errors: kubectl logs deploy/<operator> -n operators
□ Metrics normal: check reconcile_errors_total
□ Webhook functional: kubectl apply --dry-run=server -f sample-cr.yaml
```

---

## Security Best Practices

### Minimal RBAC

```yaml
# Principle of least privilege — only what the operator needs
rules:
  # Your CRD — full access
  - apiGroups: ["mycompany.io"]
    resources: ["databases"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["mycompany.io"]
    resources: ["databases/status"]
    verbs: ["get", "update", "patch"]          # Not create or delete
  - apiGroups: ["mycompany.io"]
    resources: ["databases/finalizers"]
    verbs: ["update"]                          # Only update, not delete

  # Core resources — restricted to what's needed
  - apiGroups: ["apps"]
    resources: ["statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
    # NO delete — let owner references handle GC

  # Secrets — read only (don't let operator create secrets if avoidable)
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
```

### Container Security

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532                    # Non-root UID (distroless default)
  readOnlyRootFilesystem: true        # Prevent filesystem writes
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]                     # Drop all Linux capabilities
  seccompProfile:
    type: RuntimeDefault              # Use containerd's default seccomp profile
```

### Image Security

```yaml
# Use digest instead of tag (immutable reference)
image: mycompany/database-operator@sha256:abc123...

# Or pin to specific version tag (not latest)
image: mycompany/database-operator:v1.2.3

# Add image pull policy
imagePullPolicy: Always              # Re-check on every pod start
```

---

## Operator Versioning & Upgrade Strategy

### Semantic Versioning for Operators

```
v1.0.0 — Stable, production-ready
v1.1.0 — New features, backward compatible CRD
v2.0.0 — Breaking CRD changes (requires conversion webhook)
v1.0.1 — Bug fixes, no CRD changes
```

### Zero-Downtime Operator Upgrade

```bash
# 1. Deploy new operator version alongside old (different Deployment name)
kubectl apply -f new-operator-deployment.yaml

# 2. Verify new operator is healthy
kubectl rollout status deployment/database-operator-v2 -n operators

# 3. Old operator resigns leadership (if deployment name matches leader lock ID)
# OR: scale down old operator — new one wins election
kubectl scale deployment database-operator --replicas=0 -n operators

# 4. Verify no reconcile errors after switchover
kubectl logs deployment/database-operator-v2 -n operators | grep -i error

# 5. Remove old operator
kubectl delete deployment database-operator -n operators
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
| [09](./09-advanced-operator-patterns.md) | Advanced Patterns |
| [10](./10-aks-crd-operator-focus.md) | AKS Focus |
| [11](./11-hands-on-examples.md) | Hands-On Examples |
| **12** | Production Best Practices ← you are here |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
