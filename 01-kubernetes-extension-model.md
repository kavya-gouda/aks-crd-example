# 01 — Kubernetes Extension Model: Big Picture

> **Series**: CRD & Operators on AKS | Senior Engineer (12 YOE) Interview Prep

---

## Why Kubernetes is Extensible

Kubernetes was designed as a **platform to build platforms**. Its extension points allow you to:

- Add new resource types (CRDs)
- Add custom business logic (Controllers/Operators)
- Add admission/validation logic (Admission Webhooks)
- Add custom scheduling (Scheduler Extenders, Custom Schedulers)
- Add custom metrics (Custom Metrics API, External Metrics API)
- Add custom networking (CNI plugins)
- Add custom storage (CSI drivers)

---

## The Kubernetes API Extension Hierarchy

```
Kubernetes API Server
├── Built-in Resources (Pods, Deployments, Services...)
│   └── Served by: kube-apiserver
│
├── Custom Resources (CRDs)
│   └── Served by: kube-apiserver (via CRD mechanism)
│       └── Controller (reconciler) watches and acts on them
│
└── Aggregated API Servers (AA)
    └── Served by: your own API server
        └── Registered with APIService object
```

---

## When to Use CRD vs Aggregated API Server

| Feature | CRD | Aggregated API Server |
|---|---|---|
| Complexity | Low | High |
| Custom storage | No (uses etcd) | Yes (your own DB) |
| Custom sub-resources | Limited (/status, /scale) | Full flexibility |
| Versioning/conversion | Via webhook | Full control |
| Use case | 99% of cases | Custom storage/sub-resources needed |

**Interview Tip**: Know this tradeoff. Companies like to ask: _"When would you NOT use a CRD?"_

Answer: When you need a custom backing store (not etcd), full sub-resource flexibility, or extremely high throughput that etcd can't handle — then use an Aggregated API Server.

---

## Kubernetes Extension Points Summary

```
Extension Point          Tool / Mechanism              Common Use Cases
─────────────────────────────────────────────────────────────────────
Custom Resources         CRD                           App config, operators
Custom Controllers       controller-runtime            Business logic automation
Admission Control        ValidatingWebhookConfig       Policy enforcement
                         MutatingWebhookConfig         Auto-injection (sidecars)
Conversion               ConversionWebhook             CRD schema migration
Custom Scheduling        Scheduler Plugin / Extender   GPU placement, topology
Custom Metrics           Custom Metrics API            HPA based on app metrics
CNI                      CNI Plugin (Cilium, Azure CNI) Pod networking
CSI                      CSI Driver                    Custom storage backends
Authentication           TokenReview webhook           Custom auth providers
Authorization            SubjectAccessReview webhook   Custom authz logic
```

---

## Navigation

| # | Topic |
|---|---|
| **01** | Kubernetes Extension Model ← you are here |
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
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
