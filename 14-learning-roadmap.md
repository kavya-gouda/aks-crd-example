# 14 — Learning Roadmap & Resources

> **Series**: CRD & Operators on AKS | Senior Engineer (12 YOE) Interview Prep

---

## 8-Week Learning Plan

### Week 1 — Kubernetes API Fundamentals

**Goal**: Be comfortable creating and manipulating CRDs and CRs with kubectl.

```
Topics:
├── Kubernetes API extension model (CRD vs Aggregated API Server)
├── CRD YAML anatomy: group, scope, versions, schema, subresources
├── Structural schema, defaulting, CEL validation rules
├── RBAC for custom resources (apiGroups, resources/status/finalizers)
└── Custom Resource lifecycle: create, update, delete, status

Hands-on:
├── Write a CRD from scratch for a "WebApp" resource
├── Apply it to a local cluster (kind or minikube)
├── Create CRs, explore with kubectl describe / explain / get -o yaml
└── Write ClusterRoles and test with kubectl auth can-i
```

---

### Week 2 — Controllers & Reconcile Loop

**Goal**: Understand the reconcile loop, Informers, and work queue deeply.

```
Topics:
├── Reconcile loop: idempotency, level-triggered model
├── Informers, Reflector, Delta FIFO Queue, Indexer (cache)
├── Work queue: deduplication, exponential backoff, rate limiting
├── ctrl.Result options: nil / Requeue / RequeueAfter / error
└── WaitForCacheSync and cache consistency

Hands-on:
├── Build a "HelloWorld" controller using controller-runtime
│   → Watches ConfigMaps, logs when they change
├── Add reconcile loop that creates a Secret when ConfigMap exists
└── Observe deduplication by making rapid changes
```

---

### Week 3 — Kubebuilder & Code Generation

**Goal**: Build a full CRD + Controller using Kubebuilder scaffold.

```
Topics:
├── kubebuilder init, create api, create webhook
├── Go struct markers → OpenAPI schema generation
├── Kubebuilder RBAC markers → ClusterRole generation
├── make generate && make manifests
└── Status conditions pattern (observedGeneration, Conditions slice)

Hands-on:
├── Scaffold a "Database" CRD + Controller with Kubebuilder
├── Add spec fields with markers (enum, minimum, default, pattern)
├── Implement reconcileStatefulSet with create-or-update pattern
├── Implement finalizer lifecycle
└── Add status conditions to track reconcile state
```

---

### Week 4 — Webhooks & Admission Control

**Goal**: Be able to write, register, and debug validating/mutating webhooks.

```
Topics:
├── Mutating webhook: Default() — injecting fields, normalizing values
├── Validating webhook: ValidateCreate/Update/Delete — cross-field, external checks
├── Conversion webhook: multi-version CRDs, hub-spoke model
├── Webhook registration YAML (failurePolicy, sideEffects, namespaceSelector)
└── TLS for webhooks: cert-manager Certificate + caBundle injection

Hands-on:
├── Add webhooks to Week 3 Database operator
├── Validate: engine is immutable, HA requires premium storage
├── Mutate: inject default labels and backup schedule
└── Test using kubectl apply --dry-run=server
```

---

### Week 5 — Advanced Operator Patterns

**Goal**: Know the patterns by name and be able to implement them.

```
Topics:
├── Finalizer + external resource cleanup
├── Owner references + garbage collection
├── Generation-based reconcile (observedGeneration)
├── Server-Side Apply (SSA) vs Update vs Patch
├── Pause reconciliation pattern
├── Composable operators (operator of operators)
└── Saga pattern for distributed workflows

Hands-on:
├── Add Azure Blob cleanup in finalizer (mock the Azure SDK call)
├── Implement generation check to skip redundant reconciles
├── Replace Update with SSA Patch in reconcileStatefulSet
└── Add pause annotation support
```

---

### Week 6 — AKS & Azure Integration

**Goal**: Deploy an operator to AKS with proper Azure integration.

```
Topics:
├── AKS architecture: control plane, node pools, VMSS
├── Azure Service Operator v2: CRD groups, ASO setup
├── Workload Identity: OIDC issuer, federated credentials, SA annotations
├── KEDA: ScaledObject, ScaledJob, TriggerAuthentication
├── cert-manager on AKS: ClusterIssuer (Let's Encrypt + Azure DNS)
└── AKS node placement: system pool, tolerations, topology spread

Hands-on:
├── Create AKS cluster with OIDC + Workload Identity enabled
├── Deploy Database operator with Managed Identity + Workload Identity
├── Apply ASO to create Azure SQL Server via CRD
├── Deploy KEDA + create ScaledObject for Service Bus trigger
└── Issue webhook TLS cert via cert-manager
```

---

### Week 7 — Production Readiness

**Goal**: Make your operator production-grade.

```
Topics:
├── Hardened Deployment YAML (security context, resource limits, PDB)
├── Custom Prometheus metrics (GaugeVec, HistogramVec, CounterVec)
├── Structured logging (key-value pairs, log levels)
├── Kubernetes Events via Recorder
├── Envtest: testing controllers without a real cluster
├── OLM bundle: make bundle, CSV creation
└── CRD upgrade safety checklist

Hands-on:
├── Add databasesTotal, reconcileDuration, reconcileErrors metrics
├── Annotate Deployment with prometheus.io/scrape
├── Write envtest suite for create/update/delete scenarios
└── Run operator-sdk scorecard against your bundle
```

---

### Week 8 — Interview Preparation

**Goal**: Be able to answer all 13 Q&As confidently and design an operator on a whiteboard.

```
Practice:
├── Answer each Q&A in 13-interview-qa.md out loud (2-3 min each)
├── Design a Kafka operator from scratch on paper (CRD, controller, webhooks)
├── Design a multi-region app deployment operator
├── Build a mini operator from scratch in 2 hours (timed challenge)
└── Review source code of at least 2 real-world operators
```

---

## Key Repositories to Study (Source Code)

| Repository | Why Study It |
|---|---|
| [kubernetes-sigs/controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) | Core library — read the reconciler, informer, and work queue source |
| [kubernetes-sigs/kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) | Scaffolding, e2e test patterns, project layout |
| [cert-manager/cert-manager](https://github.com/cert-manager/cert-manager) | Best-in-class production operator — CRD design, reconciler patterns, status conditions |
| [kedacore/keda](https://github.com/kedacore/keda) | Complex CRD design, AKS integration, ScaledObject controller |
| [Azure/azure-service-operator](https://github.com/Azure/azure-service-operator) | ASO v2 — multi-resource CRDs, Azure SDK integration, code generation |
| [prometheus-operator/prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) | Complex CRD hierarchy, multi-controller operator |
| [strimzi/strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator) | Kafka operator — excellent Day 2 operations design |

---

## Essential kubectl Cheat Sheet

```bash
# ── CRD Operations ─────────────────────────────────────────────────────────
kubectl get crds                                           # List all CRDs
kubectl describe crd databases.mycompany.io               # CRD details + versions
kubectl get databases -A                                   # All CRs across namespaces
kubectl get databases -n production -o wide                # Wide output (priority columns)
kubectl explain database.spec                              # Schema documentation
kubectl explain database.spec.engine --recursive           # Recursive field docs
kubectl api-resources --api-group=mycompany.io             # List all CRDs in your group

# ── Custom Resource Operations ─────────────────────────────────────────────
kubectl apply -f mydb.yaml
kubectl apply --dry-run=server -f mydb.yaml               # Validate against live schema
kubectl get database mydb -o yaml                         # Full YAML dump
kubectl get database mydb -o jsonpath='{.status.phase}'   # Specific field
kubectl patch database mydb --type=merge \
  -p '{"spec":{"replicas":5}}'
kubectl wait database mydb --for=condition=Ready \
  --timeout=120s                                          # Wait for condition

# ── Finalizer Debugging ─────────────────────────────────────────────────────
kubectl get database mydb \
  -o jsonpath='{.metadata.finalizers}'                    # Check finalizers
kubectl patch database mydb \
  -p '{"metadata":{"finalizers":[]}}' --type=merge       # EMERGENCY: force remove

# ── Operator Debugging ──────────────────────────────────────────────────────
kubectl logs deploy/database-operator -n operators -f     # Follow logs
kubectl logs deploy/database-operator -n operators -p     # Previous container (crash)
kubectl describe pod <operator-pod> -n operators          # Events, container state
kubectl get lease -n operators                            # Leader election lease
kubectl get lease database-operator-lock -n operators \
  -o jsonpath='{.spec.holderIdentity}'                    # Current leader

# ── RBAC Verification ───────────────────────────────────────────────────────
kubectl auth can-i list databases \
  --as=system:serviceaccount:operators:database-operator
kubectl auth can-i update databases/status \
  --as=system:serviceaccount:operators:database-operator -n production
kubectl auth can-i update databases/finalizers \
  --as=system:serviceaccount:operators:database-operator

# ── Webhook Debugging ───────────────────────────────────────────────────────
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
kubectl apply --dry-run=server -f mydb.yaml              # Triggers webhook

# ── AKS / Azure ────────────────────────────────────────────────────────────
az aks update -g myRG -n myAKS \
  --enable-oidc-issuer --enable-workload-identity
az aks show -g myRG -n myAKS \
  --query "oidcIssuerProfile.issuerUrl" -o tsv
az identity federated-credential list \
  --identity-name database-operator-identity \
  --resource-group myRG

# ── API Deprecation Checks (before AKS upgrade) ────────────────────────────
pluto detect-all-in-cluster --target-versions k8s=v1.30.0
kubectl api-resources --api-group=apiextensions.k8s.io
kubectl get crds -o json | \
  jq '.items[] | {name: .metadata.name, versions: [.spec.versions[].name]}'
```

---

## Recommended Certifications

| Certification | Relevance |
|---|---|
| **CKAD** — Certified Kubernetes Application Developer | Core Kubernetes — Pods, Services, RBAC, ConfigMaps |
| **CKA** — Certified Kubernetes Administrator | Cluster management, networking, storage, troubleshooting |
| **CKS** — Certified Kubernetes Security Specialist | Webhook security, RBAC hardening, seccomp, network policies |
| **AZ-104** — Azure Administrator | AKS fundamentals, Azure RBAC, Managed Identities, networking |
| **AZ-305** — Azure Solutions Architect | Enterprise AKS architecture, Landing Zones, multi-region design |

**Recommended order**: CKAD → CKA → AZ-104 → CKS → AZ-305

---

## Books & Documentation

| Resource | Link |
|---|---|
| Kubernetes API Conventions | [github.com/kubernetes/community/api-conventions.md](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) |
| Kubebuilder Book | [book.kubebuilder.io](https://book.kubebuilder.io) |
| Operator SDK Docs | [sdk.operatorframework.io](https://sdk.operatorframework.io) |
| controller-runtime Godoc | [pkg.go.dev/sigs.k8s.io/controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime) |
| ASO v2 Docs | [azure.github.io/azure-service-operator](https://azure.github.io/azure-service-operator) |
| KEDA Docs | [keda.sh/docs](https://keda.sh/docs) |
| cert-manager Docs | [cert-manager.io/docs](https://cert-manager.io/docs) |
| AKS Workload Identity | [azure.github.io/azure-workload-identity](https://azure.github.io/azure-workload-identity) |

---

## Series Complete

You have now covered:

```
01  Kubernetes Extension Model      — When and why to use CRDs
02  CRD Deep Dive                   — Full schema anatomy, sub-resources, printer columns
03  Custom Resources                — CRUD, RBAC, finalizers, owner references, status conventions
04  Versioning & Schema Validation  — Multi-version CRDs, conversion webhooks, CEL rules
05  Controllers                     — Reconcile loop, Informers, cache, work queue, level-triggered model
06  Operator Pattern                — Maturity model, leader election, admission webhooks
07  Operator Frameworks             — Kubebuilder, Operator SDK (Go/Helm/Ansible), main.go
08  OLM                             — CatalogSource, CSV, Subscription, bundle creation
09  Advanced Patterns               — Saga, composable operators, SSA, generation checks, circuit breaker
10  AKS Focus                       — ASO v2, Workload Identity, KEDA, cert-manager, Gatekeeper, upgrade checks
11  Hands-On Examples               — PostgreSQL operator, Azure backup, envtest, multi-region design
12  Production Best Practices       — Hardened deployment, metrics, CRD upgrade checklist, security
13  Interview Q&A                   — 14 deep questions with full answers + rapid-fire concepts
14  Learning Roadmap                — 8-week plan, repos, cheat sheet, certifications  ← you are here
```

---

*Guide Version: 1.0 | Updated: March 2026 | For Senior Engineers (12 YOE) | AKS + Product Company Interview Focus*
