# Descheduler Component — Day-2 Pod Rebalancer

## Role

The Descheduler is the **"Re-balancer"** — it fixes pod placement that has gone stale after initial scheduling. Kubernetes' default scheduler places pods once at creation time and never moves them. The Descheduler runs periodically to evict mis-placed pods, allowing the scheduler to re-place them optimally.

## Why CronJob (Not Deployment)?

| Mode | Behavior | Risk |
|------|----------|------|
| CronJob | Runs on schedule, stops when done | Predictable, easy to audit |
| Deployment | Runs continuously, watches cluster | Can cause churn loops; harder to tune |

**Recommendation:** CronJob running every 5 minutes. This gives controlled windows of eviction activity, with clear start/stop boundaries visible in logs.

## Strategy Reference

### Balance Strategies (process pods as a group)

| Strategy | What It Does |
|----------|-------------|
| `RemoveDuplicates` | Ensures only one pod from each RS/Deployment per node |
| `LowNodeUtilization` | Moves pods from over-utilized to under-utilized nodes |
| `HighNodeUtilization` | Moves pods OFF under-utilized nodes (bin-packing / cost saving) |

### Deschedule Strategies (process pods one-by-one)

| Strategy | What It Does |
|----------|-------------|
| `RemovePodsViolatingNodeAffinity` | Evicts pods on nodes that no longer match their affinity rules |
| `RemovePodsViolatingNodeTaints` | Evicts pods on nodes with new taints the pod doesn't tolerate |

## Policy Files

| File | Strategies | Scenario |
|------|-----------|----------|
| `balance-policy.yaml` | `RemoveDuplicates` + `LowNodeUtilization` | Scenario 2 (SRE) — even spread after CA scale-out |
| `deschedule-policy.yaml` | `RemovePodsViolatingNodeAffinity` + `RemovePodsViolatingNodeTaints` | Any — topology compliance |
| `binpacking-policy.yaml` | `HighNodeUtilization` | Scenario 3 (Manager) — cost consolidation |

## Critical Safety Controls (All Policies)

All policies enforce these protections:

1. **PDB Respect:** `DefaultEvictor` plugin checks PodDisruptionBudgets before every eviction. If eviction would violate a PDB, the pod is skipped.
2. **System pod exclusion:** `priorityThreshold: system-cluster-critical` — pods with this or higher priority class are never evicted.
3. **Namespace exclusion:** `kube-system`, `keda`, `cert-manager`, `tanzu-system-monitoring`, and VMware system namespaces are excluded.
4. **Blast radius limits:** `maxNoOfPodsToEvictPerNode`, `maxNoOfPodsToEvictPerNamespace`, `maxNoOfPodsToEvictTotal` are set on every policy.

## Scheduling Loop Prevention

**Problem:** Descheduler evicts pod from Node A; scheduler places it back on Node A (same node wins the scoring).

**Solutions implemented:**
1. `nodeFit: true` on `DefaultEvictor` — only evicts pods where a valid alternative node exists.
2. Use `PodAntiAffinity` (in scenario manifests) to steer the scheduler away from origin nodes.
3. Balance strategy thresholds are set wide enough (20% gap) that minor imbalances don't trigger evictions.

## Installation

```bash
# 1. Install RBAC
kubectl apply -f install/descheduler-install.yaml

# 2. Apply the policy ConfigMaps you need
kubectl apply -f policies/balance-policy.yaml
kubectl apply -f policies/deschedule-policy.yaml
kubectl apply -f policies/binpacking-policy.yaml

# 3. Deploy CronJob (see scenarios/ for specific CronJob manifests)
kubectl apply -f ../../scenarios/scenario-2-sre/descheduler-cronjob.yaml    # balance
kubectl apply -f ../../scenarios/scenario-3-manager/descheduler-cronjob.yaml  # bin-packing
```

## Verify

```bash
# View descheduler pod logs from last run
kubectl get pods -n kube-system | grep descheduler
kubectl logs -n kube-system <descheduler-pod-name>

# Check eviction events
kubectl get events -n online-boutique --field-selector reason=Evicted

# Manually trigger a CronJob run for testing
kubectl create job --from=cronjob/descheduler descheduler-manual-$(date +%s) -n kube-system
```

## Version Mapping (VKS / K8s compatibility)

See `docs/version-matrix.md` for the full compatibility table.

| K8s Version | Descheduler Version |
|-------------|---------------------|
| 1.27 | v0.27.x |
| 1.28 | v0.28.x |
| 1.29 | v0.29.x |
| 1.30 | v0.30.x |
| 1.31 | v0.31.x |
