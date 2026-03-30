# 07 — Operator Frameworks: Operator SDK & Kubebuilder

> **Series**: CRD & Operators on AKS | 

---

## Overview

| Framework | Language | Best For |
|---|---|---|
| **Kubebuilder** | Go | Full control, production-grade operators |
| **Operator SDK (Go)** | Go | Same as Kubebuilder + OLM integration |
| **Operator SDK (Helm)** | Helm | Wrapping existing Helm charts as operators |
| **Operator SDK (Ansible)** | Ansible | Teams with Ansible expertise |
| **KOPF** | Python | Quick prototypes, smaller teams |

---

## Kubebuilder — The Foundation

Kubebuilder is the official Kubernetes SIG project for building Go-based operators using `controller-runtime`.

### Project Setup

```bash
# Install kubebuilder
curl -L -o kubebuilder \
  "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder && sudo mv kubebuilder /usr/local/bin/

# Initialize project
kubebuilder init \
  --domain mycompany.io \
  --repo github.com/mycompany/database-operator

# Create API (CRD type + controller scaffold)
kubebuilder create api \
  --group databases \
  --version v1 \
  --kind Database \
  --resource \      # generate types file
  --controller      # generate controller file

# Create webhook scaffold
kubebuilder create webhook \
  --group databases \
  --version v1 \
  --kind Database \
  --defaulting \              # mutating webhook
  --programmatic-validation   # validating webhook
```

### Generated Project Structure

```
database-operator/
├── api/
│   └── v1/
│       ├── database_types.go          # ← Define your CRD Go structs here
│       ├── database_webhook.go        # ← Webhook logic
│       ├── database_webhook_test.go
│       └── groupversion_info.go       # API group + scheme registration
├── config/
│   ├── crd/
│   │   └── bases/
│   │       └── databases.mycompany.io_databases.yaml   # Auto-generated CRD YAML
│   ├── rbac/
│   │   ├── role.yaml                  # Auto-generated from // +kubebuilder:rbac markers
│   │   └── role_binding.yaml
│   ├── manager/
│   │   └── manager.yaml               # Operator Deployment
│   ├── webhook/
│   │   ├── manifests.yaml             # Webhook configurations
│   │   └── service.yaml
│   ├── certmanager/                   # cert-manager Certificate + Issuer
│   └── samples/
│       └── databases_v1_database.yaml # Example CR
├── controllers/
│   ├── database_controller.go         # ← Write your reconcile logic here
│   └── database_controller_test.go
├── main.go                            # Manager setup + controller registration
├── Makefile                           # Build, test, deploy commands
└── go.mod
```

### Defining CRD Types (Go Structs → OpenAPI Schema)

```go
// api/v1/database_types.go

package v1

import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

// DatabaseSpec defines the desired state
type DatabaseSpec struct {
    // Engine is the database engine type.
    // +kubebuilder:validation:Enum=postgres;mysql;mongodb
    Engine string `json:"engine"`

    // Replicas is the number of database replicas.
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    // +kubebuilder:default=1
    Replicas int32 `json:"replicas"`

    // Version is the database version in MAJOR.MINOR format.
    // +kubebuilder:validation:Pattern=`^[0-9]+\.[0-9]+$`
    Version string `json:"version"`

    // StorageSize is the persistent volume size.
    // +kubebuilder:default="10Gi"
    // +optional
    StorageSize string `json:"storageSize,omitempty"`

    // StorageClass is the Kubernetes StorageClass name.
    // On AKS use: managed-premium, managed-csi, azureblob-nfs-premium
    // +optional
    StorageClass string `json:"storageClass,omitempty"`

    // Backup configuration
    // +optional
    Backup *BackupSpec `json:"backup,omitempty"`
}

type BackupSpec struct {
    // +kubebuilder:default=false
    Enabled bool `json:"enabled"`

    // Schedule is a cron expression for backup schedule.
    // +kubebuilder:default="0 2 * * *"
    // +optional
    Schedule string `json:"schedule,omitempty"`

    StorageAccount string `json:"storageAccount,omitempty"`
}

// DatabaseStatus defines the observed state
type DatabaseStatus struct {
    // Phase is the current lifecycle phase.
    // +kubebuilder:validation:Enum=Pending;Running;Failed;Terminating
    Phase string `json:"phase,omitempty"`

    // ObservedGeneration is the .metadata.generation of the last reconciled spec.
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // ReadyReplicas is the number of ready replicas.
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`

    // Conditions represent the latest available observations.
    // +patchMergeKey=type
    // +patchStrategy=merge
    // +listType=map
    // +listMapKey=type
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas
// +kubebuilder:printcolumn:name="Engine",type=string,JSONPath=`.spec.engine`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
// +kubebuilder:resource:scope=Namespaced,shortName=db,categories=mycompany

type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DatabaseSpec   `json:"spec,omitempty"`
    Status DatabaseStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

type DatabaseList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Database `json:"items"`
}
```

### Generate CRD and RBAC from Markers

```bash
# Generate deepcopy methods, CRD YAML, RBAC YAML
make generate manifests

# Run locally against a cluster (uses your kubeconfig)
make run

# Build and push Docker image
make docker-build docker-push IMG=mycompany/database-operator:v1.0.0

# Deploy to cluster
make deploy IMG=mycompany/database-operator:v1.0.0

# Undeploy
make undeploy
```

---

## Operator SDK

Operator SDK wraps Kubebuilder for Go operators and adds:
- **OLM bundle generation** (`make bundle`)
- **Scorecard testing** (operator quality checks)
- **Helm and Ansible operator support**

### Go Operator (same as Kubebuilder)

```bash
operator-sdk init \
  --domain mycompany.io \
  --repo github.com/mycompany/db-operator

operator-sdk create api \
  --group db \
  --version v1 \
  --kind Database \
  --resource --controller
```

### Helm Operator — Wrap an Existing Helm Chart

```bash
operator-sdk init --plugins helm --domain mycompany.io

operator-sdk create api \
  --group db \
  --version v1 \
  --kind Database \
  --helm-chart=./charts/database   # Uses existing Helm chart
  # OR: --helm-chart=stable/postgresql --helm-chart-version=9.0.0
```

The Helm operator automatically:
- Maps `CR.spec` → Helm chart `values.yaml`
- Runs `helm upgrade --install` on reconcile
- Tracks Helm release status

### Ansible Operator

```bash
operator-sdk init --plugins ansible --domain mycompany.io

operator-sdk create api \
  --group db \
  --version v1 \
  --kind Database \
  --generate-role          # Creates an Ansible role scaffold
```

```yaml
# watches.yaml — maps CRD to Ansible role
- version: v1
  group: db.mycompany.io
  kind: Database
  role: database           # Ansible role name
  reconcilePeriod: "30s"
  manageStatus: true
```

```yaml
# roles/database/tasks/main.yml
- name: Create StatefulSet
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: apps/v1
      kind: StatefulSet
      metadata:
        name: "{{ ansible_operator_meta.name }}"
        namespace: "{{ ansible_operator_meta.namespace }}"
      spec:
        replicas: "{{ replicas | default(1) }}"
        # ...
```

---

## main.go — Manager Setup

```go
// main.go
func main() {
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        MetricsBindAddress:     ":8080",
        HealthProbeBindAddress: ":8081",
        LeaderElection:         true,
        LeaderElectionID:       "database-operator-lock",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }

    // Register controller
    if err = (&controllers.DatabaseReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "Database")
        os.Exit(1)
    }

    // Register webhook
    if err = (&dbv1.Database{}).SetupWebhookWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create webhook", "webhook", "Database")
        os.Exit(1)
    }

    // Health checks
    mgr.AddHealthzCheck("healthz", healthz.Ping)
    mgr.AddReadyzCheck("readyz", healthz.Ping)

    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
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
| **07** | Operator Frameworks ← you are here |
| [08](./08-operator-lifecycle-manager.md) | OLM |
| [09](./09-advanced-operator-patterns.md) | Advanced Patterns |
| [10](./10-aks-crd-operator-focus.md) | AKS Focus |
| [11](./11-hands-on-examples.md) | Hands-On Examples |
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
