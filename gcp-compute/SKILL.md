---
name: gcp-compute
description: GCP compute — Compute Engine, GKE (Autopilot/Standard), Cloud Run, Cloud Functions, App Engine, Batch
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP Compute Skill

Choose and operate the right compute primitive for your workload.

---

## Compute Engine (VMs)

### Machine Types

| Family | Use Case | Examples |
|--------|---------|---------|
| `e2` | Cost-optimized general purpose | e2-micro, e2-standard-4 |
| `n2` | Balanced general purpose | n2-standard-4, n2-highmem-8 |
| `n2d` | AMD EPYC, better price/perf | n2d-standard-8 |
| `c2` | Compute-optimized (3.8 GHz) | c2-standard-4 |
| `m1`/`m2`/`m3` | Memory-optimized | m2-ultramem-208 |
| `a2` | GPU (NVIDIA A100) | a2-highgpu-1g |
| `custom` | Custom vCPU/RAM ratio | custom-6-20480 |

```bash
# Create a VM
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --machine-type=n2-standard-2 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB \
  --boot-disk-type=pd-ssd \
  --network=my-vpc \
  --subnet=my-subnet \
  --no-address \
  --service-account=my-sa@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --tags=ssh-allowed,web-server \
  --metadata=startup-script='#!/bin/bash
apt-get update && apt-get install -y nginx'

# SSH to a VM (uses IAP automatically if no external IP)
gcloud compute ssh my-vm --zone=us-central1-a

# SSH tunnel for a service port
gcloud compute ssh my-vm --zone=us-central1-a \
  --tunnel-through-iap \
  -- -L 8080:localhost:8080

# List instances
gcloud compute instances list

# Start/stop/delete
gcloud compute instances start my-vm --zone=us-central1-a
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute instances delete my-vm --zone=us-central1-a --quiet

# Resize machine type (requires stop first)
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute instances set-machine-type my-vm \
  --zone=us-central1-a \
  --machine-type=n2-standard-4
gcloud compute instances start my-vm --zone=us-central1-a
```

### Preemptible / Spot VMs

```bash
# Preemptible (fixed 24h max, up to 91% discount)
gcloud compute instances create my-vm \
  --preemptible \
  --machine-type=n2-standard-4 \
  --zone=us-central1-a \
  [other flags...]

# Spot (no fixed max lifetime, can be reclaimed anytime, same discount)
gcloud compute instances create my-vm \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --machine-type=n2-standard-4 \
  --zone=us-central1-a \
  [other flags...]
```

**Use cases:** Batch jobs, CI workers, fault-tolerant workloads. Never use for
databases or stateful services.

### Custom Images

```bash
# Create image from a disk
gcloud compute images create my-golden-image \
  --source-disk=my-vm \
  --source-disk-zone=us-central1-a \
  --family=my-app \
  --labels=version=v1-2-0

# List images in a family
gcloud compute images list --filter="family=my-app"

# Use image in a new VM
gcloud compute instances create new-vm \
  --image-family=my-app \
  --image-project=my-project \
  --zone=us-central1-a
```

### Managed Instance Groups (MIGs)

```bash
# Create an instance template
gcloud compute instance-templates create web-template \
  --machine-type=n2-standard-2 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=web-server \
  --metadata=startup-script-url=gs://my-bucket/startup.sh \
  --service-account=web-sa@my-project.iam.gserviceaccount.com \
  --scopes=cloud-platform

# Create a MIG
gcloud compute instance-groups managed create web-mig \
  --base-instance-name=web \
  --template=web-template \
  --size=3 \
  --zone=us-central1-a

# Enable autoscaling
gcloud compute instance-groups managed set-autoscaling web-mig \
  --zone=us-central1-a \
  --min-num-replicas=2 \
  --max-num-replicas=10 \
  --target-cpu-utilization=0.6 \
  --cool-down-period=90

# Rolling update to a new template
gcloud compute instance-groups managed rolling-action start-update web-mig \
  --zone=us-central1-a \
  --version=template=web-template-v2 \
  --max-surge=2 \
  --max-unavailable=0

# Check update status
gcloud compute instance-groups managed describe web-mig --zone=us-central1-a

# Regional MIG (multi-zone, recommended for production)
gcloud compute instance-groups managed create web-mig \
  --base-instance-name=web \
  --template=web-template \
  --size=3 \
  --region=us-central1
```

---

## GKE (Google Kubernetes Engine)

### Autopilot vs Standard

| | Autopilot | Standard |
|--|-----------|---------|
| Node management | Google manages nodes | You manage nodes |
| Billing | Per Pod CPU/memory | Per node |
| Node pools | No | Yes |
| DaemonSets | Limited | Full |
| Best for | Most workloads | Custom node configs, GPU, Windows |

### Create Clusters

```bash
# Autopilot cluster (recommended for most workloads)
gcloud container clusters create-auto my-cluster \
  --region=us-central1 \
  --network=my-vpc \
  --subnetwork=my-subnet \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28

# Standard cluster
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=n2-standard-4 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --network=my-vpc \
  --subnetwork=my-subnet \
  --enable-private-nodes \
  --master-ipv4-cidr=172.16.0.0/28 \
  --workload-pool=my-project.svc.id.goog \
  --enable-ip-alias \
  --cluster-secondary-range-name=pods \
  --services-secondary-range-name=services

# Get credentials
gcloud container clusters get-credentials my-cluster \
  --zone=us-central1-a
```

### Node Pools

```bash
# Add a node pool (e.g., GPU pool)
gcloud container node-pools create gpu-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --machine-type=n1-standard-4 \
  --accelerator=type=nvidia-tesla-t4,count=1 \
  --num-nodes=1 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=4 \
  --node-labels=workload=gpu

# Add a Spot VM node pool (cost savings)
gcloud container node-pools create spot-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --machine-type=n2-standard-4 \
  --spot \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=20
```

### Workload Identity (GKE → GCP APIs)

The right way for Pods to authenticate to GCP — no SA keys.

```bash
# 1. Create a GCP SA
gcloud iam service-accounts create app-sa --project=my-project

# 2. Grant GCP SA permission to GCP APIs
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:app-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# 3. Allow Kubernetes SA to impersonate GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  app-sa@my-project.iam.gserviceaccount.com \
  --member="serviceAccount:my-project.svc.id.goog[my-namespace/my-ksa]" \
  --role="roles/iam.workloadIdentityUser"

# 4. Annotate the Kubernetes SA
kubectl annotate serviceaccount my-ksa \
  --namespace=my-namespace \
  iam.gke.io/gcp-service-account=app-sa@my-project.iam.gserviceaccount.com
```

---

## Cloud Run

Fully managed, serverless containers. Pay per request.

```bash
# Deploy a container
gcloud run deploy my-service \
  --image=us-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated \
  --port=8080 \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=10 \
  --concurrency=80 \
  --set-env-vars=ENV=production,DB_HOST=10.0.0.1

# Deploy from source (builds automatically)
gcloud run deploy my-service \
  --source=. \
  --region=us-central1

# Deploy with VPC connector (for private resources)
gcloud run deploy my-service \
  --image=... \
  --vpc-connector=my-connector \
  --vpc-egress=all-traffic \
  --region=us-central1

# Set traffic splits (canary / blue-green)
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-00002-abc=80,my-service-00001-xyz=20

# List services
gcloud run services list --region=us-central1

# Get service URL
gcloud run services describe my-service \
  --region=us-central1 \
  --format="value(status.url)"

# View logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=my-service" \
  --limit=50 --format=json
```

### Cloud Run Jobs

```bash
# Create a job
gcloud run jobs create my-job \
  --image=us-docker.pkg.dev/my-project/my-repo/my-job:latest \
  --region=us-central1 \
  --tasks=10 \
  --parallelism=5 \
  --max-retries=3 \
  --memory=2Gi \
  --cpu=2

# Execute a job
gcloud run jobs execute my-job --region=us-central1

# Execute and wait for completion
gcloud run jobs execute my-job --region=us-central1 --wait
```

---

## Cloud Functions

### 2nd Gen (recommended — based on Cloud Run)

```bash
# Deploy (HTTP trigger)
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python311 \
  --trigger-http \
  --allow-unauthenticated \
  --region=us-central1 \
  --source=. \
  --entry-point=handle_request \
  --memory=256MB \
  --timeout=60s \
  --set-env-vars=MY_VAR=value

# Pub/Sub trigger
gcloud functions deploy my-processor \
  --gen2 \
  --runtime=python311 \
  --trigger-topic=my-topic \
  --region=us-central1 \
  --source=. \
  --entry-point=process_message

# GCS trigger
gcloud functions deploy my-handler \
  --gen2 \
  --runtime=nodejs20 \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=my-bucket" \
  --region=us-central1 \
  --source=. \
  --entry-point=handleUpload

# Invoke a function
gcloud functions call my-function \
  --gen2 \
  --region=us-central1 \
  --data='{"name":"test"}'

# View logs
gcloud functions logs read my-function --gen2 --region=us-central1
```

---

## App Engine

### Standard vs Flexible

| | Standard | Flexible |
|--|----------|---------|
| Scale to zero | Yes | No (min 1 instance) |
| Cold starts | Yes | No |
| Runtime | Sandbox (Python, Java, Go, Node.js, PHP, Ruby) | Any Docker container |
| Max request timeout | 10 min | 60 min |

```bash
# Deploy
gcloud app deploy app.yaml

# View logs
gcloud app logs tail -s default

# Browse
gcloud app browse

# List versions
gcloud app versions list

# Migrate traffic
gcloud app services set-traffic default \
  --splits=v1=0.8,v2=0.2 \
  --split-by=random
```

---

## Cloud Batch

For large-scale batch jobs on VMs.

```bash
# Submit a batch job via JSON spec
gcloud batch jobs submit my-job \
  --location=us-central1 \
  --config=job.json

# List jobs
gcloud batch jobs list --location=us-central1

# Describe a job
gcloud batch jobs describe my-job --location=us-central1
```

---

## Guardrails

- **Disable default SA on VMs.** Create purpose-specific SAs with minimal permissions.
- **Don't use `--scopes=cloud-platform` without a restricted SA** — it grants all APIs.
- **GKE: enable Workload Identity** — never mount SA JSON keys into Pods.
- **Cloud Run: set `--min-instances=1`** for latency-sensitive services to avoid cold starts.
- **Cloud Run concurrency default is 80** — tune per your app's thread model.
- **Preemptible/Spot VMs can be reclaimed in 30 seconds** — always checkpoint state.
- **MIG rolling updates with `--max-unavailable=0`** prevent downtime but require surge capacity.
- **Autopilot pods must have resource requests set** — Autopilot enforces them strictly.
