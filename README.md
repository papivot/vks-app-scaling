# VKS Scaling Solution Design Library

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
│  KEDA (QPS/events)  |  HPA (memory utilization)      │
├──────────────────────────────────────────────────────┤
│  Layer 1: Pod Resources (Vertical)                   │
│  VPA — right-sizes CPU and memory requests/limits    │
├──────────────────────────────────────────────────────┤
│  Layer 0: Application                                │
│  Online Boutique — 11 microservices (gRPC + HTTP)    │
└──────────────────────────────────────────────────────┘
```

## Prerequisites / BOM

| Component | Status | Namespace | Tested Version |
|-----------|--------|-----------|----------------|
| VKS K8s cluster | Required | — | 1.35 |
| Prometheus Operator | Already installed via Addon Mgmt | `tanzu-system-monitoring` | |
| Cert Manager | Already installed via Addon Mgmt | `cert-manager` | |
| Cluster Autoscaler | Already installed via Addon Mgmt | `kube-system` | 1.35 |
| VPA | Install from this library using Helm Controller | `kube-system` | |
| Descheduler | Install from this library using Helm Controller | `kube-system` | 0.35.1 |
| KEDA | Install from this library using Helm Controller | `keda` | 2.19.0 |
| kubectl | Required | — | 1.35 |