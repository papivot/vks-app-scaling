# VKS Scaling Solution Design

A comprehensive, production-ready autoscaling reference for **vSphere Kubernetes Service** clusters running microservice workloads. Uses the [Google Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) e-commerce demo (11 microservices) to illustrate real-world scenarios.

## The Five-Layer Scaling Stack

```
┌──────────────────────────────────────────────────────┐
│  Layer 4: Node Placement (Day-2)                     │
│  Descheduler — rebalances pods after initial schedule│
├──────────────────────────────────────────────────────┤
│  Layer 3: Node Capacity                              │
│  Cluster Autoscaler — adds/removes nodes             │
├──────────────────────────────────────────────────────┤
│  Layer 2: Pod Replicas (Horizontal)                  │
│  KEDA (events)  |  HPA (memory utilization).         │
├──────────────────────────────────────────────────────┤
│  Layer 1: Pod Resources (Vertical)                   │
│  VPA — right-sizes CPU and memory requests/limits    │
├──────────────────────────────────────────────────────┤
│  Layer 0: Application - Sample app ..                │
│  Online Boutique — 11 microservices (gRPC + HTTP)    │
└──────────────────────────────────────────────────────┘
```

## Prerequisites / BOM

| Component | Status | Namespace | Tested Version |
|-----------|--------|-----------|----------------|
| VKS K8s cluster | Required | — | v1.35.2+vmware.1 |
| Prometheus Operator | Already installed via Addon Mgmt | `tanzu-system-monitoring` | 3.5.1+vmware.1-vks.1 |
| Cert Manager | Already installed via Addon Mgmt | `cert-manager` | 1.19.4+vmware.1-vks.1 |
| Cluster Autoscaler | Already installed via Addon Mgmt | `kube-system` | 1.35.0+vmware.2-vks.1 |
| VPA | Install from this library using Helm Controller | `kube-system` | 1.6.0 |
| Descheduler | Install from this library using Helm Controller | `kube-system` | 0.35.1 |
| KEDA | Install from this library using Helm Controller | `keda` | 2.19.0 |
| kubectl | Required | — | 1.35 |

## Architecture Overview

This library implements a multi-layer autoscaling architecture for VKS clusters running microservice workloads. The design uses five complementary tools — each operating at a distinct layer — to address resource efficiency, capacity management, and cost optimization.

## Design Principles

1. **Separation of Concerns:** Each tool owns a single dimension (requests, replicas, placement). No tool overrides another's domain.
2. **Accurate Signals Up the Stack:** VPA provides accurate `requests` → Cluster Autoscaler (CA) makes correct node decisions. Without this, CA over-provisions or under-provisions.
3. **Business Metrics Over Infrastructure Metrics:** KEDA uses HTTP requests and Database queries — signals that directly reflect user impact — rather than CPU% which is distorted by VM overcommit.
4. **Day-2 Scheduling:** The Descheduler corrects placement decisions that were valid at schedule time but are no longer optimal.
5. **Safety by Default:** PDBs, system pod exclusions, and blast radius limits are applied everywhere.

## Component Interaction Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                    VKS Cluster                                       │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Node Level                                                   │   │
│  │                                                               │   │
│  │  ┌──────────────────────┐     ┌────────────────────────────┐  │   │
│  │  │  Cluster Autoscaler  │     │       Descheduler          │  │   │
│  │  │  (scale nodes in/out)│     │  (rebalance pod placement).│  │   │
│  │  │                      │     │                            │  │   │
│  │  │  Input: pod requests │←─── │  Output: evicts pods       │  │   │
│  │  │  (provided by VPA)   │     │  Input: node utilization   │  │   │
│  │  └──────────────────────┘     └────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                         ↑ nodes added/removed                        │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Pod Level                                                    │   │
│  │                                                               │   │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────┐  │   │
│  │  │      VPA        │  │      KEDA        │  │     HPA      │  │   │
│  │  │  (right-sizing) │  │ (event-driven    │  │ (CPU/memory  │  │   │
│  │  │  requests/limits│  │  replica count)  │  │  replicas)   │  │   │
│  │  └─────────────────┘  └──────────────────┘  └──────────────┘  │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                         ↑ pods resized/added/removed                 │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Application Layer (online-boutique namespace)                │   │
│  │                                                               │   │
│  │  frontend  currencyservice  recommendationservice  ...        │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────┐                         │
│  │  Observability (tanzu-system-monitoring)│                         │
│  │  Prometheus Operator                    │                         │
│  │  http_requests_total, grpc_server_...   │                         │
│  └─────────────────────────────────────────┘                         │
└──────────────────────────────────────────────────────────────────────┘
```

## Data Flow: How Components Feed Each Other

```
Application Pods
    │
    │ expose /metrics (Prometheus annotations)
    ▼
Prometheus (tanzu-system-monitoring)
    │
    ├──► KEDA reads QPS metrics → adjusts replica count → more/fewer pods
    │
    └──► Grafana dashboards (optional)

VPA Recommender
    │ observes CPU/memory usage via metrics-server
    ▼
VPA sets accurate requests on pods
    │
    └──► Cluster Autoscaler reads requests for node sizing decisions
         └──► CA adds nodes (under-capacity) or removes nodes (over-capacity)

Cluster Autoscaler adds new nodes
    │
    └──► Descheduler
         └──► Rebalances pods across old and new nodes
              └──► Scheduler places evicted pods optimally
```

## Service-to-Scaler Assignment

| Service | Language | VPA Mode | Horizontal Scaler | PDB |
|---------|----------|----------|-------------------|-----|
| `frontend` | Go | Off (advisory) | KEDA (100 QPS/pod) | Yes (min 1) |
| `currencyservice` | Node.js | Off (advisory) | KEDA (200 QPS/pod) | Yes (min 1) |
| `recommendationservice` | Python | InPlaceOrRecreate | HPA (70% memory) | Yes (min 1) |
| `checkoutservice` | Go | Auto | HPA (70% memory) | Yes (min 1) |
| `productcatalogservice` | Go | Auto | HPA (70% memory) | — |
| `cartservice` | C# | Auto | HPA (75% memory) | — |
| `paymentservice` | Node.js | Auto | None | — |
| `emailservice` | Python | Auto | None | — |
| `shippingservice` | Go | Auto | None | — |
| `adservice` | Java | Auto | None | — |
| `redis-cart` | Redis | Initial | None | — |


* Indirect: VPA sets accurate requests → CA has correct data for all scenarios
```

## Conflict Prevention Architecture

The most critical design decision is **preventing VPA from fighting HPA**. The solution is role separation:

```
CPU Scaling Domain:
    VPA owns: cpu requests/limits (vertical)
    No HPA on CPU for VPA-managed services

Memory Scaling Domain:
    HPA triggers on: memory utilization % (horizontal)
    VPA also manages memory requests (vertical) — these are orthogonal

QPS/External Metric Domain:
    KEDA owns: replica count for frontend, currencyservice
    VPA set to Off for these services (advisory only)

Node Domain:
    CA owns: node count
    Descheduler: pod placement (not count)
    These two are fully orthogonal
```

## Descheduler Strategy Selection Guide

```
                    Current cluster state?
                           │
             ┌─────────────┴─────────────┐
             │                           │
    Pods unevenly spread            Pods spread, but
    after CA scale-out              nodes underutilized
             │                           │
    Use: LowNodeUtilization        Use: HighNodeUtilization
    (balance-policy.yaml)          (binpacking-policy.yaml)
    Goal: HA even spread           Goal: Consolidate, free nodes
             │                           │
    Scenario 2 (SRE)              Scenario 3 (Manager)
```

> **Never run LowNodeUtilization and HighNodeUtilization simultaneously.** They evict in opposite directions and will cause infinite eviction loops.

