---
name: coreweave-inference
description: CoreWeave inference endpoints — vLLM/TGI/Triton on Kubernetes, autoscaling, traffic splitting, A/B testing, health checks, and cost optimization
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

# CoreWeave Inference Endpoints Skill

CoreWeave inference is Kubernetes-native: you deploy standard Deployments/Services,
optionally wrap them with KNative Serving for scale-to-zero, and use Horizontal Pod
Autoscaler (HPA) or KEDA for queue-depth-based scaling. No proprietary inference API.

---

## 1. Architecture Overview

```
Internet → LoadBalancer Service → Ingress (NGINX + TLS)
                                      ↓
                              Kubernetes Deployment
                           (vLLM / TGI / Triton pods)
                                      ↓
                         GPU nodes (A100/H100 + NVLink)
                                      ↓
                    Shared NFS PVC (model weights, read-only)
```

Choose your serving framework:

| Framework | Best For | Protocol | GPU Multi-Instance |
|-----------|----------|----------|--------------------|
| **vLLM** | LLM inference, paged attention, high throughput | OpenAI-compatible REST | Tensor parallel |
| **TGI** (Text Generation Inference) | HuggingFace models, streaming | gRPC + REST | Tensor parallel |
| **Triton** | Multi-model, ONNX/TensorRT, non-LLM | gRPC + REST | Model instances |
| **Custom** | Any framework | Your choice | Your choice |

---

## 2. Serving LLMs with vLLM

### Basic vLLM Deployment (single GPU)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-llama3-8b
  namespace: your-namespace
  labels:
    app: vllm-llama3-8b
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm-llama3-8b
  template:
    metadata:
      labels:
        app: vllm-llama3-8b
        version: v1
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
                    values: ["A100_NVLINK", "RTX_A6000"]
                  - key: topology.kubernetes.io/region
                    operator: In
                    values: ["ORD1"]
      initContainers:
        # Download model weights before starting vLLM
        - name: model-loader
          image: amazon/aws-cli:latest
          command:
            - aws
            - s3
            - sync
            - s3://my-models/llama3-8b/
            - /mnt/weights/llama3-8b/
            - --endpoint-url=https://object.ord1.coreweave.com
          envFrom:
            - secretRef:
                name: coreweave-s3-creds
          volumeMounts:
            - name: model-weights
              mountPath: /mnt/weights
      containers:
        - name: vllm
          image: vllm/vllm-openai:v0.4.2
          command:
            - python3
            - -m
            - vllm.entrypoints.openai.api_server
          args:
            - --model=/mnt/weights/llama3-8b
            - --host=0.0.0.0
            - --port=8000
            - --tensor-parallel-size=1
            - --max-model-len=8192
            - --gpu-memory-utilization=0.90
            - --disable-log-requests       # Reduce log noise in prod
          ports:
            - containerPort: 8000
              name: http
          resources:
            requests:
              nvidia.com/gpu: "1"
              cpu: "8"
              memory: "32Gi"
            limits:
              nvidia.com/gpu: "1"
              cpu: "8"
              memory: "32Gi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 60       # Model load takes time
            periodSeconds: 10
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 120
            periodSeconds: 30
            failureThreshold: 3
          volumeMounts:
            - name: model-weights
              mountPath: /mnt/weights
            - name: dshm
              mountPath: /dev/shm
      volumes:
        - name: model-weights
          persistentVolumeClaim:
            claimName: model-weights       # ReadWriteMany shared NFS
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: "16Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-llama3-8b
  namespace: your-namespace
spec:
  selector:
    app: vllm-llama3-8b
  ports:
    - port: 8000
      targetPort: 8000
      name: http
  type: ClusterIP
```

### Multi-GPU Tensor Parallel (70B models)

```yaml
args:
  - --model=/mnt/weights/llama3-70b
  - --tensor-parallel-size=4          # Shard across 4 GPUs
  - --max-model-len=4096
  - --gpu-memory-utilization=0.95
resources:
  requests:
    nvidia.com/gpu: "4"
  limits:
    nvidia.com/gpu: "4"
```

For 405B models, use 8× H100 80GB:
```yaml
args:
  - --tensor-parallel-size=8
  - --pipeline-parallel-size=2       # vLLM 0.4+ supports pipeline parallelism
resources:
  requests:
    nvidia.com/gpu: "8"
```

---

## 3. Serving with TGI (Text Generation Inference)

```yaml
containers:
  - name: tgi
    image: ghcr.io/huggingface/text-generation-inference:2.0
    command: ["text-generation-launcher"]
    args:
      - --model-id=/mnt/weights/mistral-7b
      - --num-shard=1
      - --port=8080
      - --max-input-length=4096
      - --max-total-tokens=8192
      - --max-batch-prefill-tokens=16384
    ports:
      - containerPort: 8080
        name: http
      - containerPort: 8081
        name: grpc
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 90
      periodSeconds: 10
```

TGI exposes both REST (`/generate`, `/generate_stream`) and gRPC.
It's OpenAI-compatible via `--messages-api-enabled` flag in newer versions.

---

## 4. Serving with NVIDIA Triton

Best for: multi-model serving, TensorRT-optimized models, non-LLM inference.

```yaml
containers:
  - name: triton
    image: nvcr.io/nvidia/tritonserver:24.01-py3
    command: ["tritonserver"]
    args:
      - --model-repository=/mnt/models
      - --strict-model-config=false
      - --log-verbose=0
      - --grpc-port=8001
      - --http-port=8000
      - --metrics-port=8002
    ports:
      - containerPort: 8000    # REST
      - containerPort: 8001    # gRPC
      - containerPort: 8002    # Prometheus metrics
    readinessProbe:
      httpGet:
        path: /v2/health/ready
        port: 8000
      initialDelaySeconds: 30
```

Model repository layout on PVC:
```
/mnt/models/
  my_bert_model/
    config.pbtxt
    1/
      model.onnx
  my_trt_model/
    config.pbtxt
    1/
      model.plan
```

---

## 5. Autoscaling

### HPA on GPU Utilization (via Prometheus + KEDA)

Install KEDA from the CoreWeave Apps catalog, then:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-scaler
  namespace: your-namespace
spec:
  scaleTargetRef:
    name: vllm-llama3-8b
  minReplicaCount: 1              # Never scale to zero (model load time too slow)
  maxReplicaCount: 8
  cooldownPeriod: 300             # 5 min cooldown before scale-down
  pollingInterval: 15
  triggers:
    # Scale on GPU utilization via DCGM metric
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: gpu_utilization
        query: |
          avg(DCGM_FI_DEV_GPU_UTIL{namespace="your-namespace",
              pod=~"vllm-llama3-8b-.*"})
        threshold: "70"           # Scale out when avg GPU util > 70%
    # Or scale on request queue depth
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: vllm_request_queue
        query: |
          sum(vllm:num_requests_waiting{namespace="your-namespace"})
        threshold: "5"            # Scale when >5 requests queued
```

### KNative Serving (scale to zero)

Only use scale-to-zero for low-traffic models where cold start latency is acceptable.
GPU models typically take 60–120 seconds to load — users will see latency spikes.

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: vllm-small
  namespace: your-namespace
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "0"     # Allow scale to zero
        autoscaling.knative.dev/maxScale: "4"
        autoscaling.knative.dev/target: "10"       # Target 10 concurrent requests/pod
        autoscaling.knative.dev/scaleToZeroGracePeriod: "5m"
    spec:
      timeoutSeconds: 600
      containerConcurrency: 20
      containers:
        - image: vllm/vllm-openai:v0.4.2
          # ... same as Deployment spec
```

**Recommendation:** Use `minReplicaCount: 1` (HPA/KEDA) for production endpoints.
Reserve KNative scale-to-zero for dev/staging or low-frequency batch endpoints.

---

## 6. Health Checks

```yaml
readinessProbe:
  httpGet:
    path: /health          # vLLM, TGI: /health — Triton: /v2/health/ready
    port: 8000
  initialDelaySeconds: 90  # Time for model to load into VRAM
  periodSeconds: 10
  successThreshold: 1
  failureThreshold: 6       # Allow 60s of unhealthy before eviction

livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 180  # Give ample time for large models
  periodSeconds: 30
  failureThreshold: 3

startupProbe:              # Prefer startupProbe for slow-starting pods
  httpGet:
    path: /health
    port: 8000
  failureThreshold: 60     # 60 × 10s = 10 minutes max startup time
  periodSeconds: 10
```

> Use `startupProbe` instead of `initialDelaySeconds` — it disables liveness/readiness
> checks until the first success, preventing premature restarts during model loading.

---

## 7. Traffic Splitting and A/B Testing

Use a top-level Service with pod label selectors, or an Ingress with NGINX
traffic splitting:

### Label-based canary (simple)

```yaml
# Production deployment: label version=v1
# Canary deployment: label version=v2, replicas=1

# Service routes to both (80/20 by replica count)
apiVersion: v1
kind: Service
metadata:
  name: inference-canary
spec:
  selector:
    app: vllm-llama3        # Matches both v1 and v2 pods
  ports:
    - port: 8000
```

### NGINX Ingress traffic split (precise percentages)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: inference-split
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"   # 20% to canary
spec:
  ingressClassName: public
  rules:
    - host: inference.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vllm-v2-canary      # 20% traffic
                port:
                  number: 8000
---
# Main ingress (80% traffic) — no canary annotation
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: inference-main
spec:
  ingressClassName: public
  rules:
    - host: inference.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vllm-v1-stable     # 80% traffic
                port:
                  number: 8000
```

Header-based routing (route specific users to canary):
```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-by-header: "X-Model-Version"
  nginx.ingress.kubernetes.io/canary-by-header-value: "v2"
```

---

## 8. Persistent Model Storage Strategy

Load models once into a shared NFS PVC, mount read-only across all replicas:

```yaml
# Download job — runs once
apiVersion: batch/v1
kind: Job
metadata:
  name: model-download
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: downloader
          image: huggingface/transformers:latest
          command:
            - python3
            - -c
            - |
              from huggingface_hub import snapshot_download
              snapshot_download(
                repo_id="meta-llama/Meta-Llama-3-70B-Instruct",
                local_dir="/mnt/weights/llama3-70b",
                token="hf_...",
                ignore_patterns=["*.msgpack", "flax*"]
              )
          volumeMounts:
            - name: model-weights
              mountPath: /mnt/weights
      volumes:
        - name: model-weights
          persistentVolumeClaim:
            claimName: model-weights      # ReadWriteMany
```

Mount as read-only in inference pods:
```yaml
volumeMounts:
  - name: model-weights
    mountPath: /mnt/weights
    readOnly: true                       # Prevents accidental corruption
```

---

## 9. Cost Optimization for Inference

### Right-size GPU to model

| Model Size | Recommended GPU | Reason |
|------------|----------------|--------|
| ≤7B (FP16) | RTX A6000 (48 GB) | 14 GB model, headroom for KV cache |
| 13B (FP16) | A40 or A6000 | ~26 GB model |
| 70B (FP16) | 2× A100 80GB | ~140 GB total |
| 70B (AWQ int4) | A100 40GB | ~35 GB quantized |
| 405B (FP8) | 8× H100 80GB | ~200 GB FP8 |

### Use quantization

vLLM supports AWQ, GPTQ, and FP8 out of the box:
```yaml
args:
  - --quantization=awq          # Load pre-quantized AWQ model
  # or
  - --quantization=fp8          # H100 native FP8 (nearly lossless)
  - --dtype=float16
```

FP8 on H100 delivers ~2× throughput vs FP16 with minimal quality loss.

### Batching and throughput

```yaml
args:
  - --max-num-batched-tokens=32768    # Larger = better GPU utilization
  - --max-num-seqs=256                # Concurrent sequences
  - --enable-chunked-prefill          # Better latency for mixed requests
```

### Multi-model on one GPU (MIG or time-sharing)

For small models (≤10B, quantized), use MIG slices (see GPU skill) to run
multiple models on one H100:

```
H100 80GB → 7× mig-1g.10gb slices
  → Run 7 different fine-tuned 7B models simultaneously
  → Cost: 1× H100 instead of 7× A6000
```

---

## 10. Observability

```bash
# Watch inference pod GPU utilization live
kubectl exec -it <vllm-pod> -- watch -n1 nvidia-smi

# vLLM exposes Prometheus metrics at /metrics
kubectl port-forward svc/vllm-llama3-8b 8000:8000
curl http://localhost:8000/metrics | grep vllm

# Key vLLM metrics:
#   vllm:num_requests_running        — active requests
#   vllm:num_requests_waiting        — queued requests (use for HPA)
#   vllm:gpu_cache_usage_perc        — KV cache utilization
#   vllm:avg_generation_throughput   — tokens/second

# TGI metrics at /metrics (Prometheus format)
# Triton metrics at :8002/metrics
```

---

## 11. Guardrails & Gotchas

- **`initialDelaySeconds` is not enough** — large models take 2–5 minutes to load. Always use `startupProbe` with `failureThreshold: 60` and `periodSeconds: 10`.
- **KV cache OOM is silent** — if `--gpu-memory-utilization=0.95` leaves no room for KV cache, vLLM errors at request time, not startup. Start at 0.85 and tune up.
- **Tensor parallel = 1 pod** — you cannot distribute one tensor-parallel group across multiple Kubernetes pods. All shards must be in one pod (multiple GPUs via `nvidia.com/gpu: N`).
- **Pipeline parallel** is supported in vLLM 0.4+ but experimental — prefer tensor parallel for <16 GPUs.
- **NFS latency for model loads** — shared NFS can add 2–5 min to model load vs local NVMe. For latency-sensitive startups, copy model to a block PVC first.
- **vLLM `/health` returns 200 before model is ready** in some versions — test `/v1/models` instead for a more accurate readiness signal.
- **Canary weights are approximate** — NGINX distributes by connection, not exact percentage. Use at least 10 replicas for accurate splits.
- **Scale-down leaves idle GPU pods running** — set `cooldownPeriod` to at least 5 minutes to avoid thrashing on bursty traffic.
- **Object storage for model weights is slow** — downloading a 70B model at runtime adds 15–30 minutes. Always pre-download into a PVC via a one-time Job.
- **`ReadWriteMany` PVC is NFS** — no POSIX file locking. vLLM and TGI are safe; SQLite-based checkpointing is not.
