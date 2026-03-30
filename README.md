# Kubernetes CRD & Operators on AKS — Complete Study Guide

> For Senior Engineers (12 YOE) | Product-Based Company Interview Prep | AKS Focus

---

## What's in This Guide

A complete deep-dive series covering Kubernetes Custom Resource Definitions (CRDs), the Operator pattern, and Azure Kubernetes Service (AKS) integration — structured for engineers preparing for senior/staff-level product company interviews.

Each document is self-contained with full YAML examples, Go code, diagrams, and interview tips.

---

## Documents

| # | File | Topics Covered |
|---|---|---|
| 01 | [01-kubernetes-extension-model.md](./01-kubernetes-extension-model.md) | Extension model, CRD vs Aggregated API Server, extension points table |
| 02 | [02-crd-deep-dive.md](./02-crd-deep-dive.md) | Full CRD YAML anatomy, scope, structural schema, sub-resources, printer columns |
| 03 | [03-custom-resources.md](./03-custom-resources.md) | Creating CRs, kubectl operations, RBAC, finalizers, owner references, status conventions |
| 04 | [04-crd-versioning-schema-validation.md](./04-crd-versioning-schema-validation.md) | Multi-version CRDs, conversion webhooks, CEL validation rules, defaulting, migration checklist |
| 05 | [05-controllers.md](./05-controllers.md) | Reconcile loop, Informers, Delta FIFO, cache, work queue, level-triggered model, Go code |
| 06 | [06-operator-pattern.md](./06-operator-pattern.md) | Operator maturity model, architecture, leader election, validating/mutating webhooks |
| 07 | [07-operator-frameworks.md](./07-operator-frameworks.md) | Kubebuilder, Operator SDK (Go/Helm/Ansible), type markers, main.go setup |
| 08 | [08-operator-lifecycle-manager.md](./08-operator-lifecycle-manager.md) | OLM CRDs, CatalogSource, CSV, Subscription, bundle creation |
| 09 | [09-advanced-operator-patterns.md](./09-advanced-operator-patterns.md) | Saga, composable operators, SSA, generation checks, pause pattern, circuit breaker |
| 10 | [10-aks-crd-operator-focus.md](./10-aks-crd-operator-focus.md) | ASO v2, Workload Identity, KEDA, cert-manager, Gatekeeper, AKS upgrade checks, network policy |
| 11 | [11-hands-on-examples.md](./11-hands-on-examples.md) | PostgreSQL cluster operator, Azure backup integration, envtest, multi-region app CRD |
| 12 | [12-production-best-practices.md](./12-production-best-practices.md) | Hardened Deployment YAML, Prometheus metrics, structured logging, upgrade checklist, security |
| 13 | [13-interview-qa.md](./13-interview-qa.md) | 14 senior-level Q&As with full answers + rapid-fire concept table |
| 14 | [14-learning-roadmap.md](./14-learning-roadmap.md) | 8-week study plan, key repos, kubectl cheat sheet, certifications, book links |

---

## Suggested Study Order

### Fast Track (4 weeks — if you already know Kubernetes basics)
```
Week 1: 01 → 02 → 03 → 04
Week 2: 05 → 06 → 07
Week 3: 09 → 10 → 11
Week 4: 12 → 13 (review all Q&As) → 14
```

### Full Track (8 weeks — from scratch to expert)
Follow the detailed 8-week plan in [14-learning-roadmap.md](./14-learning-roadmap.md).

---

## Quick Reference

### Key Concepts at a Glance

| Concept | One-liner |
|---|---|
| CRD | Registers a new resource type with the API server |
| Custom Resource | An instance of a CRD — like a Pod is an instance of Pod type |
| Controller | Reconcile loop that moves actual state → desired state |
| Operator | CRD + Controller that encodes Day 1 + Day 2 domain knowledge |
| Finalizer | Blocks deletion until your controller runs cleanup code |
| OwnerReference | Links child resources to parent — enables auto garbage collection |
| Informer | Watch-based local cache — controllers read from this, not API server |
| Level-triggered | Controllers react to current state, not change events — enables self-healing |
| SSA | Server-Side Apply — API server tracks field ownership per controller |
| CEL Rules | Cross-field validation in CRD schema — no webhook needed (k8s 1.25+) |
| Workload Identity | AKS mechanism — K8s SA OIDC token federated with Azure AD Managed Identity |
| ASO | Azure Service Operator — manage Azure resources (SQL, Storage, Redis…) via CRDs |
| KEDA | Scale deployments/jobs to zero based on Azure event sources (Service Bus, Storage Queue…) |
| OLM | Operator Lifecycle Manager — package manager for operators |

---

## AKS-Specific CRD Groups to Know

```
azure service operator (ASO) v2:
├── resources.azure.com          Resource Groups, Subscriptions
├── compute.azure.com            VMs, Disks, ManagedClusters
├── network.azure.com            VNets, Subnets, NSGs, Load Balancers
├── sql.azure.com                Azure SQL Server, Flexible Server
├── storage.azure.com            Storage Accounts, Blob Containers
├── cache.azure.com              Redis Cache
├── eventhub.azure.com           Event Hubs, Namespaces
├── servicebus.azure.com         Service Bus Queues, Topics
├── keyvault.azure.com           Key Vault, Keys, Secrets
└── documentdb.azure.com         CosmosDB (SQL API)

keda (keda.sh):
├── ScaledObject                 Scale Deployments based on event sources
├── ScaledJob                    Create Jobs based on event sources
├── TriggerAuthentication        Credentials for KEDA triggers (per namespace)
└── ClusterTriggerAuthentication Credentials for KEDA triggers (cluster-wide)

cert-manager (cert-manager.io):
├── Certificate                  Request a TLS certificate
├── ClusterIssuer                Cluster-wide cert issuer (Let's Encrypt, CA, self-signed)
└── Issuer                       Namespace-scoped cert issuer

gatekeeper / azure policy:
├── ConstraintTemplate           Defines a policy type with Rego logic
└── Constraint                   An instance of a policy (the actual rule)
```

---

*Version 1.0 | March 2026*
