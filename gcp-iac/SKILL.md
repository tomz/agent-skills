---
name: gcp-iac
description: Infrastructure as Code for GCP — Terraform google provider, Deployment Manager, Pulumi, Config Connector
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP Infrastructure as Code Skill

Provision and manage GCP resources reliably using code. Terraform is the
recommended tool for most use cases.

---

## Terraform — Google Provider

### Provider Setup

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
  }
}

# provider.tf
provider "google" {
  project = var.project_id
  region  = var.region
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

**Gotcha:** Some resources require `google-beta` provider. Check the docs —
use `provider = google-beta` on those resources explicitly.

### Remote State in GCS (Required for Teams)

```hcl
# backend.tf
terraform {
  backend "gcs" {
    bucket = "my-tfstate-bucket"
    prefix = "env/prod"
  }
}
```

```bash
# Create the state bucket (do this once, manually or with a bootstrap script)
gsutil mb -l us-central1 gs://my-tfstate-bucket
gsutil versioning set on gs://my-tfstate-bucket
gsutil lifecycle set lifecycle.json gs://my-tfstate-bucket  # e.g. delete old versions after 90d

# Initialize with the GCS backend
terraform init
```

**Best practice:** Use separate state files per environment (prefix = "env/prod",
"env/staging"). Use a dedicated project for Terraform state storage.

### Variables and Outputs

```hcl
# variables.tf
variable "project_id" {
  type        = string
  description = "GCP project ID"
}

variable "region" {
  type    = string
  default = "us-central1"
}

variable "environment" {
  type    = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# outputs.tf
output "instance_ip" {
  value       = google_compute_instance.web.network_interface[0].access_config[0].nat_ip
  description = "External IP of the web instance"
}
```

### Common Resources

```hcl
# GCS bucket
resource "google_storage_bucket" "data" {
  name          = "${var.project_id}-data"
  location      = "US"
  force_destroy = false  # prevent accidental deletion

  versioning {
    enabled = true
  }

  lifecycle_rule {
    action { type = "Delete" }
    condition { age = 90 }
  }

  uniform_bucket_level_access = true  # disable ACLs, use IAM only
}

# Cloud SQL
resource "google_sql_database_instance" "main" {
  name             = "main-db"
  database_version = "POSTGRES_15"
  region           = var.region
  deletion_protection = true  # IMPORTANT: prevents terraform destroy from deleting the DB

  settings {
    tier = "db-g1-small"

    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.id
    }
  }
}

# Service account with binding
resource "google_service_account" "app" {
  account_id   = "app-sa"
  display_name = "Application Service Account"
}

resource "google_project_iam_member" "app_sa_role" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

### Enabling APIs

```hcl
resource "google_project_service" "apis" {
  for_each = toset([
    "compute.googleapis.com",
    "container.googleapis.com",
    "sqladmin.googleapis.com",
    "secretmanager.googleapis.com",
  ])

  project            = var.project_id
  service            = each.value
  disable_on_destroy = false  # don't disable APIs when running terraform destroy
}
```

### State Management Commands

```bash
# Import existing resource into state
terraform import google_storage_bucket.data my-existing-bucket

# Show current state
terraform state list
terraform state show google_compute_instance.web

# Move resource in state (refactoring)
terraform state mv google_compute_instance.web module.compute.google_compute_instance.web

# Remove from state without destroying (use with care)
terraform state rm google_storage_bucket.legacy

# Refresh state from real infrastructure
terraform apply -refresh-only

# Target a specific resource
terraform plan -target=google_compute_instance.web
terraform apply -target=google_compute_instance.web
```

### Workspace Pattern (for Environments)

```bash
terraform workspace new staging
terraform workspace select staging
terraform workspace list
```

**Gotcha:** Workspaces share the same backend bucket but use different state keys.
Prefer separate GCS prefixes or separate directories for stronger isolation.

---

## Module Best Practices

```hcl
# modules/gke-cluster/main.tf
variable "cluster_name" {}
variable "network"       {}
variable "subnetwork"    {}

resource "google_container_cluster" "this" {
  name     = var.cluster_name
  network  = var.network
  subnetwork = var.subnetwork
  # ... etc
}

output "cluster_endpoint" {
  value = google_container_cluster.this.endpoint
}

# Root module usage
module "gke" {
  source       = "./modules/gke-cluster"
  cluster_name = "prod-cluster"
  network      = google_compute_network.vpc.name
  subnetwork   = google_compute_subnetwork.nodes.name
}
```

Use the [Google Cloud Foundation Fabric](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric)
for battle-tested modules.

---

## Deployment Manager

Google's native IaC tool (YAML/Jinja2/Python). Prefer Terraform for new projects;
Deployment Manager is useful when you need deep GCP integration without external tools.

### Jinja2 Template

```yaml
# config.yaml
imports:
  - path: vm.jinja

resources:
  - name: my-vm
    type: vm.jinja
    properties:
      zone: us-central1-a
      machineType: n1-standard-1
      diskImage: projects/debian-cloud/global/images/family/debian-11
```

```jinja2
{# vm.jinja #}
resources:
  - name: {{ env["name"] }}
    type: compute.v1.instance
    properties:
      zone: {{ properties["zone"] }}
      machineType: zones/{{ properties["zone"] }}/machineTypes/{{ properties["machineType"] }}
      disks:
        - deviceName: boot
          type: PERSISTENT
          boot: true
          autoDelete: true
          initializeParams:
            sourceImage: {{ properties["diskImage"] }}
      networkInterfaces:
        - network: global/networks/default
```

```bash
# Deploy
gcloud deployment-manager deployments create my-deployment --config config.yaml

# Update
gcloud deployment-manager deployments update my-deployment --config config.yaml

# Preview changes
gcloud deployment-manager deployments update my-deployment --config config.yaml --preview

# Cancel preview
gcloud deployment-manager deployments cancel-preview my-deployment

# Delete
gcloud deployment-manager deployments delete my-deployment
```

---

## Pulumi for GCP

```python
# __main__.py
import pulumi
import pulumi_gcp as gcp

# Create a GCS bucket
bucket = gcp.storage.Bucket("my-bucket",
    location="US",
    uniform_bucket_level_access=True,
)

# Export the bucket URL
pulumi.export("bucket_url", bucket.url)
```

```bash
# Setup
pip install pulumi pulumi-gcp
pulumi new gcp-python

# State backend (use GCS for teams)
pulumi login gs://my-pulumi-state-bucket

# Preview
pulumi preview

# Deploy
pulumi up

# Destroy
pulumi destroy

# Stack management
pulumi stack init prod
pulumi stack select prod
```

---

## Config Connector (GKE)

Config Connector lets you manage GCP resources via Kubernetes CRDs.

```bash
# Install Config Connector addon on GKE cluster
gcloud container clusters update my-cluster \
  --update-addons ConfigConnector=ENABLED \
  --zone us-central1-a

# Apply a GCP resource via kubectl
kubectl apply -f storage-bucket.yaml
```

```yaml
# storage-bucket.yaml
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  name: my-bucket
  namespace: config-connector
  annotations:
    cnrm.cloud.google.com/project-id: my-project
spec:
  location: US
  uniformBucketLevelAccess: true
```

---

## CI/CD Integration

```bash
# Authenticate in CI using Workload Identity Federation (no SA keys)
# GitHub Actions example — see gcp-devops skill for full workflow

# Run Terraform in CI
terraform init
terraform validate
terraform plan -out=tfplan
terraform apply tfplan  # only on merge to main
```

---

## Guardrails

- **Always use remote state** — never commit `.tfstate` files.
- **Set `deletion_protection = true`** on databases and critical resources.
- **Use `prevent_destroy = true`** lifecycle rule for production resources.
- **Never use `terraform apply` without reviewing `terraform plan` first.**
- **Don't store credentials in `.tf` files.** Use environment variables or ADC.
- **Lock provider versions** with `~>` — never use `>= 5.0` without an upper bound.
- **`-target` is a last resort** — it leaves state inconsistent; prefer refactoring.
- **Deployment Manager and Terraform don't share state** — pick one per resource.
