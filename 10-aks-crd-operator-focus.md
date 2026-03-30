# 10 — Azure Kubernetes Service (AKS): CRD & Operator Focus

> **Series**: CRD & Operators on AKS | 

---

## AKS Architecture Relevant to Operators

```
Azure Control Plane
├── Azure Resource Manager (ARM)
├── AKS Resource Provider  → manages k8s control plane lifecycle
└── Azure AD / Entra ID    → identity for workloads and admins

AKS Cluster
├── Control Plane (fully managed by Azure — no direct access)
│   ├── kube-apiserver
│   ├── etcd
│   ├── controller-manager
│   └── scheduler
├── Node Pools (Azure VMSS-backed)
│   ├── System Node Pool   → system components (CoreDNS, kube-proxy, operators)
│   └── User Node Pools    → your workloads
└── Azure Integrations via CRDs/Operators
    ├── Azure Service Operator (ASO) v2
    ├── KEDA (Event-Driven Autoscaling)
    ├── cert-manager
    ├── Workload Identity (replaces AAD Pod Identity)
    ├── Azure CNI / Cilium CNI
    └── Azure Policy / OPA Gatekeeper
```

---

## Azure Service Operator (ASO) v2

ASO lets you manage **Azure resources** directly from Kubernetes using CRDs — GitOps-friendly, no ARM templates needed.

### Installing ASO v2 on AKS

```bash
# Install via Helm
helm repo add aso2 https://raw.githubusercontent.com/Azure/azure-service-operator/main/v2/charts
helm repo update

helm install aso2 aso2/azure-service-operator \
  --create-namespace \
  --namespace azureserviceoperator-system \
  --set azureSubscriptionID=$AZURE_SUBSCRIPTION_ID \
  --set azureTenantID=$AZURE_TENANT_ID \
  --set azureClientID=$MANAGED_IDENTITY_CLIENT_ID \  # Workload Identity
  --set useWorkloadIdentityAuth=true
```

### ASO CRD Examples

```yaml
# Resource Group
apiVersion: resources.azure.com/v1api20200601
kind: ResourceGroup
metadata:
  name: my-rg
  namespace: default
spec:
  location: eastus
  owner:
    name: my-subscription

---
# Azure SQL Server
apiVersion: sql.azure.com/v1api20211101
kind: Server
metadata:
  name: my-sql-server
  namespace: default
spec:
  location: eastus
  owner:
    name: my-rg
  administratorLogin: sqladmin
  administratorLoginPassword:
    name: sql-admin-secret
    key: password
  version: "12.0"
  minimalTlsVersion: "1.2"

---
# Azure Redis Cache
apiVersion: cache.azure.com/v1api20201201
kind: Redis
metadata:
  name: my-redis
  namespace: default
spec:
  location: eastus
  owner:
    name: my-rg
  sku:
    name: Standard
    family: C
    capacity: 1
  enableNonSslPort: false
  minimumTlsVersion: "1.2"

---
# Azure Storage Account
apiVersion: storage.azure.com/v1api20210401
kind: StorageAccount
metadata:
  name: mystorageaccount
  namespace: default
spec:
  location: eastus
  owner:
    name: my-rg
  kind: StorageV2
  sku:
    name: Standard_LRS
  accessTier: Hot

---
# Azure Event Hub Namespace
apiVersion: eventhub.azure.com/v1api20211101
kind: Namespace
metadata:
  name: my-eventhub-ns
  namespace: default
spec:
  location: eastus
  owner:
    name: my-rg
  sku:
    name: Standard
    tier: Standard
    capacity: 2
```

### Key ASO CRD API Groups

| API Group | Resources |
|---|---|
| `resources.azure.com` | ResourceGroup, Subscription |
| `compute.azure.com` | VirtualMachine, Disk, ManagedCluster |
| `network.azure.com` | VirtualNetwork, Subnet, NetworkSecurityGroup, LoadBalancer |
| `sql.azure.com` | Server, Database, FlexibleServer |
| `storage.azure.com` | StorageAccount, BlobContainer, Queue |
| `cache.azure.com` | Redis |
| `eventhub.azure.com` | Namespace, EventHub, ConsumerGroup |
| `servicebus.azure.com` | Namespace, Queue, Topic, Subscription |
| `containerservice.azure.com` | ManagedCluster (AKS itself!) |
| `dbformongodb.azure.com` | MongodbDatabase, Collection |
| `documentdb.azure.com` | DatabaseAccount (CosmosDB SQL) |
| `keyvault.azure.com` | Vault, Key, Secret |

---

## Workload Identity — Secure Azure Auth for Operators

### Why Workload Identity (not AAD Pod Identity)

| | AAD Pod Identity (deprecated) | Workload Identity (recommended) |
|---|---|---|
| Mechanism | NMI daemon intercepts IMDS | OIDC federation with Azure AD |
| Credentials | Long-lived token via NMI | Short-lived federated tokens |
| Security | Requires privileged DaemonSet | No privileged components |
| Maintenance | Custom CRDs to manage | Standard k8s SA annotations |

### Setup Workload Identity for an Operator

```bash
# 1. Enable OIDC issuer + Workload Identity on AKS cluster
az aks update \
  -g myRG -n myAKS \
  --enable-oidc-issuer \
  --enable-workload-identity

# 2. Get OIDC issuer URL
OIDC_ISSUER=$(az aks show \
  -g myRG -n myAKS \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# 3. Create a User-Assigned Managed Identity
az identity create \
  --name database-operator-identity \
  --resource-group myRG

MANAGED_IDENTITY_CLIENT_ID=$(az identity show \
  --name database-operator-identity \
  --resource-group myRG \
  --query clientId -o tsv)

# 4. Assign Azure RBAC roles (least privilege)
az role assignment create \
  --assignee $MANAGED_IDENTITY_CLIENT_ID \
  --role "Contributor" \                         # Scope to specific RG, not subscription
  --scope /subscriptions/$SUB_ID/resourceGroups/target-rg

# 5. Create federated credential (links K8s SA → Azure MI)
az identity federated-credential create \
  --name database-operator-federated-cred \
  --identity-name database-operator-identity \
  --resource-group myRG \
  --issuer $OIDC_ISSUER \
  --subject system:serviceaccount:operators:database-operator \
  --audience api://AzureADTokenExchange

# 6. Annotate the Kubernetes Service Account
kubectl annotate serviceaccount database-operator \
  -n operators \
  azure.workload.identity/client-id=$MANAGED_IDENTITY_CLIENT_ID \
  azure.workload.identity/tenant-id=$AZURE_TENANT_ID
```

### Kubernetes Resources for Workload Identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database-operator
  namespace: operators
  annotations:
    azure.workload.identity/client-id: "<managed-identity-client-id>"
    azure.workload.identity/tenant-id: "<tenant-id>"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-operator
  namespace: operators
spec:
  template:
    metadata:
      labels:
        app: database-operator
        azure.workload.identity/use: "true"    # ← Required label for token injection
    spec:
      serviceAccountName: database-operator
      containers:
        - name: manager
          image: mycompany/database-operator:v1.0.0
          # Azure SDK auto-discovers credentials via:
          # AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_FEDERATED_TOKEN_FILE
          # These are auto-injected by the Workload Identity mutating webhook
```

---

## KEDA — Kubernetes Event-Driven Autoscaling

KEDA is deeply integrated with AKS and enables **scale-to-zero** based on Azure event sources.

### ScaledObject — Scale Deployments

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor
  pollingInterval: 15             # Check every 15 seconds
  cooldownPeriod: 300             # Wait 300s before scaling down
  minReplicaCount: 0              # Scale to ZERO when idle
  maxReplicaCount: 50
  advanced:
    restoreToOriginalReplicaCount: true
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: orders
        namespace: myservicebus.servicebus.windows.net
        messageCount: "5"         # 1 replica per 5 messages
      authenticationRef:
        name: servicebus-workload-identity-auth

---
# ScaledJob — for batch processing (creates Jobs, not scales Deployment)
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: batch-processor
  namespace: production
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    template:
      spec:
        containers:
          - name: processor
            image: mycompany/batch-processor:latest
        restartPolicy: OnFailure
  pollingInterval: 30
  maxReplicaCount: 20
  triggers:
    - type: azure-storage-queue
      metadata:
        queueName: batch-jobs
        accountName: mystorageaccount
        queueLength: "1"          # 1 job per queue message
      authenticationRef:
        name: storage-workload-identity-auth

---
# TriggerAuthentication — with Workload Identity
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: servicebus-workload-identity-auth
  namespace: production
spec:
  podIdentity:
    provider: azure-workload      # Use AKS Workload Identity

---
# ClusterTriggerAuthentication — cluster-wide (reusable across namespaces)
apiVersion: keda.sh/v1alpha1
kind: ClusterTriggerAuthentication
metadata:
  name: global-servicebus-auth
spec:
  podIdentity:
    provider: azure-workload
```

### KEDA Scalers for Azure

| Scaler | Trigger | Scale Based On |
|---|---|---|
| `azure-servicebus` | Service Bus Queue/Topic | Message count |
| `azure-storage-queue` | Azure Storage Queue | Queue length |
| `azure-eventhub` | Event Hub | Consumer lag |
| `azure-blob` | Blob Storage | Blob count |
| `azure-monitor` | Azure Monitor Metrics | Any metric |
| `azure-pipelines` | Azure DevOps Pipelines | Pending jobs |
| `prometheus` | Prometheus | Custom app metric |
| `kubernetes-workload` | Pod count | Other deployment replicas |

---

## cert-manager on AKS

cert-manager is essential for issuing TLS certificates for operator webhooks.

```yaml
# ClusterIssuer with Azure DNS challenge (for private domains)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@mycompany.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - dns01:
          azureDNS:
            subscriptionID: <subscription-id>
            resourceGroupName: dns-rg
            hostedZoneName: mycompany.com
            environment: AzurePublicCloud
            managedIdentity:
              clientID: <cert-manager-managed-identity-client-id>

---
# Self-signed ClusterIssuer (for internal/webhook certs)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}

---
# CA ClusterIssuer (use internal CA)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-ca-key-pair      # Contains tls.crt + tls.key

---
# Certificate for operator webhook
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: database-operator-webhook-cert
  namespace: operators
spec:
  secretName: database-operator-webhook-tls
  duration: 8760h                          # 1 year
  renewBefore: 720h                        # Renew 30 days before expiry
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
    - database-operator-webhook-service.operators.svc
    - database-operator-webhook-service.operators.svc.cluster.local
```

---

## Azure Policy with OPA Gatekeeper

AKS integrates with Azure Policy using **Gatekeeper** (OPA-based admission controller):

```yaml
# ConstraintTemplate — defines a reusable policy with Rego logic
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requireoperatorlabels
spec:
  crd:
    spec:
      names:
        kind: RequireOperatorLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requireoperatorlabels
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }

---
# Constraint — instance of the ConstraintTemplate (the actual policy rule)
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireOperatorLabels
metadata:
  name: require-labels-on-databases
spec:
  enforcementAction: deny                    # deny | warn | dryrun
  match:
    kinds:
      - apiGroups: ["mycompany.io"]
        kinds: ["Database"]
    namespaces: ["production", "staging"]
  parameters:
    labels: ["app", "environment", "team", "cost-center"]
```

---

## AKS Cluster Upgrade & Operator Compatibility

```bash
# Before any AKS upgrade — check deprecated APIs
kubectl api-resources --api-group=apiextensions.k8s.io

# Run Pluto to scan entire cluster for deprecated API usage
pluto detect-all-in-cluster --target-versions k8s=v1.30.0

# Check which CRDs use v1beta1 (removed in k8s 1.22)
kubectl get crds -o json | \
  jq '.items[] | select(.apiVersion | contains("v1beta1")) | .metadata.name'

# Check deprecated admission webhook configs
kubectl get validatingwebhookconfigurations -o json | \
  jq '.items[] | select(.apiVersion | contains("v1beta1")) | .metadata.name'

# Verify operator compatibility with target k8s version
kubectl get deployment database-operator -n operators -o json | \
  jq '.spec.template.spec.containers[].image'
# Then check operator release notes for supported k8s versions
```

### Key API Removals by K8s Version

| Removed In | Deprecated API | Use Instead |
|---|---|---|
| 1.22 | `apiextensions.k8s.io/v1beta1` CRD | `apiextensions.k8s.io/v1` |
| 1.22 | `admissionregistration.k8s.io/v1beta1` | `admissionregistration.k8s.io/v1` |
| 1.22 | `rbac.authorization.k8s.io/v1beta1` | `rbac.authorization.k8s.io/v1` |
| 1.25 | `batch/v1beta1` CronJob | `batch/v1` |
| 1.26 | `autoscaling/v2beta2` HPA | `autoscaling/v2` |

---

## AKS Node Pool Placement for Operators

```yaml
spec:
  # Run operators on system node pool (reserved for system workloads)
  tolerations:
    - key: CriticalAddonsOnly
      operator: Exists

  nodeSelector:
    kubernetes.azure.com/mode: system      # AKS system node pool label

  # Spread operator replicas across nodes/zones
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: database-operator
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone   # Spread across AZs
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: database-operator

  # Anti-affinity — don't co-locate operator replicas on same node
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: database-operator
          topologyKey: kubernetes.io/hostname
```

---

## Network Policy for Operators on AKS (Azure CNI / Cilium)

```yaml
# Allow operator pod to reach API server and DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-operator-egress
  namespace: operators
spec:
  podSelector:
    matchLabels:
      app: database-operator
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - ports:
        - protocol: TCP
          port: 9443              # Webhook server — from API server
  egress:
    - ports:
        - protocol: TCP
          port: 443               # API server (HTTPS)
    - to:
        - namespaceSelector: {}   # DNS resolution (all namespaces)
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
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
| **10** | AKS Focus ← you are here |
| [11](./11-hands-on-examples.md) | Hands-On Examples |
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
