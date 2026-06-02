# VKS Add-on Management Manifests

This directory contains example VKS Add-on Management manifests for installing workload scaling add-ons on VKS clusters.

These examples were verified with `VsphereKubernetesService 3.7.0`. In VKS 3.7.0, the Helm controller is installed automatically, so these examples do not include a separate Helm controller installation manifest.

## Directory Layout

```text
vks-addon-management/
  README.md
  vpa/
    vpa-addonrepository.yaml
    vpa-addonrepositoryinstall.yaml
    vpa-addonconfig.yaml
    vpa-addoninstall.yaml
  keda/
    keda-addonrepository.yaml
    keda-addonrepositoryinstall.yaml
    keda-addonconfig.yaml
    keda-addoninstall.yaml
  descheduler/
    descheduler-addonrepository.yaml
    descheduler-addonrepositoryinstall.yaml
    descheduler-addonconfig.yaml
    descheduler-addoninstall.yaml
```

## Add-ons

| Add-on | Purpose | Selected release |
| --- | --- | --- |
| Vertical Pod Autoscaler (VPA) | Recommends or updates pod CPU and memory requests | `vertical-pod-autoscaler.0.9.0` |
| KEDA | Scales workloads horizontally from event, queue, schedule, or Prometheus metrics | `keda.2.19.0` |
| Descheduler | Evicts eligible pods so the Kubernetes scheduler can rebalance placement | `descheduler.0.34.0` |

## Before You Apply

- Review and update the namespace used by `AddonConfig` and `AddonInstall` resources. The examples use `test-ns`.
- Update each `AddonConfig` name to match your workload cluster name. The Add-on Management system expects the `AddonConfig` name to follow the pattern `{clustername}-{addonname}`. If the name does not follow this pattern, the add-on framework will not use your custom values and will create an empty `AddonConfig` with default values.
- Review `spec.clusters: []` in each `AddonInstall`. If your VKS environment requires explicit cluster targeting, populate this field before applying.
- Validate the selected add-on release against your VKS version and support policy.
- Install only the add-ons required by your workload scenario. Do not treat the full set as a production-certified bundle unless you have tested the add-ons together with representative workloads in the same cluster profile.

For example, if your cluster name is `prod-vks-01`, use:

| Add-on | Example `AddonConfig` name |
| --- | --- |
| VPA | `prod-vks-01-vertical-pod-autoscaler` |
| KEDA | `prod-vks-01-keda` |
| Descheduler | `prod-vks-01-descheduler` |

## Apply Order

Apply each add-on in this order:

1. `AddonRepository`
2. `AddonRepositoryInstall`
3. `AddonConfig`
4. `AddonInstall`

## Install VPA

```bash
kubectl apply -f vks-addon-management/vpa/vpa-addonrepository.yaml
kubectl apply -f vks-addon-management/vpa/vpa-addonrepositoryinstall.yaml
kubectl apply -f vks-addon-management/vpa/vpa-addonconfig.yaml
kubectl apply -f vks-addon-management/vpa/vpa-addoninstall.yaml
```

Validate from the workload cluster:

```bash
kubectl get Pod -n kube-system | grep vpa
kubectl get CustomResourceDefinition | grep verticalpodautoscalers
```

## Install KEDA

```bash
kubectl apply -f vks-addon-management/keda/keda-addonrepository.yaml
kubectl apply -f vks-addon-management/keda/keda-addonrepositoryinstall.yaml
kubectl apply -f vks-addon-management/keda/keda-addonconfig.yaml
kubectl apply -f vks-addon-management/keda/keda-addoninstall.yaml
```

Validate from the workload cluster:

```bash
kubectl get Pod -n kube-system | grep keda
kubectl get CustomResourceDefinition | grep keda
kubectl get ScaledObject -A
```

## Install Descheduler

```bash
kubectl apply -f vks-addon-management/descheduler/descheduler-addonrepository.yaml
kubectl apply -f vks-addon-management/descheduler/descheduler-addonrepositoryinstall.yaml
kubectl apply -f vks-addon-management/descheduler/descheduler-addonconfig.yaml
kubectl apply -f vks-addon-management/descheduler/descheduler-addoninstall.yaml
```

Validate from the workload cluster:

```bash
kubectl get Pod -n kube-system | grep descheduler
kubectl logs -n kube-system Deployment/descheduler
```

## Production Guidance

For production use, validate one autoscaling behavior at a time before combining multiple control loops:

- Use VPA in `Off` mode first to collect recommendations before enabling automated updates.
- Use KEDA with bounded `minReplicaCount` and `maxReplicaCount` values based on application and downstream capacity.
- Use Descheduler only after workloads have appropriate replica counts, readiness probes, and PodDisruptionBudgets.
- Confirm metrics availability before relying on VPA recommendations or KEDA Prometheus triggers.
- Monitor pod restarts, pending pods, node scale events, KEDA scaler errors, VPA recommendations, and Descheduler evictions during rollout.

The main risks of unvalidated production use are control-loop conflicts, unexpected pod eviction or restart, dependency on unavailable metrics, over-scaling cost, under-scaling during traffic spikes, webhook or CRD compatibility issues, and node scale behavior that differs from the example environment.
