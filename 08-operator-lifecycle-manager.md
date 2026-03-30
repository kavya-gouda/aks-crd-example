# 08 — Operator Lifecycle Manager (OLM)

> **Series**: CRD & Operators on AKS | Senior Engineer (12 YOE) Interview Prep

---

## What is OLM?

**Operator Lifecycle Manager (OLM)** is a framework for managing the lifecycle of operators themselves — installing, updating, and managing operators in a cluster. It is to operators what a package manager (apt, brew) is to software.

OLM is part of the Operator Framework project and is widely used in OpenShift. It can also be installed on AKS.

---

## OLM Core CRDs

```
OLM Custom Resources:
├── CatalogSource        — Points to a registry of operator bundles (OCI image)
├── Subscription         — "I want operator X, channel Y, version Z"
├── InstallPlan          — What OLM will install/upgrade (requires approval)
├── ClusterServiceVersion (CSV) — The operator package: CRDs + RBAC + Deployment
└── OperatorGroup        — Defines which namespaces the operator watches
```

### Install Flow

```
CatalogSource (registry)
        │
        ▼ OLM polls catalog
Subscription created by admin:
  "Install database-operator from stable channel"
        │
        ▼ OLM resolves dependencies
InstallPlan created:
  "Will install: database-operator.v1.0.0, cert-manager.v1.12.0"
        │
        ▼ Admin approves (or auto-approved)
OLM applies:
  - CRD yamls
  - RBAC
  - Operator Deployment (from CSV)
        │
        ▼
Operator is running ✅
```

---

## OLM Resource Examples

### CatalogSource

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: mycompany-operators
  namespace: olm
spec:
  sourceType: grpc
  image: mycompany/operator-catalog:latest   # OCI image with operator bundles
  displayName: MyCompany Operators
  publisher: MyCompany
  updateStrategy:
    registryPoll:
      interval: 30m                          # Poll for new operator versions
```

### Subscription

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: database-operator
  namespace: operators
spec:
  channel: stable                            # stable, alpha, beta channels
  name: database-operator
  source: mycompany-operators
  sourceNamespace: olm
  installPlanApproval: Automatic             # Automatic or Manual
  startingCSV: database-operator.v1.0.0     # Pin to specific version
  config:
    env:
      - name: LOG_LEVEL
        value: info
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
```

### OperatorGroup

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: operators-group
  namespace: operators
spec:
  targetNamespaces:
    - production                             # Operator watches only production namespace
    - staging
  # For cluster-wide operator:
  # targetNamespaces: []                    # Empty = all namespaces
```

---

## ClusterServiceVersion (CSV) — The Core OLM Resource

The CSV is the heart of an OLM bundle. It describes everything needed to run your operator.

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: database-operator.v1.0.0
  namespace: operators
  annotations:
    capabilities: "Full Lifecycle"           # Maturity level
    categories: "Database"
    description: "Manages PostgreSQL, MySQL, MongoDB on Kubernetes"
spec:
  displayName: Database Operator
  version: 1.0.0
  replaces: database-operator.v0.9.0        # Upgrade path from previous version
  maturity: stable

  # Owned CRDs — CRDs this operator installs and manages
  customresourcedefinitions:
    owned:
      - name: databases.mycompany.io
        version: v1
        kind: Database
        displayName: Database
        description: Manages a database instance
        resources:                           # Resources this CR creates
          - version: v1
            kind: StatefulSet
          - version: v1
            kind: Service
        specDescriptors:                     # UI hints for OpenShift console
          - path: replicas
            displayName: Replicas
            x-descriptors: ["urn:alm:descriptor:com.tectonic.ui:podCount"]
        statusDescriptors:
          - path: phase
            displayName: Phase
            x-descriptors: ["urn:alm:descriptor:io.kubernetes.phase"]

    # CRDs this operator requires from other operators
    required:
      - name: certificates.cert-manager.io
        version: v1
        kind: Certificate
        displayName: Certificate
        description: Required for webhook TLS

  # Install strategy — defines RBAC + Deployment
  install:
    strategy: deployment
    spec:
      clusterPermissions:
        - serviceAccountName: database-operator
          rules:
            - apiGroups: ["mycompany.io"]
              resources: ["databases", "databases/status", "databases/finalizers"]
              verbs: ["*"]
      deployments:
        - name: database-operator
          spec:
            replicas: 2
            selector:
              matchLabels:
                app: database-operator
            template:
              metadata:
                labels:
                  app: database-operator
              spec:
                serviceAccountName: database-operator
                containers:
                  - name: manager
                    image: mycompany/database-operator:v1.0.0
                    args: ["--leader-elect"]

  # Webhooks registered via OLM
  webhookdefinitions:
    - type: ValidatingAdmissionWebhook
      admissionReviewVersions: ["v1"]
      containerPort: 9443
      deploymentName: database-operator
      failurePolicy: Fail
      generateName: validate.databases.mycompany.io
      rules:
        - apiGroups: ["mycompany.io"]
          apiVersions: ["v1"]
          operations: ["CREATE", "UPDATE"]
          resources: ["databases"]
      sideEffects: None
      targetPort: 9443
      webhookPath: /validate-mycompany-io-v1-database
```

---

## Building an OLM Bundle

```bash
# 1. Generate OLM bundle using Operator SDK or Kubebuilder
make bundle IMG=mycompany/database-operator:v1.0.0

# Generated bundle/ directory:
bundle/
├── manifests/
│   ├── database-operator.clusterserviceversion.yaml
│   ├── databases.mycompany.io_databases.yaml
│   └── mycompany.io_v1_database_sample.yaml
├── metadata/
│   └── annotations.yaml                   # Bundle channel/package metadata
└── tests/
    └── scorecard/
        └── config.yaml                    # Scorecard test config

# 2. Build and push bundle image
make bundle-build bundle-push \
  BUNDLE_IMG=mycompany/database-operator-bundle:v1.0.0

# 3. Build catalog (index) image
opm index add \
  --bundles mycompany/database-operator-bundle:v1.0.0 \
  --tag mycompany/operator-catalog:latest

docker push mycompany/operator-catalog:latest

# 4. Run OLM scorecard tests (operator quality checks)
operator-sdk scorecard bundle/ --selector=suite=basic
```

---

## OLM vs Manual Deployment — When to Use Each

| | Manual (kubectl apply) | OLM |
|---|---|---|
| Simple deploys | ✅ | Overkill |
| Multi-operator platforms | ❌ Hard to manage | ✅ |
| Upgrade automation | Manual | ✅ Automatic |
| Dependency management | None | ✅ Resolves deps |
| OpenShift marketplace | No | ✅ Required |
| Air-gapped clusters | Simple | Needs catalog mirroring |

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
| **08** | OLM ← you are here |
| [09](./09-advanced-operator-patterns.md) | Advanced Patterns |
| [10](./10-aks-crd-operator-focus.md) | AKS Focus |
| [11](./11-hands-on-examples.md) | Hands-On Examples |
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
