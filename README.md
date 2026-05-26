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
│  KEDA (events)  |  HPA (memory utilization).         │
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
| VKS K8s cluster | Required | — | v1.35.2+vmware.1 |
| Prometheus Operator | Already installed via Addon Mgmt | `tanzu-system-monitoring` | 3.5.1+vmware.1-vks.1 |
| Cert Manager | Already installed via Addon Mgmt | `cert-manager` | 1.19.4+vmware.1-vks.1 |
| Cluster Autoscaler | Already installed via Addon Mgmt | `kube-system` | 1.35.0+vmware.2-vks.1 |
| VPA | Install from this library using Helm Controller | `kube-system` | 1.6.0 |
| Descheduler | Install from this library using Helm Controller | `kube-system` | 0.35.1 |
| KEDA | Install from this library using Helm Controller | `keda` | 2.19.0 |
| kubectl | Required | — | 1.35 |

## 