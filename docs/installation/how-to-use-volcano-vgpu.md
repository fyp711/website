---
title: Use Volcano vGPU
---

# Volcano vGPU device plugin for Kubernetes

:::note

You *DON'T* need to install HAMi when using volcano-vgpu, only use  
[Volcano vGPU device-plugin](https://github.com/Project-HAMi/volcano-vgpu-device-plugin) is good enough. It can provide device-sharing mechanism for NVIDIA devices managed by Volcano.

This is based on [Nvidia Device Plugin](https://github.com/NVIDIA/k8s-device-plugin), it uses [HAMi-core](https://github.com/Project-HAMi/HAMi-core) to support hard isolation of GPU cards.

Volcano vGPU is only available in Volcano > v1.9.

:::

## Quick Start

### Configure scheduler

Update the scheduler configuration:

```shell
kubectl edit cm -n volcano-system volcano-scheduler-configmap
```

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: volcano-scheduler-configmap
  namespace: volcano-system
data:
  volcano-scheduler.conf: |
    actions: "enqueue, allocate, backfill"
    tiers:
    - plugins:
      - name: priority
      - name: gang
      - name: conformance
    - plugins:
      - name: drf
      - name: deviceshare
        arguments:
          deviceshare.VGPUEnable: true # enable vgpu
      - name: predicates
      - name: proportion
      - name: nodeorder
      - name: binpack
```

### Enabling GPU Support in Kubernetes

Once you have enabled this option on *all* the GPU nodes you wish to use,
you can then enable GPU support in your cluster by deploying the following DaemonSet:

```shell
kubectl create -f https://raw.githubusercontent.com/Project-HAMi/volcano-vgpu-device-plugin/main/volcano-vgpu-device-plugin.yml
```

### Verify environment is ready

Check the node status, it is ok if `volcano.sh/vgpu-number` is included in the allocatable resources.

```shell
kubectl get node {node name} -oyaml
```

```yaml
...
status:
  addresses:
  - address: 172.17.0.3
    type: InternalIP
  - address: volcano-control-plane
    type: Hostname
  allocatable:
    cpu: "4"
    ephemeral-storage: 123722704Ki
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 8174332Ki
    pods: "110"
    volcano.sh/gpu-number: "10"    # vGPU resource
  capacity:
    cpu: "4"
    ephemeral-storage: 123722704Ki
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 8174332Ki
    pods: "110"
    volcano.sh/gpu-memory: "89424"
    volcano.sh/gpu-number: "10"   # vGPU resource
```

### Running vGPU Jobs

vGPU can be requested by both set "volcano.sh/vgpu-number", "volcano.sh/vgpu-cores" and "volcano.sh/vgpu-memory" in resources.limits.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod1
spec:
  containers:
    - name: cuda-container
      image: nvidia/cuda:9.0-devel
      command: ["sleep"]
      args: ["100000"]
      resources:
        limits:
          volcano.sh/vgpu-number: 2 # requesting 2 gpu cards
          volcano.sh/vgpu-memory: 3000 # (optinal)each vGPU uses 3G device memory
          volcano.sh/vgpu-cores: 50 # (optional)each vGPU uses 50% core
EOF
```

You can validate device memory using nvidia-smi inside container:

:::warning

If you don't request GPUs when using the device plugin with NVIDIA images, all
the GPUs on the machine will be exposed inside your container.
The number of vGPUs used by a container can not exceed the number of GPUs on that node.

:::

### Monitor

volcano-scheduler-metrics records all GPU usage and limits. Visit the following address to get the metrics.

```shell
curl {volcano scheduler cluster ip}:8080/metrics
```
