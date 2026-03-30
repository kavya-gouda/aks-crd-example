# 13 — Senior-Level Interview Q&A (12 YOE)

> **Series**: CRD & Operators on AKS | Senior Engineer (12 YOE) Interview Prep

---

## How to Use This File

Each question is marked by category. Product-based companies typically ask a mix of:
- **Architecture & Design** — open-ended system design
- **Deep Concepts** — internals, trade-offs
- **AKS / Azure** — platform-specific knowledge
- **Debugging** — how you troubleshoot production issues
- **Coding / Schema Design** — live design or whiteboard

Read the expected answer, then practice explaining it out loud in 2–3 minutes without reading.

---

## Architecture & Design Questions

---

**Q1: How would you design an operator for a stateful application like Kafka at scale?**

> **Expected Answer:**
>
> **CRD Design:**
> - `KafkaCluster` — main resource (broker count, ZooKeeper/KRaft mode, storage)
> - `KafkaTopic` — self-service topic provisioning (separate CRD for clean ownership)
> - `KafkaUser` — SASL/ACL user management
>
> **Controller Design:**
> - Use `StatefulSet` for brokers — stable network identities (`broker-0.kafka-headless`)
> - Use `PodDisruptionBudget`: `minAvailable: ceil(replicas/2)+1` to maintain quorum
> - Operator must understand quorum — never roll more than 1 broker at a time
> - Status tracks ISR (in-sync replicas) per partition, not just pod readiness
>
> **Day 2 Operations:**
> - Rolling upgrade: cordon broker → wait for partition re-assignment → proceed
> - Topic rebalancing on scale-out (trigger preferred replica leader election)
> - Backup: snapshot Kafka MirrorMaker or periodic topic export to Azure Blob
>
> **On AKS:**
> - Use `managed-premium` storage class (Azure Premium SSD) — low latency for log segments
> - Enable zone-redundant storage (ZRS) for storage class
> - Spread brokers across AZs using `topologySpreadConstraints`
> - Use Azure CNI for direct pod networking (no NAT, important for Kafka clients)

---

**Q2: How do you handle CRD schema migrations in a live cluster without downtime?**

> **Expected Answer:**
>
> **Safe changes (no webhook needed):**
> - Adding optional fields with defaults → safe, just add to schema
> - Expanding enum values → safe
> - Relaxing validation rules → safe
>
> **Breaking changes (need conversion webhook):**
> 1. Introduce new version (e.g., `v2`) alongside `v1` in the CRD
> 2. Deploy conversion webhook **before** adding `v2` to the CRD
> 3. Use hub-spoke model: `v1` is the hub, all conversions go through it
> 4. Update CRD with both versions served
> 5. Migrate clients to use `v2`
> 6. Run storage migration job (read all `v1` objects, apply as `v2`)
> 7. Set `v2` as storage version, `v1` served: false
>
> **Key risks to mention:**
> - A broken conversion webhook blocks ALL access to that resource — deploy with redundancy, test extensively
> - Never tighten validation on existing fields — objects already in etcd would fail re-admission
> - Use `kubectl apply --dry-run=server` to test before real apply

---

**Q3: What happens if your operator's reconcile function runs longer than the leader lease duration?**

> **Expected Answer:**
>
> **What happens:**
> - Default lease duration: 15s, renew deadline: 10s
> - If the leader doesn't renew within 15s, another replica acquires the lease
> - Two instances now consider themselves leader → both reconcile the same resources simultaneously
> - Race conditions: both try to create/update the same child resources → conflicts, duplicate resources
>
> **How to prevent it:**
> - Use `ctx.Done()` channel — all API calls propagate context, so they cancel when leadership is lost
> - Break long reconcile into smaller steps with explicit `RequeueAfter` intervals
> - Increase `LeaseDuration` / `RenewDeadline` if reconcile legitimately takes longer
> - Use `MaxConcurrentReconciles` thoughtfully — high parallelism can slow individual reconciles
>
> **Detection:**
> - Watch for `"leader election lost"` in operator logs
> - Reconcile error rate spikes → sign of two competing leaders

---

**Q4: How do you prevent a buggy operator from cascading failures across all resources?**

> **Expected Answer:**
>
> - **Rate limiting**: Built-in exponential backoff in controller-runtime — errors don't immediately retry
> - **MaxConcurrentReconciles**: Caps how many reconciles run in parallel — limits blast radius
> - **Circuit breaker**: Custom logic to stop reconciling a resource after N consecutive failures
> - **Pause annotation**: `mycompany.io/reconcile-paused: "true"` — emergency stop per resource
> - **Namespace-scoped operator**: Operator watches only specific namespaces — failed operator in NS A doesn't affect NS B
> - **Resource quotas**: Prevent operator from creating unlimited Pods/PVCs
> - **Admission webhooks**: Validate CRs before creation — stops garbage input early
> - **Canary deployment**: New operator version watches a test namespace first before going cluster-wide
> - **Feature flags via ConfigMap**: Operator reads a ConfigMap at startup or per-reconcile to toggle features off

---

**Q5: How would you implement multi-tenancy in a CRD-based platform?**

> **Expected Answer:**
>
> **Option A — Namespace isolation (strongest):**
> - One operator instance per tenant namespace
> - RBAC: tenants only have CRUD on resources in their own namespace
> - `ResourceQuota`: limits CRs, Pods, PVCs per namespace
> - Network policies: namespace-level isolation
>
> **Option B — Shared operator with label-based tenancy:**
> - Single operator watches all namespaces
> - Tenants opt-in via label `mycompany.io/tenant: team-alpha`
> - Operator uses label selectors in watches
> - Admission webhook enforces tenant labels and validates tenant context
>
> **Cross-cutting concerns:**
> - Status visibility: tenants should NOT see other tenants' status conditions
> - Audit logging: who created/modified what
> - Cost allocation: label-based chargeback
>
> **On AKS:**
> - Azure AD groups → K8s ClusterRoleBinding per team
> - Azure Policy (Gatekeeper) enforces labeling standards across all namespaces

---

**Q6: Explain the difference between a Finalizer and an OwnerReference for resource cleanup.**

> **Expected Answer:**
>
> | | OwnerReference | Finalizer |
> |---|---|---|
> | Mechanism | Declarative GC — K8s auto-deletes children | Imperative hook — controller runs code |
> | Custom logic | None | Full control (API calls, external cleanup) |
> | Scope | In-cluster only | In-cluster + external resources |
> | Who removes | Kubernetes garbage collector | Your controller |
>
> **OwnerReference** — use when:
> - Child resources are K8s objects (Pods, Services, ConfigMaps)
> - No cleanup logic needed — just delete when parent is gone
> - Set with `ctrl.SetControllerReference(parent, child, scheme)`
>
> **Finalizer** — use when:
> - External resources need cleanup (cloud DB, DNS record, license release)
> - Ordering matters (delete replica before primary)
> - Need to send notifications before deletion
> - Need to archive data before deletion
>
> **Both together** (most real operators):
> - OwnerReferences for all in-cluster children (auto GC)
> - Finalizer for external resource cleanup

---

**Q7: How does the Informer cache affect your controller's behavior?**

> **Expected Answer:**
>
> **What the cache is:**
> - Controllers read from a local in-memory cache, NOT directly from the API server
> - Cache is populated via Watch (long-poll HTTP/2 stream) — eventually consistent
> - After a Create/Update, the cache may lag by milliseconds to seconds
>
> **Why it matters:**
> - Reading from cache is fast (no API server round-trip) — critical for high-throughput reconcilers
> - But it introduces eventual consistency — never assume the cache is perfectly current
>
> **Practical implications:**
> - `cache.WaitForCacheSync()` at startup — don't reconcile until cache is populated
> - After updating a resource, the triggered reconcile will re-read from cache — it will be fresh by then
> - If you need guaranteed freshness: use `client.Reader` with `Uncached` option (direct API call)
>
> **Common bug pattern to mention:**
> ```go
> r.Update(ctx, db)        // Update written to API server
> r.Get(ctx, key, db)      // Reading from CACHE — may still return old version!
> if db.Status.Phase == "Running" { ... }  // Bug: stale data
> ```
> Fix: trust that the next watch event will deliver the fresh version.

---

## AKS-Specific Questions

---

**Q8: How do you securely give an operator access to Azure resources in AKS?**

> **Expected Answer:**
>
> **Use Workload Identity (not AAD Pod Identity — deprecated)**
>
> **Step-by-step:**
> 1. Enable OIDC issuer on AKS: `az aks update --enable-oidc-issuer --enable-workload-identity`
> 2. Create User-Assigned Managed Identity in Azure
> 3. Assign least-privilege Azure RBAC roles to the identity (e.g., `Storage Blob Data Contributor` scoped to a specific storage account — not subscription-wide)
> 4. Create Federated Credential: links K8s ServiceAccount OIDC token → Azure AD identity
> 5. Annotate ServiceAccount with `azure.workload.identity/client-id`
> 6. Add label `azure.workload.identity/use: "true"` to operator pods
>
> **What happens at runtime:**
> - Workload Identity webhook (mutating) injects env vars + token volume into the pod
> - Azure SDK reads `AZURE_FEDERATED_TOKEN_FILE` + `AZURE_CLIENT_ID` automatically
> - Tokens are short-lived (~1hr) and auto-refreshed — no secret rotation needed
>
> **What to avoid:**
> - Service principal client secrets in Kubernetes Secrets (rotation burden, secret sprawl)
> - Cluster-wide contributor roles — always scope to minimum needed resource group

---

**Q9: You're upgrading AKS from 1.27 to 1.30. Walk me through your operator-related checks.**

> **Expected Answer:**
>
> **Pre-upgrade checks:**
> 1. Run `pluto detect-all-in-cluster --target-versions k8s=v1.30.0` — finds deprecated API usage
> 2. Verify all CRDs: `kubectl get crds -o json | jq '.items[].apiVersion'` — all should be `apiextensions.k8s.io/v1`
> 3. Verify webhook configs use `admissionregistration.k8s.io/v1`
> 4. Check operator release notes — confirm supported Kubernetes version range
> 5. Verify add-on compatibility: cert-manager, KEDA, ASO v2, Gatekeeper
> 6. Check Azure CNI / Cilium CNI version compatibility
>
> **Test strategy:**
> 1. Create a staging AKS cluster at 1.30
> 2. Deploy operator + all CRDs to staging
> 3. Run full test suite (envtest + integration tests)
> 4. Smoke test: create CRs, verify reconcile, delete CRs
>
> **Production upgrade order:**
> 1. Upgrade system node pool first (AKS control plane upgrade happens automatically)
> 2. Verify operator still healthy after control plane upgrade
> 3. Upgrade user node pools one at a time
> 4. Verify operator reconciling correctly after each pool upgrade
>
> **Rollback plan:**
> - AKS doesn't support downgrade — plan is: keep old node pool active until confident
> - Have operator rollback image ready: `kubectl set image deployment/operator manager=v1.old`

---

**Q10: How would you debug an operator that has stopped reconciling resources?**

> **Expected Answer (systematic runbook):**
>
> **Step 1 — Is the operator running?**
> ```bash
> kubectl get pods -n operators
> kubectl describe pod <operator-pod> -n operators
> ```
>
> **Step 2 — Check operator logs for errors**
> ```bash
> kubectl logs deployment/database-operator -n operators --tail=200
> kubectl logs deployment/database-operator -n operators -p  # previous crashed container
> ```
>
> **Step 3 — Is there a leader?**
> ```bash
> kubectl get lease database-operator-lock -n operators -o yaml
> # Check holderIdentity — is it set? Is it a currently running pod?
> ```
>
> **Step 4 — Check the specific resource**
> ```bash
> kubectl describe database mydb -n production
> # Events section will show last reconcile activity or errors
> ```
>
> **Step 5 — Check RBAC**
> ```bash
> kubectl auth can-i update databases/status \
>   --as=system:serviceaccount:operators:database-operator -n production
> ```
>
> **Step 6 — Check if finalizer is blocking deletion**
> ```bash
> kubectl get database mydb -o jsonpath='{.metadata.finalizers}'
> ```
>
> **Step 7 — Check webhook health**
> ```bash
> kubectl get validatingwebhookconfigurations
> kubectl apply --dry-run=server -f sample-db.yaml
> # If webhook is broken, this will return an error
> ```
>
> **Step 8 — Check network policy**
> ```bash
> kubectl get networkpolicy -n operators
> # Is there a policy blocking egress to API server (port 443)?
> ```
>
> **Step 9 — Enable debug logging**
> ```bash
> kubectl set env deployment/database-operator \
>   -n operators ZAP_LOG_LEVEL=debug
> ```
>
> **Step 10 — Check cluster events**
> ```bash
> kubectl get events -n operators --sort-by=.lastTimestamp
> kubectl get events -n production --sort-by=.lastTimestamp | grep database
> ```

---

## Coding & Schema Design Questions

---

**Q11: Design the CRD schema for a multi-region application deployment on AKS.**

> **Expected Answer:**

```yaml
apiVersion: mycompany.io/v1
kind: MultiRegionApp
metadata:
  name: payments-service
spec:
  image: mycompany/payments:v2.3.1

  regions:
    - name: eastus
      aksCluster: aks-eastus-prod
      resourceGroup: rg-eastus-prod
      replicas: 5
      weight: 60                       # Traffic % to this region

    - name: westeurope
      aksCluster: aks-westeurope-prod
      resourceGroup: rg-westeurope-prod
      replicas: 3
      weight: 40

  globalLoadBalancer:
    type: azure-front-door             # azure-front-door | azure-traffic-manager
    healthCheckPath: /health
    healthCheckInterval: 30
    failoverThreshold: 50              # Failover if region drops below 50% health

  rollout:
    strategy: canary                   # rolling | canary | bluegreen
    canaryWeight: 10
    canaryAnalysis:
      interval: 5m
      threshold: 5                     # Max error rate % before rollback

  env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: host

status:
  globalEndpoint: payments.mycompany.azurefd.net
  overallPhase: Running

  regions:
    - name: eastus
      phase: Running
      readyReplicas: 5
      endpoint: payments-eastus.mycompany.com
      lastReconcileTime: "2026-03-30T10:00:00Z"

    - name: westeurope
      phase: Running
      readyReplicas: 3
      endpoint: payments-westeurope.mycompany.com
      lastReconcileTime: "2026-03-30T10:01:00Z"

  conditions:
    - type: Ready
      status: "True"
      reason: AllRegionsHealthy
    - type: Progressing
      status: "False"
      reason: Idle
```

> **Bonus points — mention:**
> - The controller needs credentials for multiple AKS clusters → use per-cluster ServiceAccount + kubeconfig stored in Secrets, or Azure Service Principal with access to all clusters
> - The controller should be idempotent for partial failures (one region deployed, other failed)
> - Use finalizers to clean up Front Door routing rules on deletion

---

**Q12: How would you implement a "pause reconciliation" feature in your operator?**

> **Expected Answer:**

```go
// Check for pause at the TOP of Reconcile — before any other logic
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    db := &v1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Support both spec field AND annotation (annotation = emergency, spec = planned)
    isPaused := db.Spec.Paused ||
        db.Annotations["mycompany.io/reconcile-paused"] == "true"

    if isPaused {
        // Emit event so it shows up in `kubectl describe`
        r.Recorder.Event(db, corev1.EventTypeNormal, "Paused",
            "Reconciliation is paused — set spec.paused: false or remove annotation to resume")

        // Update status condition
        setCondition(db, "Paused", "True", "PausedByUser",
            "Reconciliation paused — remove annotation or set spec.paused: false to resume")
        r.Status().Update(ctx, db)

        // No requeue — we only restart when the annotation/spec changes (triggers new event)
        return ctrl.Result{}, nil
    }

    // Clear Paused condition when resuming
    removeCondition(db, "Paused")

    // ... normal reconcile logic ...
}
```

> **Usage:**
> ```bash
> # Pause (emergency)
> kubectl annotate database mydb mycompany.io/reconcile-paused=true
>
> # Pause (planned — via spec)
> kubectl patch database mydb --type=merge -p '{"spec":{"paused":true}}'
>
> # Resume
> kubectl annotate database mydb mycompany.io/reconcile-paused-
> kubectl patch database mydb --type=merge -p '{"spec":{"paused":false}}'
> ```

---

**Q13: Explain the trade-offs between a Helm chart and a Kubernetes Operator for deploying a database.**

> **Expected Answer:**
>
> | | Helm Chart | Operator |
> |---|---|---|
> | Installation (Day 1) | ✅ Simple, declarative | ✅ CRD-driven |
> | Upgrades (Day 2) | ❌ Manual helm upgrade | ✅ Automated, with health checks |
> | Backup/Restore | ❌ Not included | ✅ Built-in if operator implements it |
> | Failure recovery | ❌ None — user must intervene | ✅ Automatic reconcile and heal |
> | Learning curve | Low | High (Go, controller-runtime) |
> | Custom business logic | ❌ Limited (hooks only) | ✅ Full control |
> | Audit/Observability | ❌ Limited | ✅ Custom metrics, events, conditions |
> | Complexity | Low | High |
>
> **When Helm is enough:**
> - Stateless apps, simple config management
> - Team doesn't have Go expertise
> - No Day 2 operational requirements
>
> **When you need an Operator:**
> - Stateful applications needing ordered operations (Kafka, Postgres, Cassandra)
> - Need automated backup, restore, upgrades with quorum awareness
> - Platform team wants to provide self-service database provisioning to other teams
>
> **Hybrid approach** (often best for product companies):
> - Helm Operator wraps existing Helm chart for quick Day 1
> - Evolve to full Go Operator as Day 2 requirements emerge

---

**Q14: How do you ensure your CRD and operator handle a split-brain scenario in AKS (two controllers running simultaneously)?**

> **Expected Answer:**
>
> **Root cause of split-brain:**
> - Leader election lease expires (e.g., network issue prevents renewal)
> - New leader acquires lease
> - Old leader (still alive but disconnected from API server) hasn't stopped yet
>
> **Protections:**
> 1. **Context cancellation**: All reconcile operations use `ctx` — when leader loses election, `ctx.Done()` fires, cancelling in-flight API calls
> 2. **Optimistic concurrency (resourceVersion)**: Every Kubernetes object has a `resourceVersion`. If two controllers try to update the same object, the second one gets a `409 Conflict` — it must re-fetch and retry
> 3. **Server-Side Apply field ownership**: Each controller owns specific fields. SSA prevents two controllers from overwriting each other's fields
> 4. **Idempotent reconcile**: Even if both run, the end result is correct — creating a StatefulSet that already exists returns a `409`, controller handles it gracefully (check `errors.IsAlreadyExists`)
>
> **AKS-specific:**
> - AKS etcd latency can temporarily cause missed lease renewals on heavy clusters
> - Tune `LeaseDuration` (15s → 30s) and `RenewDeadline` (10s → 20s) for large clusters
> - Monitor `database_operator_is_leader` metric to detect dual-leader scenarios

---

## Rapid-Fire Concepts (Common in Phone Screens)

| Question | Short Answer |
|---|---|
| What's the storage version of a CRD? | The version written to etcd. Only one allowed. Others are converted on-read. |
| What does `served: false` mean for a CRD version? | That version's API endpoint is disabled. Objects stored as that version still exist in etcd but can't be accessed until migrated. |
| Why do you need `status: {}` as a subresource? | To separate RBAC for spec writes (users) vs status writes (controllers), and to prevent status updates from triggering spec reconcile. |
| What is `observedGeneration`? | The `.metadata.generation` of the last spec the controller successfully reconciled. Used to tell if status reflects current spec. |
| What's the difference between `Requeue: true` and `RequeueAfter`? | `Requeue: true` = immediate requeue. `RequeueAfter` = requeue after a delay. Both skip exponential backoff. |
| What is pruning in CRDs? | Structural schema enables pruning — unknown fields not in the schema are silently dropped by the API server on admission. |
| What are CEL rules? | `x-kubernetes-validations` — Common Expression Language rules in CRD schema for cross-field validation without a webhook (since k8s 1.25). |
| What's the FieldOwner in SSA? | A string identifying who manages specific fields. Conflicts are detected per-field-owner. Use your controller name. |
| What's `blockOwnerDeletion` in ownerReferences? | Prevents the parent from being deleted until the child is fully deleted. Used with finalizers on the parent. |
| How does KEDA scale to zero? | KEDA bypasses HPA and directly manages the Deployment replicas. When trigger count drops to 0, it sets `replicas: 0`. HPA is paused by KEDA. |

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
| [12](./12-production-best-practices.md) | Production Best Practices |
| **13** | Interview Q&A ← you are here |
| [14](./14-learning-roadmap.md) | Learning Roadmap |
