---
name: do-k8s
description: DigitalOcean Kubernetes (DOKS), node pools, auto-scaling, Container Registry, and monitoring
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

# DigitalOcean Kubernetes (DOKS) Skill

## Overview

DOKS is a managed Kubernetes service — DO manages the control plane (API server, etcd,
scheduler) at no charge. You pay only for worker nodes (Droplets), load balancers, and volumes.

---

## Creating a Cluster

```bash
# Minimal cluster — 2 nodes, basic size
doctl kubernetes cluster create dev-cluster \
  --region nyc3 \
  --node-pool "name=default;size=s-2vcpu-4gb;count=2" \
  --version latest

# Production cluster — auto-scaling node pool
doctl kubernetes cluster create prod-cluster \
  --region nyc3 \
  --node-pool "name=workers;size=s-4vcpu-8gb;count=3;min-nodes=2;max-nodes=10;auto-scale=true;tag=role:worker" \
  --version 1.29  \
  --maintenance-policy "day=sunday;start_time=03:00" \
  --ha   # highly available control plane (3 replicas, small upcharge)

# Multi-pool cluster (web + compute workloads separated)
doctl kubernetes cluster create mixed-cluster \
  --region nyc3 \
  --node-pool "name=web;size=s-2vcpu-4gb;count=2;tag=role:web" \
  --node-pool "name=compute;size=c-4;count=2;tag=role:compute" \
  --version latest
```

### Available Kubernetes versions

```bash
doctl kubernetes options versions

# Available node sizes
doctl kubernetes options sizes

# Available regions
doctl kubernetes options regions
```

---

## kubeconfig

```bash
# Download and merge kubeconfig (sets current context automatically)
doctl kubernetes cluster kubeconfig save prod-cluster

# Save to a specific file
doctl kubernetes cluster kubeconfig save prod-cluster --kubeconfig ~/.kube/do-prod.yaml

# Set KUBECONFIG to use multiple config files
export KUBECONFIG=~/.kube/config:~/.kube/do-prod.yaml

# Verify kubectl works
kubectl get nodes
kubectl cluster-info
```

---

## Cluster Management

```bash
# List clusters
doctl kubernetes cluster list

# Get cluster details
doctl kubernetes cluster get prod-cluster

# Update cluster name
doctl kubernetes cluster update prod-cluster --new-name production

# Upgrade Kubernetes version
doctl kubernetes cluster upgrade prod-cluster --version 1.30

# Delete cluster (also deletes all nodes)
doctl kubernetes cluster delete prod-cluster
```

---

## Node Pools

Node pools let you have different Droplet sizes for different workload types.

```bash
# List node pools
doctl kubernetes cluster node-pool list prod-cluster

# Get pool details
doctl kubernetes cluster node-pool get prod-cluster $POOL_ID

# Add a new pool
doctl kubernetes cluster node-pool create prod-cluster \
  --name gpu-pool \
  --size s-4vcpu-8gb \
  --count 2 \
  --tag role:gpu

# Scale a pool manually
doctl kubernetes cluster node-pool update prod-cluster $POOL_ID \
  --count 5

# Enable auto-scaling on an existing pool
doctl kubernetes cluster node-pool update prod-cluster $POOL_ID \
  --auto-scale \
  --min-nodes 2 \
  --max-nodes 8

# Delete a pool (pods are evicted first)
doctl kubernetes cluster node-pool delete prod-cluster $POOL_ID

# Recycle nodes (replace all nodes one by one — for major updates)
doctl kubernetes cluster node-pool recycle prod-cluster $POOL_ID \
  --node-ids node-id-1,node-id-2
```

---

## DigitalOcean Container Registry (DOCR)

DOCR stores Docker images. Integrates natively with DOKS — no credentials needed.

```bash
# Create registry (one per account, region-based)
doctl registry create my-registry --region nyc3 --subscription-tier basic

# tiers: starter (free, 1 repo), basic ($5/mo), professional ($20/mo)

# Authenticate Docker with DOCR
doctl registry login

# Build and push an image
docker build -t registry.digitalocean.com/my-registry/my-app:v1.0.0 .
docker push registry.digitalocean.com/my-registry/my-app:v1.0.0

# List repositories
doctl registry repository list

# List tags for a repo
doctl registry repository list-tags my-registry/my-app

# Delete a tag
doctl registry repository delete-tag my-registry/my-app v0.9.0

# Run garbage collection (reclaim storage from deleted tags)
doctl registry garbage-collection start my-registry

# Integrate registry with cluster (injects pull credentials as a K8s secret)
doctl registry kubernetes-manifest | kubectl apply -f -
```

### Using DOCR images in Kubernetes

```yaml
# After running: doctl registry kubernetes-manifest | kubectl apply -f -
# The imagePullSecret is automatically available.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: registry.digitalocean.com/my-registry/my-app:v1.0.0
      imagePullSecrets:
        - name: registry-my-registry  # created by doctl registry kubernetes-manifest
```

---

## 1-Click Apps (Helm Charts)

DOKS supports deploying common apps via the marketplace:

```bash
# List available 1-Click apps
doctl kubernetes 1-clicks list

# Install via the control panel UI, or use Helm directly:
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args[0]="--kubelet-insecure-tls"
```

---

## Monitoring Stack

DOKS integrates with DO Monitoring and optionally the Metrics Server / kube-state-metrics.

```bash
# Enable cluster metrics (installs metrics-server via marketplace 1-click)
# Or install manually:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check node resource usage
kubectl top nodes

# Check pod resource usage
kubectl top pods --all-namespaces

# DO alert policies for cluster
doctl monitoring alert create \
  --type v1/insights/cluster/memory_utilization_percent \
  --compare GreaterThan \
  --value 80 \
  --window 5m \
  --entities $CLUSTER_ID \
  --description "High cluster memory" \
  --notifications '{"email": ["ops@example.com"]}'
```

---

## Storage Classes in DOKS

DO provides a CSI driver that provisions block volumes (Volumes) automatically.

```yaml
# Default storage class: do-block-storage
# Creates a DO Volume (persistent disk) per PVC

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes:
    - ReadWriteOnce       # block storage is single-node only
  storageClassName: do-block-storage
  resources:
    requests:
      storage: 10Gi       # minimum 1Gi, billed per GB
```

```bash
# List storage classes
kubectl get storageclass

# do-block-storage           — standard (network SSD)
# do-block-storage-retain    — PVC delete doesn't delete the volume
# do-block-storage-xfs       — XFS filesystem instead of ext4
```

---

## Useful kubectl Patterns for DOKS

```bash
# Get all resources in a namespace
kubectl get all -n my-namespace

# Watch rollout status
kubectl rollout status deployment/my-app

# Force restart a deployment (zero-downtime rolling restart)
kubectl rollout restart deployment/my-app

# Scale a deployment
kubectl scale deployment my-app --replicas=5

# Get logs from all pods of a deployment
kubectl logs -l app=my-app --all-containers --tail=100 -f

# Exec into a pod
kubectl exec -it $(kubectl get pod -l app=my-app -o name | head -1) -- /bin/sh

# Get events (troubleshooting)
kubectl get events --sort-by='.lastTimestamp' -A
```

---

## Maintenance Windows

```bash
# Set/update maintenance window
doctl kubernetes cluster update prod-cluster \
  --maintenance-policy "day=sunday;start_time=03:00"

# What happens during maintenance:
# - Minor Kubernetes version patches applied
# - Node OS security patches applied
# - HA control plane: rolling, no downtime
# - Single-control-plane: brief API server downtime
```

---

## Guardrails

- **`--ha` for production** — control plane HA prevents API downtime during updates
- **Auto-scaling needs resource requests set** on pods — the CA uses these to decide when to scale
- **`ReadWriteOnce` volumes** can't be shared between nodes — use Spaces for shared storage
- **`doctl registry kubernetes-manifest`** must be re-run after adding new clusters
- **Node pool recycle** for OS updates — avoids manual node draining
- **Upgrade one minor version at a time** — Kubernetes doesn't support skipping minors
- **Tag node pools** — use `nodeSelector` or taints to route workloads to the right pool
- **Check version compatibility** before upgrading — your Helm charts and CRDs may lag
