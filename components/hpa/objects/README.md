# HPA Component

## Role

HPA provides **replica-level scaling** based on resource utilization. In this library, HPA is intentionally scoped to services where CPU/memory utilization is a reliable signal and where neither KEDA nor VPA-only is sufficient.

## Limitations Addressed by This Design

| HPA Limitation | Mitigation in This Library |
|----------------|---------------------------|
| CPU metrics unreliable on over-committed VMs | Scale on **memory** (not CPU) for most services |
| No business metrics (QPS, Queue Depth, Datatbase metrics) | **KEDA** handles `frontend` and `currencyservice` |
| VPA + HPA CPU conflict | HPA scales on **memory**; VPA manages **CPU** right-sizing |

## Service Coverage

| Service | Scaler | Rationale |
|---------|--------|-----------|
| `frontend` | KEDA | QPS-based; business metric |
| `currencyservice` | KEDA | Highest QPS; gRPC rate |
| `recommendationservice` | VPA only | Scenario 1; OOM auto-recovery |
| `checkoutservice` | HPA (memory) | Critical checkout path |
| `productcatalogservice` | HPA (memory) | Spikes with traffic |
| `cartservice` | HPA (memory) | Active user sessions |
| `emailservice` | VPA Auto | Low traffic |
| `paymentservice` | VPA Auto | Low replica count acceptable |
| `shippingservice` | VPA Auto | Low traffic |
| `adservice` | VPA Auto | Java startup cost |
| `redis-cart` | VPA Initial | Stateful |

## VPA + HPA Coexistence Rules

```
IF service has HPA on CPU  →  set VPA mode = Off OR controlledResources = ["memory"]
IF service has KEDA         →  set VPA mode = Off (recommendations only)
IF service has HPA on memory →  VPA can manage CPU in Auto mode (no conflict)
```

All HPAs in this library scale on **memory utilization** to allow VPA to independently right-size CPU.

## Scale Behavior Summary

| Service | Min | Max | Trigger | Scale-down stabilization |
|---------|-----|-----|---------|--------------------------|
| `checkoutservice` | 2 | 10 | 70% memory | 180s |
| `productcatalogservice` | 2 | 10 | 70% memory | 120s |
| `cartservice` | 2 | 8 | 75% memory | 300s |

## Usage

```bash
kubectl apply -f objects/hpa-objects.yaml

# Verify
kubectl get hpa -n online-boutique

# Watch scaling events
kubectl describe hpa hpa-checkoutservice -n online-boutique
kubectl get events -n online-boutique --field-selector reason=SuccessfulRescale
```
