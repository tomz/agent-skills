---
name: ibm-openshift
description: Red Hat OpenShift on IBM Cloud (ROKS) — cluster lifecycle, worker pools, oc CLI, networking, IBM Cloud integrations
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

# Red Hat OpenShift on IBM Cloud (ROKS) Skill

ROKS is a fully managed OpenShift offering on IBM Cloud. Clusters run on IBM Cloud infrastructure
(VPC or Classic) with IBM managing the control plane. You interact via `ibmcloud ks` (the
container-service plugin) and the standard `oc` CLI.

---

## Prerequisites

```bash
# Install required plugins
ibmcloud plugin install container-service   # provides `ibmcloud ks`
ibmcloud plugin install container-registry  # provides `ibmcloud cr`

# Install oc CLI (matches your cluster's OCP version)
# Download from: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/
# Or via IBM Cloud console → cluster → OpenShift web console → ? → Command line tools

# Verify
ibmcloud ks version
oc version
```

---

## Cluster Creation

### VPC cluster (recommended for new deployments)
```bash
# List available VPC OpenShift versions
ibmcloud ks versions --show-version openshift

# Create a VPC Gen2 ROKS cluster
ibmcloud ks cluster create vpc-gen2 \
  --name my-roks-cluster \
  --version 4.15_openshift \
  --zone us-south-1 \
  --vpc-id $VPC_ID \
  --subnet-id $SUBNET_ID \
  --flavor bx2.4x16 \
  --workers 3 \
  --resource-group-name production \
  --disable-public-service-endpoint          # private-only (optional, high-security)

# List VPCs
ibmcloud is vpcs
# List subnets in a zone
ibmcloud is subnets --zone us-south-1
# List available flavors
ibmcloud ks flavors --zone us-south-1 --provider vpc-gen2
```

### Classic cluster
```bash
ibmcloud ks cluster create classic \
  --name my-classic-roks \
  --version 4.15_openshift \
  --zone dal10 \
  --machine-type b3c.4x16 \
  --workers 3 \
  --public-vlan $PUBLIC_VLAN \
  --private-vlan $PRIVATE_VLAN
```

---

## Cluster Lifecycle

```bash
# List clusters
ibmcloud ks clusters
ibmcloud ks clusters --output json | jq -r '.[] | "\(.name)\t\(.state)\t\(.masterKubeVersion)"'

# Get cluster details
ibmcloud ks cluster get --cluster my-roks-cluster

# Watch cluster state (poll until normal)
watch -n 15 'ibmcloud ks cluster get --cluster my-roks-cluster | grep State'

# Download kubeconfig / oc login
ibmcloud ks cluster config --cluster my-roks-cluster
# For admin access (bypasses OAuth — use sparingly)
ibmcloud ks cluster config --cluster my-roks-cluster --admin

# Get the cluster's OpenShift web console URL
ibmcloud ks cluster get --cluster my-roks-cluster | grep "Public Service Endpoint URL"

# Update master to a new version
ibmcloud ks cluster master update --cluster my-roks-cluster --version 4.16_openshift

# Remove cluster (DESTRUCTIVE — no undo)
ibmcloud ks cluster rm --cluster my-roks-cluster -f
```

---

## Worker Pools

ROKS clusters have worker pools (not individual workers). The default pool is created at
cluster creation; add more for workload isolation or zone redundancy.

```bash
# List worker pools
ibmcloud ks worker-pool ls --cluster my-roks-cluster

# Get pool details (zones, flavor, count)
ibmcloud ks worker-pool get --cluster my-roks-cluster --worker-pool default

# Add a new worker pool
ibmcloud ks worker-pool create vpc-gen2 \
  --cluster my-roks-cluster \
  --name gpu-pool \
  --flavor gx2.8x64.2xv100 \
  --size-per-zone 2 \
  --label workload=gpu

# Add a zone to a worker pool (spread across AZs for HA)
ibmcloud ks zone add vpc-gen2 \
  --cluster my-roks-cluster \
  --worker-pool default \
  --zone us-south-2 \
  --subnet-id $SUBNET_ID_ZONE2

# Resize a pool
ibmcloud ks worker-pool resize \
  --cluster my-roks-cluster \
  --worker-pool default \
  --size-per-zone 5

# Delete a worker pool
ibmcloud ks worker-pool rm --cluster my-roks-cluster --worker-pool gpu-pool -f

# Enable cluster autoscaler on a pool
ibmcloud ks worker-pool autoscale set \
  --cluster my-roks-cluster \
  --worker-pool default \
  --min-workers 2 \
  --max-workers 10
```

**Gotcha:** Worker pool changes (resize, zone add) can take 10–20 minutes. Workers go through
`provisioning → deploying → normal`. Check with:
```bash
ibmcloud ks workers --cluster my-roks-cluster --worker-pool default
```

---

## oc CLI — OpenShift-Specific Commands

After `ibmcloud ks cluster config --cluster my-roks-cluster`:

```bash
# Cluster info
oc cluster-info
oc get nodes
oc get nodes -o wide   # shows IPs, OS, kernel version

# Projects (namespaces)
oc new-project my-app
oc project my-app       # switch to project
oc get projects

# Deploy from image
oc new-app --image=us.icr.io/my-ns/my-app:latest --name=my-app

# Expose as Route (OpenShift's Ingress equivalent)
oc expose svc/my-app
oc get routes

# DeploymentConfig vs Deployment
# DeploymentConfig (OCP-native, older):
oc get dc
oc rollout latest dc/my-app
oc rollout status dc/my-app

# Deployment (Kubernetes-native, preferred for new apps):
oc get deployments
oc rollout restart deployment/my-app
oc rollout status deployment/my-app

# Scale
oc scale deployment/my-app --replicas=5

# Logs
oc logs deployment/my-app --tail=100 -f
oc logs dc/my-app -c my-container

# Exec into pod
oc exec -it $(oc get pod -l app=my-app -o name | head -1) -- /bin/bash

# Debug a node (starts privileged pod on that node)
oc debug node/<node-name>

# Resource quotas and limits
oc get resourcequota
oc get limitrange
```

---

## Routes (OpenShift Ingress)

```bash
# Create a route from a service
oc expose svc/my-app --hostname=my-app.apps.my-cluster.example.com

# TLS edge-terminated route
oc create route edge my-app-tls \
  --service=my-app \
  --cert=tls.crt \
  --key=tls.key \
  --hostname=my-app.example.com

# Passthrough (TLS handled by pod)
oc create route passthrough my-app-passthrough \
  --service=my-app \
  --hostname=my-app.example.com

# List routes
oc get routes -A   # all namespaces

# Check route status (admitted = working)
oc describe route my-app | grep -A5 "Ingress"
```

---

## IBM Cloud Registry (ICR) Integration

```bash
# Login to ICR (sets Docker credentials)
ibmcloud cr login

# Create a namespace in ICR
ibmcloud cr namespace-add my-namespace

# List images
ibmcloud cr images --namespace my-namespace

# Build and push
docker build -t us.icr.io/my-namespace/my-app:latest .
docker push us.icr.io/my-namespace/my-app:latest

# Create image pull secret in OpenShift project
ibmcloud ks cluster pull-secret apply --cluster my-roks-cluster
# This patches the default service account with ICR pull credentials

# Manually create pull secret
oc create secret docker-registry icr-secret \
  --docker-server=us.icr.io \
  --docker-username=iamapikey \
  --docker-password="$IBMCLOUD_API_KEY"

oc secrets link default icr-secret --for=pull
```

**Regional ICR endpoints:** `us.icr.io` (Dallas), `uk.icr.io`, `de.icr.io`, `au.icr.io`, `jp.icr.io`

---

## IBM Cloud Integrations

### Secrets Manager
```bash
# Install the Secrets Manager operator from OperatorHub in OCP console
# Or use the External Secrets Operator:
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace

# Create a SecretStore pointing to IBM Secrets Manager
# (requires a service ID API key as a k8s secret)
```

### Log Analysis (IBM Cloud Logging)
```bash
# Enable log forwarding from cluster
ibmcloud ob logging config create \
  --cluster my-roks-cluster \
  --instance $LOG_ANALYSIS_INSTANCE_ID

# Check logging config
ibmcloud ob logging config list --cluster my-roks-cluster
```

### IBM Cloud Monitoring (Sysdig)
```bash
ibmcloud ob monitoring config create \
  --cluster my-roks-cluster \
  --instance $MONITORING_INSTANCE_ID

ibmcloud ob monitoring config list --cluster my-roks-cluster
```

---

## Operators

```bash
# List installed operators
oc get operators -A
oc get csv -A   # ClusterServiceVersions = installed operator versions

# Install operator via CLI (OperatorGroup + Subscription)
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: my-operator
  source: certified-operators
  sourceNamespace: openshift-marketplace
EOF

# Check operator status
oc get csv -n openshift-operators
oc describe csv my-operator.v1.2.3 -n openshift-operators | grep -A10 "Phase"
```

---

## VPC Networking for ROKS

```bash
# Security groups (VPC clusters)
ibmcloud is security-groups | grep $CLUSTER_ID

# The cluster creates security groups:
# kube-<cluster-id>       — worker-to-worker + master communication
# kube-<cluster-id>-alb   — load balancer traffic

# Allow custom inbound to workers (e.g., NodePort 30000-32767)
ibmcloud is sg-rule-add kube-$CLUSTER_ID \
  --direction inbound \
  --protocol tcp \
  --port-min 30000 \
  --port-max 32767 \
  --source 0.0.0.0/0

# VPC Load Balancer (created by LoadBalancer service type)
ibmcloud is load-balancers | grep $CLUSTER_ID
```

---

## Guardrails

- **Never use `--admin` kubeconfig in day-to-day work** — it bypasses RBAC and leaves no audit trail.
- **Prefer `Deployment` over `DeploymentConfig`** for new apps — DC is ROKS-specific and being phased out.
- **Worker pool updates are rolling** — removing zones or resizing down can cause workload disruption; drain first.
- **ICR pull secrets expire if API key is rotated** — re-run `ibmcloud ks cluster pull-secret apply` after key rotation.
- **Routes on ROKS use wildcard DNS** `*.apps.<cluster-domain>` — custom hostnames need DNS delegation.
- **`oc debug node/` requires cluster-admin** — use judiciously in production.
- **Resource quotas are project-scoped** — always check `oc get resourcequota` before large deployments.
