---
name: coreweave-k8s
description: CoreWeave Kubernetes platform — kubeconfig, namespaces, storage, networking, node affinity, and the CoreWeave Apps catalog
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

# CoreWeave Kubernetes Skill

CoreWeave is a Kubernetes-native GPU cloud. Every workload — inference endpoints,
training jobs, batch pipelines — runs as a standard Kubernetes resource. There is
no proprietary VM abstraction; you interact directly with the Kubernetes API.

---

## 1. Kubeconfig Setup

CoreWeave issues per-namespace kubeconfig files from the Cloud UI
(`cloud.coreweave.com → API Access`). Download the file and merge it:

```bash
# Merge into existing kubeconfig
KUBECONFIG=~/.kube/config:~/Downloads/coreweave-kubeconfig.yaml \
  kubectl config view --flatten > ~/.kube/config-merged
mv ~/.kube/config-merged ~/.kube/config

# Or set for session only
export KUBECONFIG=~/Downloads/coreweave-kubeconfig.yaml

# Verify
kubectl cluster-info
kubectl get nodes -o wide
```

**Context name** is usually `coreweave` or your tenant slug. CoreWeave namespaces
map 1:1 to your organization unit — you cannot cross-namespace unless you own both.

```bash
# List available namespaces (only yours are visible)
kubectl get namespaces

# Set default namespace for session
kubectl config set-context --current --namespace=<your-namespace>
```

---

## 2. CoreWeave Regions

CoreWeave data centers are referenced as **region labels** on nodes.

| Region Code    | Location              | Notes                          |
|----------------|-----------------------|--------------------------------|
| `ORD1`         | Chicago (US-CENTRAL)  | US-CENTRAL-02A in docs         |
| `LAS1`         | Las Vegas             | GPU-heavy region               |
| `EWR1`         | New Jersey (US-EAST)  | US-EAST-04A in docs            |
| `BKK1`         | Bangkok               | APAC expansion                 |
| `LGA1`         | New York area         | Legacy; prefer EWR1            |

**Pin workloads to a region** using node affinity (see §6).

Check which region your nodes live in:
```bash
kubectl get nodes -L topology.kubernetes.io/region
```

---

## 3. Namespaces and RBAC

CoreWeave tenants get one or more namespaces. RBAC is managed in the Cloud UI
or via standard Kubernetes RBAC objects.

```yaml
# ServiceAccount for a workload
apiVersion: v1
kind: ServiceAccount
metadata:
  name: training-runner
  namespace: your-namespace
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: your-namespace
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: training-runner-pod-reader
  namespace: your-namespace
subjects:
  - kind: ServiceAccount
    name: training-runner
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 4. CoreWeave-Specific Annotations

CoreWeave extends Kubernetes with annotations for billing, scheduling, and features.

```yaml
metadata:
  annotations:
    # Billing labels (show up on invoice line items)
    billing.coreweave.com/project: "llm-training"
    billing.coreweave.com/team: "ml-platform"

    # Prevent pods from being evicted during node maintenance
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"

    # CoreWeave networking: attach to specific VPC / tenant network
    # (set at namespace level usually, but can annotate pods)
    networking.coreweave.com/subnet: "10.135.0.0/24"
```

Node labels used for scheduling (read-only; set by CoreWeave):
```
gpu.nvidia.com/class=A100_NVLINK    # GPU model class
topology.kubernetes.io/region=ORD1  # Data center
node.coreweave.cloud/cpu=amd        # CPU vendor
node.coreweave.cloud/class=gpu      # Node class
```

---

## 5. Storage

CoreWeave offers three storage tiers, all Kubernetes-native.

### 5a. Shared Filesystem (ReadWriteMany NFS-backed)

Best for: model weights shared across many pods, dataset staging.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-weights
  namespace: your-namespace
spec:
  accessModes:
    - ReadWriteMany          # Multiple pods can mount simultaneously
  resources:
    requests:
      storage: 500Gi
  storageClassName: shared-hdd-ord1    # HDD-backed, cheaper
  # storageClassName: shared-nvme-ord1 # NVMe-backed, faster
```

Storage classes (vary by region; check `kubectl get storageclasses`):
- `shared-hdd-<region>` — ~$0.03/GiB/month, good for cold model storage
- `shared-nvme-<region>` — ~$0.10/GiB/month, low-latency checkpointing

### 5b. Block Storage (ReadWriteOnce)

Best for: databases, single-pod scratch space, OS volumes.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-scratch
  namespace: your-namespace
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Ti
  storageClassName: block-nvme-ord1
```

### 5c. Object Storage (S3-compatible)

CoreWeave Object Storage exposes an S3-compatible API. No Kubernetes PVC needed —
access via standard S3 SDKs/tools with CoreWeave-issued credentials.

```bash
# Configure AWS CLI to point at CoreWeave Object Storage
aws configure --profile coreweave
# AWS Access Key ID: <from Cloud UI>
# AWS Secret Access Key: <from Cloud UI>
# Default region: us-east-1  (use us-east-1 regardless of data center)
# Default output format: json

# Set endpoint override
export AWS_ENDPOINT_URL=https://object.ord1.coreweave.com

# List buckets
aws s3 ls --profile coreweave

# Sync model weights
aws s3 sync s3://my-bucket/llama-70b /mnt/weights/ --profile coreweave
```

Store S3 credentials as a Kubernetes Secret for pods:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: coreweave-s3-creds
  namespace: your-namespace
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "AKIAIOSFODNN7EXAMPLE"
  AWS_SECRET_ACCESS_KEY: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  AWS_ENDPOINT_URL: "https://object.ord1.coreweave.com"
```

---

## 6. Node Affinity for GPU Types

**Always use `nodeAffinity` + `tolerations`** — CoreWeave GPU nodes have taints.

```yaml
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
                values:
                  - A100_NVLINK     # Prefer: A100_NVLINK, H100_NVLINK_80GB
              - key: topology.kubernetes.io/region
                operator: In
                values:
                  - ORD1
```

---

## 7. Networking

### Internal ClusterIP Services

Pod-to-pod within namespace: standard Kubernetes ClusterIP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: inference-backend
  namespace: your-namespace
spec:
  selector:
    app: inference
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

### External LoadBalancer

CoreWeave allocates a real public IP automatically:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: inference-public
  namespace: your-namespace
  annotations:
    # Optional: request a specific IP family
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
spec:
  selector:
    app: inference
  ports:
    - port: 443
      targetPort: 8080
  type: LoadBalancer
```

Check assigned IP: `kubectl get svc inference-public -w`

### Ingress (HTTPS termination)

CoreWeave supports NGINX ingress with automatic cert-manager TLS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: inference-ingress
  namespace: your-namespace
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  ingressClassName: public
  tls:
    - hosts:
        - inference.mycompany.com
      secretName: inference-tls
  rules:
    - host: inference.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: inference-public
                port:
                  number: 8080
```

---

## 8. CoreWeave Apps Catalog

CoreWeave provides a Helm-based Apps catalog (similar to ArtifactHub) accessible
via `cloud.coreweave.com → Apps`. Common catalog apps:

| App | Use case |
|-----|----------|
| Argo Workflows | ML pipeline orchestration |
| Jupyter Hub | Interactive notebooks on GPU nodes |
| MLflow | Experiment tracking |
| Grafana + Prometheus | GPU/cluster metrics |
| DCGM Exporter | Per-GPU Prometheus metrics |
| Volcano | Batch scheduling for ML jobs |

Install via Helm (CoreWeave repo):
```bash
helm repo add coreweave https://helm.coreweave.com
helm repo update
helm search repo coreweave
helm install my-jupyter coreweave/jupyter --namespace your-namespace \
  --set gpu.type=RTX_A6000 \
  --set storage.pvc.name=model-weights
```

---

## 9. Common kubectl Patterns

```bash
# Watch pod startup (GPU pods can take 2-5 min to pull images)
kubectl get pods -w

# Stream logs from training job
kubectl logs -f job/training-job --all-containers

# Describe node to see GPU capacity
kubectl describe node <node-name> | grep -A5 "Capacity"

# Execute into a running GPU pod
kubectl exec -it <pod-name> -- bash

# Check GPU allocation across namespace
kubectl describe nodes | grep -E "nvidia.com/gpu|Name:"

# Delete all completed/failed pods (cleanup)
kubectl delete pods --field-selector=status.phase=Succeeded
kubectl delete pods --field-selector=status.phase=Failed

# Port-forward a service locally
kubectl port-forward svc/inference-backend 8080:8080
```

---

## 10. Guardrails & Gotchas

- **GPU nodes are tainted** — always add the `nvidia.com/gpu` toleration or pods stay Pending forever.
- **Images take time** — large PyTorch images (20–30 GB) can take 5–10 min to pull on first schedule. Use `imagePullPolicy: IfNotPresent` and pre-warm with a DaemonSet or CoreWeave's image caching.
- **PVCs are zonal** — a `block-nvme-ord1` PVC can only mount to a pod in ORD1. Match PVC region to node affinity region.
- **Shared PVCs are eventually consistent** — don't use shared NFS for SQLite or any file-locking workload.
- **Object storage egress costs money** — keep training data in the same region as compute.
- **LoadBalancer IPs persist** across pod restarts but not PVC deletions — annotate the Service with `metallb.universe.tf/allow-shared-ip` if you need stable IPs.
- **Billing labels are free** — always set `billing.coreweave.com/project` for cost attribution.
- **Context deadline exceeded** on `kubectl apply` — CoreWeave's API server has a 30s timeout; break large manifests into batches.
