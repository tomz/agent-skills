---
name: coreweave-gpu
description: CoreWeave GPU workloads — requesting GPUs, multi-GPU jobs, MIG, spot pricing, NCCL tuning, and node selectors for every GPU class
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools:
  - shell
  - read_file
  - write_file
  - glob
  - grep
---

# CoreWeave GPU Workloads Skill

CoreWeave's core product is GPU compute. This skill covers everything from a
single-GPU dev pod to 512-GPU distributed training runs.

---

## 1. GPU Catalog

CoreWeave offers more GPU SKUs than any other cloud provider. Key classes:

| GPU Class Label         | GPU             | VRAM   | NVLink | Best For                         |
|-------------------------|-----------------|--------|--------|----------------------------------|
| `H200_SXM5`             | H200 SXM5       | 141 GB | Yes    | Frontier training, large MoE     |
| `H100_NVLINK_80GB`      | H100 SXM5       | 80 GB  | Yes    | LLM training, FP8 inference      |
| `H100_PCIE`             | H100 PCIe       | 80 GB  | No     | Inference, smaller training      |
| `A100_NVLINK_80GB`      | A100 80GB SXM4  | 80 GB  | Yes    | Training, solid price/perf       |
| `A100_NVLINK`           | A100 40GB SXM4  | 40 GB  | Yes    | Training, widely available       |
| `A100_PCIE_40GB`        | A100 40GB PCIe  | 40 GB  | No     | Inference, batch jobs            |
| `A40`                   | A40             | 48 GB  | No     | Rendering, medium inference      |
| `RTX_A6000`             | RTX A6000       | 48 GB  | No     | Fine-tuning, interactive work    |
| `RTX_A5000`             | RTX A5000       | 24 GB  | No     | Smaller fine-tuning, dev work    |
| `RTX_A4000`             | RTX A4000       | 16 GB  | No     | Light inference, CI/CD           |
| `Quadro_RTX_5000`       | Quadro RTX 5000 | 16 GB  | No     | Legacy; prefer A4000             |
| `GeForce_RTX_3090`      | RTX 3090        | 24 GB  | No     | Budget dev / small experiments   |
| `Tesla_V100_NVLINK_16GB`| V100            | 16 GB  | Yes    | Legacy; cheap throughput         |

Check live availability (node count per GPU class):
```bash
kubectl get nodes -L gpu.nvidia.com/class --no-headers | \
  awk '{print $NF}' | sort | uniq -c | sort -rn
```

---

## 2. Requesting GPUs in Pod Specs

The NVIDIA device plugin exposes GPUs as `nvidia.com/gpu` extended resources.

### Minimal single-GPU pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-dev
  namespace: your-namespace
spec:
  restartPolicy: Never
  tolerations:
    - key: "nvidia.com/gpu"
      operator: "Exists"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu.nvidia.com/class
                operator: In
                values: ["RTX_A6000"]
  containers:
    - name: main
      image: nvcr.io/nvidia/pytorch:24.01-py3
      command: ["sleep", "infinity"]
      resources:
        requests:
          nvidia.com/gpu: "1"
          cpu: "8"
          memory: "32Gi"
        limits:
          nvidia.com/gpu: "1"
          cpu: "8"
          memory: "32Gi"
      volumeMounts:
        - name: dshm
          mountPath: /dev/shm     # Required for PyTorch multiprocessing
  volumes:
    - name: dshm
      emptyDir:
        medium: Memory
        sizeLimit: "16Gi"
```

> **Always set requests == limits for GPUs.** CoreWeave does not support
> burstable GPU requests; fractional GPU allocation is via MIG only.

---

## 3. Multi-GPU Single-Node

Request multiple GPUs on the same node:

```yaml
resources:
  requests:
    nvidia.com/gpu: "8"     # Request all 8 GPUs on one node
  limits:
    nvidia.com/gpu: "8"
```

Verify inside the pod:
```bash
nvidia-smi
nvidia-smi topo -m        # NVLink topology matrix
```

For PyTorch DDP on 8 GPUs:
```bash
torchrun --nproc_per_node=8 --nnodes=1 train.py
```

---

## 4. Multi-Node Distributed Training (PyTorch + Volcano)

CoreWeave recommends **Volcano** for batch/gang scheduling. Install from Apps catalog.

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: llm-training
  namespace: your-namespace
spec:
  minAvailable: 16          # All 16 pods must schedule together (gang scheduling)
  schedulerName: volcano
  plugins:
    ssh: []                 # Volcano sets up passwordless SSH between pods
    svc: []                 # Volcano creates headless service for pod DNS
  tasks:
    - name: master
      replicas: 1
      template:
        spec:
          tolerations:
            - key: "nvidia.com/gpu"
              operator: "Exists"
              effect: "NoSchedule"
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                  - matchExpressions:
                      - key: gpu.nvidia.com/class
                        operator: In
                        values: ["H100_NVLINK_80GB"]
                      - key: topology.kubernetes.io/region
                        operator: In
                        values: ["ORD1"]
          containers:
            - name: trainer
              image: nvcr.io/nvidia/pytorch:24.01-py3
              command:
                - torchrun
                - --nproc_per_node=8
                - --nnodes=16
                - --node_rank=$(VC_TASK_INDEX)
                - --master_addr=$(VC_master_0_HOSTS)
                - --master_port=29500
                - train.py
              env:
                - name: NCCL_DEBUG
                  value: INFO
                - name: NCCL_IB_DISABLE
                  value: "0"          # Enable InfiniBand
                - name: NCCL_NET_GDR_LEVEL
                  value: "5"          # GPUDirect RDMA
              resources:
                requests:
                  nvidia.com/gpu: "8"
                limits:
                  nvidia.com/gpu: "8"
              volumeMounts:
                - name: dshm
                  mountPath: /dev/shm
                - name: model-weights
                  mountPath: /mnt/weights
          volumes:
            - name: dshm
              emptyDir:
                medium: Memory
                sizeLimit: "64Gi"
            - name: model-weights
              persistentVolumeClaim:
                claimName: model-weights
    - name: worker
      replicas: 15           # 15 more workers = 16 total nodes × 8 GPUs = 128 GPUs
      template:
        spec: {}             # Same spec as master (copy in production)
```

---

## 5. NCCL Optimization for Multi-Node

CoreWeave's inter-node fabric is **InfiniBand** (HDR/NDR) on H100/A100 NVLink nodes.

### Essential NCCL environment variables

```yaml
env:
  # InfiniBand / RDMA
  - name: NCCL_IB_DISABLE
    value: "0"
  - name: NCCL_IB_HCA
    value: "mlx5"              # Mellanox HCA name on CW nodes
  - name: NCCL_NET_GDR_LEVEL
    value: "5"                 # GPUDirect RDMA — bypasses CPU for GPU↔GPU across nodes
  - name: NCCL_NET_GDR_READ
    value: "1"

  # Tuning
  - name: NCCL_BUFFSIZE
    value: "4194304"           # 4 MB comm buffer
  - name: NCCL_SOCKET_NTHREADS
    value: "4"
  - name: NCCL_NSOCKS_PERTHREAD
    value: "4"

  # Topology
  - name: NCCL_TOPO_FILE
    value: "/tmp/topo.xml"     # Optional: provide custom topo for CoreWeave node layout

  # Debugging (remove in production)
  - name: NCCL_DEBUG
    value: "WARN"              # INFO = verbose, WARN = errors only
  - name: NCCL_DEBUG_SUBSYS
    value: "GRAPH,TUNING"
```

Test NCCL bandwidth before starting a big run:
```bash
# Inside pod — download nccl-tests and run:
/opt/nccl-tests/build/all_reduce_perf -b 1G -e 1G -f 2 -g 8
```

Expected all-reduce bandwidth on H100 NVLink nodes: **~400 GB/s** intra-node,
**~200 GB/s** inter-node via IB.

---

## 6. GPU Node Selectors and Affinity Patterns

### Prefer one GPU type, fall back to another

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
            - key: gpu.nvidia.com/class
              operator: In
              values: ["H100_NVLINK_80GB"]
      - weight: 50
        preference:
          matchExpressions:
            - key: gpu.nvidia.com/class
              operator: In
              values: ["A100_NVLINK_80GB"]
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: gpu.nvidia.com/class
              operator: In
              values: ["H100_NVLINK_80GB", "A100_NVLINK_80GB", "A100_NVLINK"]
```

---

## 7. GPU Sharing and MIG

### Multi-Instance GPU (MIG) — H100/A100 only

MIG partitions one physical GPU into isolated slices. CoreWeave pre-configures
MIG on some node pools. Request a MIG slice:

```yaml
resources:
  limits:
    nvidia.com/mig-3g.40gb: "1"    # A100: 3 compute slices, 40 GB
    # nvidia.com/mig-1g.10gb: "1"  # Smaller slice
    # nvidia.com/mig-7g.80gb: "1"  # Full GPU as MIG (rarely useful)
```

Available MIG profiles (A100 80 GB):
- `mig-1g.10gb` — 1/7 GPU, 10 GB VRAM
- `mig-2g.20gb` — 2/7 GPU, 20 GB VRAM
- `mig-3g.40gb` — 3/7 GPU, 40 GB VRAM
- `mig-7g.80gb` — full GPU

Check MIG availability:
```bash
kubectl get nodes -L nvidia.com/mig.strategy
kubectl describe node <mig-node> | grep mig
```

> **MIG vs time-slicing:** MIG gives hard memory isolation. Time-slicing (not
> officially supported on CoreWeave) shares one GPU with no memory protection.
> Use MIG for multi-tenant inference; use full GPUs for training.

---

## 8. Spot vs On-Demand vs Reserved Pricing

| Tier       | Discount      | Eviction Risk | Best For                          |
|------------|---------------|---------------|-----------------------------------|
| On-Demand  | 0%            | None          | Inference endpoints, critical jobs|
| Spot       | 50–70%        | May evict     | Fault-tolerant training, batch    |
| Reserved   | 30–45%        | None          | Steady-state inference, always-on |

### Spot GPU pods

Spot eviction sends a SIGTERM 30 seconds before pod termination.
Use `preemptionPolicy` and checkpoint frequently:

```yaml
metadata:
  labels:
    billing.coreweave.com/spot: "true"
spec:
  priorityClassName: spot     # CoreWeave creates this PriorityClass
  terminationGracePeriodSeconds: 60
  containers:
    - name: trainer
      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - "kill -SIGUSR1 1"    # Signal trainer to checkpoint immediately
```

Checkpoint to shared NFS or object storage every N steps so a spot eviction
loses at most N steps of work.

---

## 9. NVIDIA Device Plugin and Monitoring

CoreWeave runs the NVIDIA device plugin and DCGM exporter cluster-wide.
You don't need to install them — they're pre-deployed.

```bash
# Check device plugin is running
kubectl get ds -n kube-system | grep nvidia

# GPU metrics via Prometheus (if Grafana is installed from Apps catalog)
# Metric examples:
#   DCGM_FI_DEV_GPU_UTIL         — GPU utilization %
#   DCGM_FI_DEV_MEM_COPY_UTIL   — Memory bandwidth utilization %
#   DCGM_FI_DEV_FB_USED         — Framebuffer used (VRAM)
#   DCGM_FI_DEV_POWER_USAGE     — Power draw (watts)
#   DCGM_FI_DEV_SM_CLOCK        — SM clock speed

# Quick GPU check inside a pod
nvidia-smi --query-gpu=name,utilization.gpu,memory.used,memory.total,power.draw \
  --format=csv,noheader,nounits
```

---

## 10. Guardrails & Gotchas

- **requests must equal limits** for `nvidia.com/gpu` — Kubernetes GPU resources are not burstable.
- **One GPU per container** — multiple containers in a pod sharing GPUs is not supported. Give all GPUs to one container or use separate pods.
- **GPU node images are large** — `nvcr.io/nvidia/pytorch:24.01-py3` is ~22 GB. Always specify `imagePullPolicy: IfNotPresent` and consider using CoreWeave's image caching / init-container warm-up.
- **NVLink nodes require matching GPU count** — requesting 3 GPUs on an 8-GPU NVLink node wastes 5. Request 1 or 8.
- **Spot evictions are immediate** in high-demand periods — set `checkpointFrequency` to at most 10 minutes for training jobs.
- **MIG nodes cannot mix MIG and non-MIG workloads** — if a node is in MIG mode, every container must request a MIG profile.
- **`/dev/shm` defaults to 64 MB** — PyTorch DataLoader workers will OOM on the default. Always mount a Memory-backed `emptyDir` to `/dev/shm`.
- **IB interfaces are named `mlx5_*`** — set `NCCL_IB_HCA=mlx5` explicitly; auto-detection sometimes picks the wrong interface.
- **H200 nodes are scarce** — reserve in advance for large training runs; spot availability is limited.
