# VPA Component

## Role

VPA is the **"Right-Sizer"** — it observes historical CPU/memory usage and automatically adjusts pod `requests` and `limits` to match actual consumption. This makes resource requests accurate, which in turn gives Cluster Autoscaler correct signals for node scale-in/out decisions.

## Architecture

```
VPA Recommender  ←──── metrics-server / Prometheus
       │  writes recommendations
       ▼
VPA Checkpoint CRD
       │
VPA Updater ──────────── evicts pods (if mode = Auto/Recreate)
       │                 or patches in-place (mode = InPlaceOrRecreate)
       ▼
VPA Admission Controller ──── mutates new pods on creation
```

## Update Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `Off` | Recommends only, never changes pods | KEDA/HPA-managed services (advisory) |
| `Initial` | Sets requests on pod creation only | Stateful workloads (redis, databases) |
| `Auto` | Full lifecycle management; recreates pods | General services |
| `InPlaceOrRecreate` | In-place resize first, recreate if needed | K8s 1.27+; minimal disruption |

## Service Mode Assignment

| Service | Mode | Reason |
|---------|------|--------|
| `frontend` | `Off` | KEDA manages replicas - with HTTP ADDON |
| `currencyservice` | `Off` | KEDA manages replicas |
| `recommendationservice` | `InPlaceOrRecreate` | Scenario 1 - OOM recovery target |
| `checkoutservice` | `Auto` | High criticality, needs PDB |
| `productcatalogservice` | `Auto` | Moderate traffic |
| `cartservice` | `Auto` | Stateless-like session store |
| `paymentservice` | `Auto` | Financial - tight bounds |
| `emailservice` | `Auto` | Low traffic, safe |
| `shippingservice` | `Auto` | Low traffic, safe |
| `adservice` | `Auto` | Java - high baseline memory |
| `redis-cart` | `Initial` | Stateful, live resize unsafe |

## VPA + HPA Conflict Prevention

**Rule:** Never run VPA in `Auto`/`InPlaceOrRecreate` mode on the same container that an HPA scales based on **CPU** utilization.

- For KEDA-managed services (`frontend`, `currencyservice`): VPA mode is `Off`. VPA recommendations are still generated and visible — use them to inform manual `requests` tuning.
- For services with CPU-based HPA: set VPA to `Off` or use VPA only on memory (`controlledResources: ["memory"]`).
- For standalone services: VPA `Auto` is safe.

## Installation

```bash
# 1. Install VPA (includes CRDs, admission controller, recommender, updater)
kubectl apply -f install/vpa-install.yaml

# 2. Wait for VPA components to be ready
kubectl rollout status deployment/vpa-recommender -n kube-system
kubectl rollout status deployment/vpa-updater -n kube-system
kubectl rollout status deployment/vpa-admission-controller -n kube-system

# 3. Apply VPA objects for Online Boutique
kubectl apply -f objects/vpa-objects.yaml

# 4. View recommendations (allow 5-10 min for initial data)
kubectl describe vpa vpa-recommendationservice -n online-boutique
```

## Verify

```bash
# Check all VPA objects
kubectl get vpa -n online-boutique -o wide

# View current recommendation for recommendationservice
kubectl get vpa vpa-recommendationservice -n online-boutique -o jsonpath='{.status.recommendation}' | jq .

# Check VPA updater events
kubectl get events -n kube-system --field-selector reason=EvictedByVPA
```

## Key Tuning Parameters (vpa-install.yaml)

| Parameter | Value | Effect |
|-----------|-------|--------|
| `--recommendation-margin-fraction` | `0.15` | 15% safety buffer above observed peak |
| `--pod-recommendation-min-cpu-millicores` | `25` | Floor for CPU recommendations |
| `--pod-recommendation-min-memory-mb` | `32` | Floor for memory recommendations |
| `--min-replicas` (updater) | `2` | Won't evict if only 1 replica running |
| `--evict-after-oom-threshold` | `10m` | Evicts and resizes pods that OOM within 10 min |
