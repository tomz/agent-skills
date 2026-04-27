---
name: oci-compute
description: OCI Compute — instances, shapes, OKE, Container Instances, Functions, and Autoscaling
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

# OCI Compute

## Instance Shapes

OCI shapes define CPU, memory, and network for compute instances.

### Shape Families
| Family | Use Case | Notes |
|--------|----------|-------|
| `VM.Standard.E4.Flex` | General purpose (AMD EPYC) | Most cost-effective flex shape |
| `VM.Standard.E5.Flex` | General purpose (AMD EPYC 4th gen) | Newer, higher performance |
| `VM.Standard.A1.Flex` | Arm (Ampere Altra) | Cheapest per OCPU — Always Free eligible |
| `VM.Standard3.Flex` | General purpose (Intel) | Intel Xeon |
| `VM.GPU.A10.1` | ML/AI inference | NVIDIA A10, 24 GB VRAM |
| `VM.GPU.A100.2` | ML/AI training | 2x NVIDIA A100 80 GB |
| `BM.GPU.A100-v2.8` | Large-scale training | 8x A100, bare metal |
| `VM.DenseIO2.16` | NVMe storage | Local NVMe, fast I/O |
| `BM.HPC2.36` | HPC | RDMA networking, bare metal |
| `VM.Optimized3.Flex` | High-frequency compute | Intel, higher clock speed |

### Flex Shape Config
```bash
# List available shapes with OCPU/memory limits
oci compute shape list --compartment-id $C --all | \
  jq '.data[] | select(.shape | startswith("VM.Standard")) | {shape, ocpuOptions, memoryOptions}'

# Launch flex instance
oci compute instance launch \
  --compartment-id $C \
  --availability-domain "kWVD:US-ASHBURN-AD-1" \
  --shape "VM.Standard.E4.Flex" \
  --shape-config '{"ocpus":4,"memoryInGBs":32}' \
  --image-id $IMAGE_ID \
  --subnet-id $SUBNET_ID \
  --display-name "flex-vm" \
  --assign-public-ip true
```

## Finding Images

```bash
# List Oracle platform images
oci compute image list --compartment-id $C \
  --operating-system "Oracle Linux" \
  --operating-system-version "9" \
  --shape "VM.Standard.E4.Flex" \
  --sort-by TIMECREATED --sort-order DESC \
  --limit 5 --output table

# List Ubuntu images
oci compute image list --compartment-id $C \
  --operating-system "Canonical Ubuntu" \
  --operating-system-version "22.04" \
  --shape "VM.Standard.E4.Flex" \
  --sort-by TIMECREATED --sort-order DESC --limit 3

# Get the latest image OCID
LATEST_IMAGE=$(oci compute image list --compartment-id $C \
  --operating-system "Oracle Linux" \
  --operating-system-version "9" \
  --sort-by TIMECREATED --sort-order DESC \
  --query 'data[0].id' --raw-output)
```

## Cloud-Init / User Data

```yaml
# cloud-init.yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - nginx
  - git
  - python3-pip

write_files:
  - path: /etc/nginx/conf.d/app.conf
    content: |
      server {
        listen 80;
        location / { proxy_pass http://localhost:8080; }
      }

runcmd:
  - systemctl enable nginx
  - systemctl start nginx
  - firewall-cmd --add-service=http --permanent
  - firewall-cmd --reload

users:
  - name: deploy
    groups: wheel
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ssh-rsa AAAA...
```

```bash
# Pass cloud-init to instance
oci compute instance launch \
  --user-data "$(base64 -w0 cloud-init.yaml)" \
  ... (other flags)
```

## Boot Volumes

```bash
# Increase boot volume size (must stop instance first)
oci compute instance action --instance-id $INST_ID --action STOP
oci bv boot-volume update --boot-volume-id $BV_ID --size-in-gbs 200

# Create custom image from instance
oci compute image create \
  --compartment-id $C \
  --instance-id $INST_ID \
  --display-name "my-golden-image-$(date +%Y%m%d)"

# Clone boot volume
oci bv boot-volume create \
  --availability-domain "kWVD:US-ASHBURN-AD-1" \
  --compartment-id $C \
  --source-details '{"type":"bootVolume","id":"'$BV_ID'"}'
```

## Instance Metadata Service (IMDS)

From within an OCI instance:
```bash
# Get instance metadata (v2 — requires auth header)
curl -H "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/

# Get instance OCID
curl -sH "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq -r '.id'

# Get compartment OCID
curl -sH "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/ | jq -r '.compartmentId'

# Get user data
curl -sH "Authorization: Bearer Oracle" \
  http://169.254.169.254/opc/v2/instance/metadata/
```

## OKE — Oracle Kubernetes Engine

### Create Cluster
```bash
# Create enhanced cluster (recommended — supports virtual nodes, add-ons)
oci ce cluster create \
  --compartment-id $C \
  --name "prod-k8s" \
  --kubernetes-version "v1.29.1" \
  --vcn-id $VCN_ID \
  --type "ENHANCED_CLUSTER" \
  --endpoint-config '{"isPublicIpEnabled":true,"subnetId":"'$PUBLIC_SUBNET_ID'"}' \
  --cluster-pod-network-options '[{"cniType":"OCI_VCN_IP_NATIVE"}]'

# List available Kubernetes versions
oci ce cluster-options get --cluster-option-id all \
  --query 'data."kubernetes-versions"'
```

### Node Pools
```bash
# Create managed node pool
oci ce node-pool create \
  --compartment-id $C \
  --cluster-id $CLUSTER_ID \
  --name "workers" \
  --kubernetes-version "v1.29.1" \
  --node-shape "VM.Standard.E4.Flex" \
  --node-shape-config '{"ocpus":4,"memoryInGBs":32}' \
  --node-image-name "Oracle-Linux-8.9-aarch64-2024.03.22-0" \
  --node-config-details '{
    "size": 3,
    "placementConfigs": [
      {"availabilityDomain":"kWVD:US-ASHBURN-AD-1","subnetId":"'$WORKER_SUBNET_ID'"},
      {"availabilityDomain":"kWVD:US-ASHBURN-AD-2","subnetId":"'$WORKER_SUBNET_ID'"},
      {"availabilityDomain":"kWVD:US-ASHBURN-AD-3","subnetId":"'$WORKER_SUBNET_ID'"}
    ]
  }'

# Virtual node pool (serverless — no worker node VMs to manage)
oci ce virtual-node-pool create \
  --compartment-id $C \
  --cluster-id $CLUSTER_ID \
  --display-name "virtual-workers" \
  --kubernetes-version "v1.29.1" \
  --pod-configuration '{"shape":"Pod.Standard.E4.Flex","subnetId":"'$POD_SUBNET_ID'"}' \
  --placement-configurations '[{"availabilityDomain":"kWVD:US-ASHBURN-AD-1","subnetId":"'$VN_SUBNET_ID'"}]' \
  --size 10
```

### Kubeconfig
```bash
# Generate kubeconfig
oci ce cluster create-kubeconfig \
  --cluster-id $CLUSTER_ID \
  --file ~/.kube/config \
  --region us-ashburn-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT

# Verify
kubectl get nodes
kubectl get pods --all-namespaces
```

### OKE Add-ons (Enhanced Clusters)
```bash
# List available add-ons
oci ce addon-option list --kubernetes-version "v1.29.1"

# Install CoreDNS, kube-proxy add-ons are automatic
# Install Cluster Autoscaler
oci ce addon install \
  --addon-name "ClusterAutoscaler" \
  --cluster-id $CLUSTER_ID \
  --configurations '[{"key":"nodePool0Id","value":"'$NODEPOOL_ID'"},{"key":"nodePool0MinSize","value":"1"},{"key":"nodePool0MaxSize","value":"10"}]'
```

## Container Instances

Run containers without managing VMs — serverless containers.

```bash
oci container-instances container-instance create \
  --compartment-id $C \
  --availability-domain "kWVD:US-ASHBURN-AD-1" \
  --display-name "my-app" \
  --shape "CI.Standard.E4.Flex" \
  --shape-config '{"ocpus":2,"memoryInGBs":8}' \
  --containers '[{
    "displayName": "app",
    "imageUrl": "iad.ocir.io/mytenancy/myrepo/myapp:latest",
    "resourceConfig": {"vcpusLimit":2,"memoryLimitInGBs":8},
    "environmentVariables": {"ENV":"prod","PORT":"8080"}
  }]' \
  --vnics '[{
    "subnetId": "'$SUBNET_ID'",
    "isPublicIpAssigned": false
  }]' \
  --image-pull-secrets '[{
    "registryEndpoint": "iad.ocir.io",
    "username": "'$OCIR_USER'",
    "password": "'$OCIR_TOKEN'",
    "secretType": "BASIC"
  }]'
```

## Functions (OCI Functions / Fn Project)

```bash
# Install Fn CLI
curl -LSs https://raw.githubusercontent.com/fnproject/cli/master/install | sh

# Configure Fn to use OCI
fn create context oci --provider oracle
fn use context oci
fn update context api-url https://functions.us-ashburn-1.oraclecloud.com
fn update context oracle.compartment-id $C
fn update context registry iad.ocir.io/$TENANCY_NAMESPACE/functions

# Create application
oci fn application create \
  --compartment-id $C \
  --display-name "my-app" \
  --subnet-ids "[\"$SUBNET_ID\"]"

# Deploy function (from directory with func.yaml)
fn --verbose deploy --app my-app

# Invoke function
fn invoke my-app my-function '{"key":"value"}'

# Via OCI CLI
oci fn function invoke \
  --function-id $FUNCTION_ID \
  --file "-" \
  --body '{"name":"world"}'
```

## Autoscaling

### Instance Pools + Autoscaling
```bash
# 1. Create instance configuration (template)
INSTANCE_CONFIG_ID=$(oci compute-management instance-configuration create \
  --compartment-id $C \
  --display-name "web-config" \
  --instance-details '{
    "instanceType": "compute",
    "launchDetails": {
      "compartmentId": "'$C'",
      "shape": "VM.Standard.E4.Flex",
      "shapeConfig": {"ocpus":2,"memoryInGBs":16},
      "sourceDetails": {"sourceType":"image","imageId":"'$IMAGE_ID'"},
      "createVnicDetails": {"subnetId":"'$SUBNET_ID'","assignPublicIp":false}
    }
  }' --query 'data.id' --raw-output)

# 2. Create instance pool
POOL_ID=$(oci compute-management instance-pool create \
  --compartment-id $C \
  --display-name "web-pool" \
  --instance-configuration-id $INSTANCE_CONFIG_ID \
  --size 2 \
  --placement-configurations '[{
    "availabilityDomain": "kWVD:US-ASHBURN-AD-1",
    "primarySubnetId": "'$SUBNET_ID'"
  }]' --query 'data.id' --raw-output)

# 3. Create autoscaling config
oci autoscaling auto-scaling-configuration create \
  --compartment-id $C \
  --display-name "web-autoscaler" \
  --resource '{"id":"'$POOL_ID'","type":"instancePool"}' \
  --policies '[{
    "policyType": "threshold",
    "displayName": "scale-out-on-cpu",
    "capacity": {"initial":2,"min":1,"max":10},
    "rules": [
      {
        "action": {"type":"CHANGE_COUNT_BY","value":1},
        "displayName": "scale-out",
        "metric": {"metricType":"CPU_UTILIZATION","threshold":{"operator":"GT","value":70}}
      },
      {
        "action": {"type":"CHANGE_COUNT_BY","value":-1},
        "displayName": "scale-in",
        "metric": {"metricType":"CPU_UTILIZATION","threshold":{"operator":"LT","value":30}}
      }
    ]
  }]'
```

## Gotchas & Tips

- **Flex shapes** — `ocpus` and `memoryInGBs` must be within shape limits. Memory-to-CPU ratio must stay between 1–64 GB per OCPU for E4.Flex.
- **Always Free A1** — 4 OCPUs / 24 GB RAM total across all A1 instances in a tenancy (in Always Free). Very generous for hobbyists.
- **OKE VCN-native pods** — `OCI_VCN_IP_NATIVE` CNI assigns pod IPs from VCN CIDR, enabling direct routing without overlay. Requires pod subnet CIDR planning upfront.
- **OKE cluster endpoint** — use `PRIVATE_ENDPOINT` for production (bastion or VPN required for kubectl access). `PUBLIC_ENDPOINT` is convenient but exposes API server.
- **Functions cold start** — Fn functions cold start takes 3–10 seconds. Use provisioned concurrency for latency-sensitive workloads (available via OCI CLI).
- **Instance pool + LB** — attach instance pool to a load balancer backend set so new instances auto-register: add `--load-balancers` to pool create.
- **Boot volume performance** — default boot volumes run at "balanced" VPU. Upgrade to "higher performance" (VPU=20) for production databases. Flex VPU costs extra.
- **Custom images cross-region** — use `oci compute image copy` to replicate golden images to other regions. Same OCID won't work across regions.
