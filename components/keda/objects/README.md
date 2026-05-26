# KEDA Component

## Role

KEDA is the **"Event-Driven Scaler"** — it extends Kubernetes HPA with external and custom metric triggers. Unlike native HPA which is limited to CPU/memory, KEDA can scale workloads based on any observable signal: queue depth, database row count, Prometheus queries, and more.

## Architecture

```
External Metrics (Redis Queue, Prometheus, etc.)
          │
    KEDA Operator
          │   creates/updates
          ▼
  HPA (managed by KEDA)
          │   scales replicas
          ▼
     Deployment
```

KEDA creates and manages an HPA on your behalf. The HPA is driven by custom metrics surfaced through the KEDA metrics server (implements the external metrics API).

## Scaler: Redis Cart Database (Current)

The active ScaledObject uses the **Redis Database Size Scaler** to drive autoscaling based on cart activity.

- **How it works:** KEDA monitors the Redis cart database for key count, scales `currencyservice` based on active cart transactions.
- **Redis:** `redis-cart.online-boutique.svc.cluster.local:6379` (shared with Online Boutique shopping cart)
- **Trigger:** Database size (number of keys) — each pod handles ~100 active carts.
- **Scale-to-zero:** Not enabled (`minReplicaCount: 1`) — keeps at least one pod running for cart operations.

**Scaling formula:**
```
desiredReplicas = ceil(db_size / dbSizeTrigger)

Example (dbSizeTrigger: 100):
  - 0 keys    → 1 pod (minReplicaCount)
  - 100 keys  → 1 pod
  - 250 keys  → 3 pods
  - 500 keys  → 5 pods
  - 1500 keys → 15 pods (hits maxReplicaCount)
```

## Scale Behavior

The Redis ScaledObject for `currencyservice` configures:

| Parameter | Value |
|-----------|-------|
| `minReplicaCount` | 1 (always at least 1 pod for cart operations) |
| `maxReplicaCount` | 15 |
| `pollingInterval` | 10s |
| `cooldownPeriod` | 120s |
| Scale-down stabilization | 180s (stable idle detection) |
| Scale-up max step | 5 pods per 15s (immediate response to cart activity) |

## Installation

```bash
# 1. Add Helm repo
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# 2. Install KEDA (see install/keda-install.yaml for values)
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --version 2.16.0 \
  -f install/keda-install.yaml

# 3. Verify
kubectl get pods -n keda
kubectl get crd | grep keda

# 4. Apply Redis ScaledObject
kubectl apply -f objects/scaledobject-redis.yaml

# 5. Check ScaledObjects
kubectl get scaledobject -n online-boutique
kubectl describe scaledobject keda-currencyservice-redis-demo -n online-boutique
```

## Verify Scaling

```bash
# Watch replicas change as cart activity increases
kubectl get hpa -n online-boutique --watch

# Check KEDA operator logs for scaling decisions
kubectl logs -n keda -l app=keda-operator --tail=50

# Monitor Redis cart database size
kubectl exec redis-cart -n online-boutique -- redis-cli DBSIZE

# Generate cart activity (via loadgenerator)
kubectl scale deployment loadgenerator -n online-boutique --replicas=3

# Observe scaling response as cart keys increase
```

## KEDA + VPA Coexistence

KEDA manages **replica count** (horizontal). VPA manages **resource requests** (vertical). These are orthogonal and do NOT conflict. VPA mode is set to `Off` for KEDA-managed services, meaning VPA generates recommendations but does not apply them — recommendations can be used to inform manual request tuning.
