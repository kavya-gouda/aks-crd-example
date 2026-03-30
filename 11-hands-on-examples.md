# 11 — Real-World Hands-On Examples

> **Series**: CRD & Operators on AKS | 

---

## Example 1: PostgreSQL Cluster Operator on AKS

**Scenario**: Full operator that manages a PostgreSQL HA cluster using Azure Premium Disk storage.

### Step 1 — The CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.db.mycompany.io
spec:
  group: db.mycompany.io
  names:
    kind: PostgresCluster
    plural: postgresclusters
    singular: postgrescluster
    shortNames: [pgcluster]
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [replicas, version, storage]
              x-kubernetes-validations:
                - rule: "self.replicas <= 5"
                  message: "Maximum 5 replicas supported"
                - rule: "self.replicas == 1 || self.replicas == 3 || self.replicas == 5"
                  message: "Replicas must be 1, 3, or 5 (odd numbers for quorum)"
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 5
                version:
                  type: string
                  enum: ["14", "15", "16"]
                storage:
                  type: object
                  required: [size]
                  properties:
                    size:
                      type: string
                    storageClassName:
                      type: string
                      default: "managed-premium"       # Azure Premium SSD
                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    storageAccount:
                      type: string
                    containerName:
                      type: string
                      default: "postgres-backups"
                    schedule:
                      type: string
                      default: "0 2 * * *"
                resources:
                  type: object
                  default:
                    requests:
                      cpu: "500m"
                      memory: "1Gi"
                    limits:
                      cpu: "2"
                      memory: "4Gi"
            status:
              type: object
              properties:
                phase:
                  type: string
                primaryEndpoint:
                  type: string
                readEndpoint:
                  type: string
                readyReplicas:
                  type: integer
                observedGeneration:
                  type: integer
                  format: int64
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type: {type: string}
                      status: {type: string}
                      reason: {type: string}
                      message: {type: string}
                      lastTransitionTime: {type: string, format: date-time}
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
      additionalPrinterColumns:
        - name: Version
          jsonPath: .spec.version
          type: string
        - name: Replicas
          jsonPath: .spec.replicas
          type: integer
        - name: Phase
          jsonPath: .status.phase
          type: string
        - name: Primary
          jsonPath: .status.primaryEndpoint
          type: string
          priority: 1              # Only shown with -o wide
        - name: Age
          jsonPath: .metadata.creationTimestamp
          type: date
```

### Step 2 — Sample CR

```yaml
apiVersion: db.mycompany.io/v1
kind: PostgresCluster
metadata:
  name: orders-db
  namespace: production
  labels:
    app: orders
    environment: production
    team: backend
spec:
  replicas: 3
  version: "15"
  storage:
    size: "100Gi"
    storageClassName: managed-premium
  backup:
    enabled: true
    storageAccount: mybackupstorage
    containerName: postgres-backups
    schedule: "0 1 * * *"
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "4"
      memory: "8Gi"
```

### Step 3 — Controller Reconcile Logic (key parts)

```go
func (r *PostgresClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    pg := &dbv1.PostgresCluster{}

    if err := r.Get(ctx, req.NamespacedName, pg); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    if !pg.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, pg)
    }

    if !controllerutil.ContainsFinalizer(pg, pgFinalizer) {
        controllerutil.AddFinalizer(pg, pgFinalizer)
        return ctrl.Result{}, r.Update(ctx, pg)
    }

    // Reconcile all child resources in order
    steps := []func(context.Context, *dbv1.PostgresCluster) error{
        r.reconcileConfigMap,
        r.reconcileSecret,
        r.reconcileStatefulSet,
        r.reconcileHeadlessService,
        r.reconcileReadService,
        r.reconcileBackupCronJob,    // Only if backup enabled
        r.reconcilePodDisruptionBudget,
    }

    for _, step := range steps {
        if err := step(ctx, pg); err != nil {
            log.Error(err, "reconcile step failed")
            setCondition(pg, "Ready", "False", "ReconcileFailed", err.Error())
            _ = r.Status().Update(ctx, pg)
            return ctrl.Result{}, err
        }
    }

    // Update status
    if err := r.updateStatus(ctx, pg); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *PostgresClusterReconciler) reconcileStatefulSet(ctx context.Context, pg *dbv1.PostgresCluster) error {
    desired := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      pg.Name,
            Namespace: pg.Namespace,
        },
        Spec: appsv1.StatefulSetSpec{
            ServiceName: pg.Name + "-headless",
            Replicas:    &pg.Spec.Replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{"app": pg.Name},
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app":                          pg.Name,
                        "azure.workload.identity/use":  "true",
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  "postgres",
                        Image: fmt.Sprintf("postgres:%s", pg.Spec.Version),
                        Env: []corev1.EnvVar{
                            {Name: "POSTGRES_PASSWORD", ValueFrom: &corev1.EnvVarSource{
                                SecretKeyRef: &corev1.SecretKeySelector{
                                    LocalObjectReference: corev1.LocalObjectReference{Name: pg.Name + "-secret"},
                                    Key: "password",
                                },
                            }},
                        },
                        Resources: pg.Spec.Resources,
                        VolumeMounts: []corev1.VolumeMount{{
                            Name:      "data",
                            MountPath: "/var/lib/postgresql/data",
                        }},
                    }},
                },
            },
            VolumeClaimTemplates: []corev1.PersistentVolumeClaim{{
                ObjectMeta: metav1.ObjectMeta{Name: "data"},
                Spec: corev1.PersistentVolumeClaimSpec{
                    AccessModes: []corev1.PersistentVolumeAccessMode{corev1.ReadWriteOnce},
                    StorageClassName: &pg.Spec.Storage.StorageClassName,
                    Resources: corev1.ResourceRequirements{
                        Requests: corev1.ResourceList{
                            corev1.ResourceStorage: resource.MustParse(pg.Spec.Storage.Size),
                        },
                    },
                },
            }},
        },
    }

    ctrl.SetControllerReference(pg, desired, r.Scheme)
    return r.createOrUpdate(ctx, desired)
}
```

---

## Example 2: Azure Backup Integration via ASO CRDs

```go
// Operator creates ASO BackupVault + BackupPolicy + BackupInstance CRs
func (r *PostgresClusterReconciler) reconcileAzureBackup(ctx context.Context, pg *dbv1.PostgresCluster) error {
    if !pg.Spec.Backup.Enabled {
        return nil
    }

    // Create a CronJob that triggers Azure Blob backup on schedule
    cronJob := &batchv1.CronJob{
        ObjectMeta: metav1.ObjectMeta{
            Name:      pg.Name + "-backup",
            Namespace: pg.Namespace,
        },
        Spec: batchv1.CronJobSpec{
            Schedule:          pg.Spec.Backup.Schedule,
            ConcurrencyPolicy: batchv1.ForbidConcurrent,
            JobTemplate: batchv1.JobTemplateSpec{
                Spec: batchv1.JobSpec{
                    Template: corev1.PodTemplateSpec{
                        Metadata: metav1.ObjectMeta{
                            Labels: map[string]string{
                                "azure.workload.identity/use": "true",
                            },
                        },
                        Spec: corev1.PodSpec{
                            ServiceAccountName: "postgres-backup-sa",  // Has storage blob write permissions
                            RestartPolicy:      corev1.RestartPolicyOnFailure,
                            Containers: []corev1.Container{{
                                Name:  "pg-backup",
                                Image: "mycompany/pg-backup:latest",
                                Env: []corev1.EnvVar{
                                    {Name: "PG_HOST", Value: pg.Status.PrimaryEndpoint},
                                    {Name: "STORAGE_ACCOUNT", Value: pg.Spec.Backup.StorageAccount},
                                    {Name: "CONTAINER_NAME", Value: pg.Spec.Backup.ContainerName},
                                    {Name: "BACKUP_NAME", Value: fmt.Sprintf("%s-$(date +%%Y%%m%%d%%H%%M%%S)", pg.Name)},
                                },
                            }},
                        },
                    },
                },
            },
        },
    }

    ctrl.SetControllerReference(pg, cronJob, r.Scheme)
    return r.createOrUpdate(ctx, cronJob)
}
```

---

## Example 3: Operator Testing with Envtest

Envtest spins up a real API server and etcd locally — no cluster needed.

```go
package controllers_test

import (
    "context"
    "path/filepath"
    "testing"
    "time"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    appsv1 "k8s.io/api/apps/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    "sigs.k8s.io/controller-runtime/pkg/envtest"

    dbv1 "github.com/mycompany/database-operator/api/v1"
)

var (
    k8sClient client.Client
    testEnv   *envtest.Environment
)

func TestControllers(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    testEnv = &envtest.Environment{
        CRDDirectoryPaths: []string{
            filepath.Join("..", "config", "crd", "bases"),
        },
        ErrorIfCRDPathMissing: true,
    }

    cfg, err := testEnv.Start()
    Expect(err).NotTo(HaveOccurred())

    scheme := runtime.NewScheme()
    _ = dbv1.AddToScheme(scheme)
    _ = appsv1.AddToScheme(scheme)

    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme})
    Expect(err).NotTo(HaveOccurred())

    // Start controller manager
    mgr, _ := ctrl.NewManager(cfg, ctrl.Options{Scheme: scheme})
    (&DatabaseReconciler{Client: mgr.GetClient(), Scheme: mgr.GetScheme()}).
        SetupWithManager(mgr)

    go func() { mgr.Start(ctrl.SetupSignalHandler()) }()
})

var _ = AfterSuite(func() { testEnv.Stop() })

var _ = Describe("PostgresCluster Controller", func() {
    const (
        timeout  = 10 * time.Second
        interval = 250 * time.Millisecond
    )

    Context("Creating a PostgresCluster", func() {
        It("should create a StatefulSet with correct replicas", func() {
            ctx := context.Background()

            pg := &dbv1.PostgresCluster{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-pg",
                    Namespace: "default",
                },
                Spec: dbv1.PostgresClusterSpec{
                    Replicas: 3,
                    Version:  "15",
                    Storage:  dbv1.StorageSpec{Size: "10Gi"},
                },
            }
            Expect(k8sClient.Create(ctx, pg)).Should(Succeed())

            // StatefulSet should be created
            sts := &appsv1.StatefulSet{}
            Eventually(func() error {
                return k8sClient.Get(ctx, types.NamespacedName{
                    Name: "test-pg", Namespace: "default",
                }, sts)
            }, timeout, interval).Should(Succeed())

            Expect(*sts.Spec.Replicas).To(Equal(int32(3)))

            // Status should be updated
            Eventually(func() string {
                k8sClient.Get(ctx, types.NamespacedName{
                    Name: "test-pg", Namespace: "default",
                }, pg)
                return pg.Status.Phase
            }, timeout, interval).Should(Equal("Running"))
        })

        It("should reject invalid replica count", func() {
            ctx := context.Background()
            pg := &dbv1.PostgresCluster{
                ObjectMeta: metav1.ObjectMeta{
                    Name: "invalid-pg", Namespace: "default",
                },
                Spec: dbv1.PostgresClusterSpec{
                    Replicas: 2,    // Even number — violates CEL rule
                    Version:  "15",
                    Storage:  dbv1.StorageSpec{Size: "10Gi"},
                },
            }
            err := k8sClient.Create(ctx, pg)
            Expect(err).To(HaveOccurred())
            Expect(err.Error()).To(ContainSubstring("Replicas must be 1, 3, or 5"))
        })
    })
})
```

---

## Example 4: Multi-Region App Operator

```yaml
# CRD for deploying an app across multiple AKS clusters/regions
apiVersion: mycompany.io/v1
kind: MultiRegionApp
metadata:
  name: payments-service
  namespace: platform
spec:
  image: mycompany/payments:v2.3.1
  regions:
    - name: eastus
      aksCluster: aks-eastus-prod
      resourceGroup: rg-eastus-prod
      replicas: 5
      weight: 60                       # 60% traffic via Azure Front Door
    - name: westeurope
      aksCluster: aks-westeurope-prod
      resourceGroup: rg-westeurope-prod
      replicas: 3
      weight: 40
  globalLoadBalancer:
    type: azure-front-door
    healthCheckPath: /health
    healthCheckInterval: 30
  rollout:
    strategy: bluegreen                # bluegreen | canary | rolling
    canaryWeight: 10                   # Start with 10% traffic to new version
status:
  globalEndpoint: payments.mycompany.azurefd.net
  regions:
    - name: eastus
      phase: Running
      readyReplicas: 5
      endpoint: payments-eastus.mycompany.com
    - name: westeurope
      phase: Running
      readyReplicas: 3
      endpoint: payments-westeurope.mycompany.com
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
| **11** | Hands-On Examples ← you are here |
| [12](./12-production-best-practices.md) | Production Best Practices |
| [13](./13-interview-qa.md) | Interview Q&A |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
