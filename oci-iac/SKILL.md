---
name: oci-iac
description: Infrastructure as Code for OCI — Terraform provider, Resource Manager, Ansible, and state management patterns
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

# OCI Infrastructure as Code

## Terraform OCI Provider

```hcl
terraform {
  required_providers {
    oci = {
      source  = "oracle/oci"
      version = "~> 6.0"
    }
  }
  required_version = ">= 1.5"
}

provider "oci" {
  tenancy_ocid     = var.tenancy_ocid
  user_ocid        = var.user_ocid
  fingerprint      = var.fingerprint
  private_key_path = var.private_key_path
  region           = var.region
}
```

### Auth Alternatives (avoid hardcoded creds)

```hcl
# Instance principal (for automation running on OCI Compute)
provider "oci" {
  auth   = "InstancePrincipal"
  region = var.region
}

# Use env vars — Terraform reads them automatically:
# TF_VAR_tenancy_ocid, TF_VAR_user_ocid, etc.
# Or OCI SDK standard env: OCI_CLI_TENANCY, OCI_CLI_USER, OCI_CLI_FINGERPRINT,
#   OCI_CLI_KEY_FILE, OCI_CLI_REGION

# Config file profile
provider "oci" {
  config_file_profile = "prod"
  region              = var.region
}
```

### Variables Pattern

```hcl
# variables.tf
variable "tenancy_ocid"     { type = string }
variable "region"           { type = string  default = "us-ashburn-1" }
variable "compartment_ocid" { type = string }

# terraform.tfvars (gitignored!)
tenancy_ocid     = "ocid1.tenancy.oc1..aaaa..."
compartment_ocid = "ocid1.compartment.oc1..bbbb..."
region           = "us-ashburn-1"
```

## Key OCI Terraform Resources

### Compartment
```hcl
resource "oci_identity_compartment" "dev" {
  compartment_id = var.tenancy_ocid
  name           = "dev"
  description    = "Development environment"
  enable_delete  = true   # Allow terraform destroy to delete compartment
}
```

### VCN + Networking
```hcl
resource "oci_core_vcn" "main" {
  compartment_id = var.compartment_ocid
  cidr_blocks    = ["10.0.0.0/16"]
  display_name   = "main-vcn"
  dns_label      = "mainvcn"
}

resource "oci_core_subnet" "public" {
  compartment_id    = var.compartment_ocid
  vcn_id            = oci_core_vcn.main.id
  cidr_block        = "10.0.1.0/24"
  display_name      = "public-subnet"
  dns_label         = "pub"
  route_table_id    = oci_core_route_table.public.id
  security_list_ids = [oci_core_security_list.public.id]
  prohibit_public_ip_on_vnic = false
}

resource "oci_core_internet_gateway" "igw" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.main.id
  display_name   = "igw"
  enabled        = true
}

resource "oci_core_route_table" "public" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.main.id
  display_name   = "public-rt"

  route_rules {
    network_entity_id = oci_core_internet_gateway.igw.id
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
  }
}
```

### Compute Instance
```hcl
data "oci_core_images" "oracle_linux" {
  compartment_id           = var.compartment_ocid
  operating_system         = "Oracle Linux"
  operating_system_version = "9"
  shape                    = "VM.Standard.E4.Flex"
  sort_by                  = "TIMECREATED"
  sort_order               = "DESC"
}

resource "oci_core_instance" "web" {
  compartment_id      = var.compartment_ocid
  availability_domain = data.oci_identity_availability_domains.ads.availability_domains[0].name
  display_name        = "web-server"
  shape               = "VM.Standard.E4.Flex"

  shape_config {
    ocpus         = 2
    memory_in_gbs = 16
  }

  source_details {
    source_type             = "image"
    source_id               = data.oci_core_images.oracle_linux.images[0].id
    boot_volume_size_in_gbs = 100
  }

  create_vnic_details {
    subnet_id        = oci_core_subnet.public.id
    assign_public_ip = true
    hostname_label   = "webserver"
  }

  metadata = {
    ssh_authorized_keys = file("~/.ssh/id_rsa.pub")
    user_data           = base64encode(file("cloud-init.yaml"))
  }

  freeform_tags = {
    "Environment" = "dev"
    "ManagedBy"   = "terraform"
  }
}
```

### Autonomous Database
```hcl
resource "oci_database_autonomous_database" "atp" {
  compartment_id           = var.compartment_ocid
  db_name                  = "MYATP"
  display_name             = "my-atp"
  db_workload              = "OLTP"           # OLTP = ATP, DW = ADW
  compute_model            = "ECPU"
  compute_count            = 2
  data_storage_size_in_tbs = 1
  admin_password           = var.db_admin_password
  is_free_tier             = false
  is_auto_scaling_enabled  = true

  freeform_tags = { "Environment" = "dev" }
}
```

## Resource Manager (OCI-Native Terraform)

OCI Resource Manager runs Terraform in a managed environment — no local Terraform install needed.

### Create Stack via CLI
```bash
# Zip your Terraform configs
zip -r stack.zip *.tf terraform.tfvars

# Create stack
oci resource-manager stack create \
  --compartment-id $C \
  --display-name "MyStack" \
  --config-source '{"configSourceType":"ZIP_UPLOAD","zipFileBase64Encoded":"'$(base64 -w0 stack.zip)'"}' \
  --variables '{"compartment_ocid":"ocid1...","region":"us-ashburn-1"}'

# Or from a GitHub repo
oci resource-manager stack create \
  --compartment-id $C \
  --display-name "MyStack-Git" \
  --config-source '{
    "configSourceType": "GIT_CONFIG_SOURCE",
    "repositoryUrl": "https://github.com/myorg/oci-infra",
    "branchName": "main",
    "workingDirectory": "terraform/"
  }'
```

### Run Jobs
```bash
STACK_ID=$(oci resource-manager stack list --compartment-id $C \
  --query "data[?\"display-name\"=='MyStack'].id | [0]" --raw-output)

# Plan
oci resource-manager job create-plan-job --stack-id $STACK_ID

# Apply
oci resource-manager job create-apply-job \
  --stack-id $STACK_ID \
  --execution-plan-strategy AUTO_APPROVED

# Destroy
oci resource-manager job create-destroy-job \
  --stack-id $STACK_ID \
  --execution-plan-strategy AUTO_APPROVED

# Check job status
oci resource-manager job get --job-id <job-ocid> \
  --query 'data."lifecycle-state"' --raw-output

# Stream job logs
oci resource-manager job get-job-logs --job-id <job-ocid>
```

### Drift Detection
```bash
# Detect config drift
oci resource-manager stack detect-drift --stack-id $STACK_ID
# Check result
oci resource-manager stack list-resource-drift-details --stack-id $STACK_ID \
  --query 'data[?"resource-drift-status"!=`NOT_MODIFIED`]'
```

## Remote State with OCI Object Storage

Store Terraform state in OCI Object Storage instead of locally:

```hcl
terraform {
  backend "http" {
    address        = "https://objectstorage.us-ashburn-1.oraclecloud.com/p/<PAR>/n/<namespace>/b/<bucket>/o/terraform.tfstate"
    update_method  = "PUT"
  }
}
```

Better: use a Pre-Authenticated Request (PAR) with PUT access, or the OCI S3-compatible endpoint:

```hcl
terraform {
  backend "s3" {
    bucket   = "terraform-state"
    key      = "myproject/terraform.tfstate"
    region   = "us-ashburn-1"
    endpoint = "https://<namespace>.compat.objectstorage.us-ashburn-1.oraclecloud.com"

    skip_region_validation      = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    force_path_style            = true

    access_key = "<customer-secret-key-id>"
    secret_key = "<customer-secret-key>"
  }
}
```

## Ansible OCI Collection

```bash
pip install oci ansible ansible-collections
ansible-galaxy collection install oracle.oci
```

```yaml
# inventory/oci.yaml — dynamic inventory
plugin: oracle.oci.oci
regions:
  - us-ashburn-1
compartments:
  - compartment_ocid: "ocid1.compartment.oc1..aaaa..."
    fetch_hosts_from_subcompartments: true
compose:
  ansible_host: public_ip
hostnames:
  - "display_name"
keyed_groups:
  - key: freeform_tags.Environment
    prefix: env
```

```yaml
# playbook: provision-instance.yaml
- hosts: localhost
  collections:
    - oracle.oci
  tasks:
    - name: Create compute instance
      oci_compute_instance:
        compartment_id: "{{ compartment_ocid }}"
        availability_domain: "AD-1"
        display_name: "ansible-vm"
        shape: "VM.Standard.E4.Flex"
        shape_config:
          ocpus: 2
          memory_in_gbs: 16
        source_details:
          source_type: image
          image_id: "{{ image_id }}"
        create_vnic_details:
          subnet_id: "{{ subnet_id }}"
          assign_public_ip: true
      register: result
```

## Compartment-Based Organization Pattern

Organize resources into compartments by environment and team:

```
Tenancy (root)
├── shared-infra/          ← VCNs, DNS, shared services
├── security/              ← Vault, Cloud Guard, Bastion
├── production/
│   ├── prod-app/          ← App servers
│   └── prod-data/         ← Databases
└── development/
    ├── dev-app/
    └── dev-data/
```

```hcl
# compartments.tf — define the hierarchy
locals {
  compartments = {
    shared_infra = { parent = var.tenancy_ocid, name = "shared-infra" }
    security     = { parent = var.tenancy_ocid, name = "security" }
    production   = { parent = var.tenancy_ocid, name = "production" }
    prod_app     = { parent = "production",     name = "prod-app" }
    prod_data    = { parent = "production",     name = "prod-data" }
  }
}
```

## Gotchas & Tips

- **`enable_delete = true`** on compartments — by default, Terraform cannot delete OCI compartments. Set this flag or the destroy will silently skip it.
- **Resource Manager state** — state is managed per-stack inside OCI. You can't easily import existing resources created manually. Use `oci resource-manager stack import-tf-state` to inject a state file.
- **Provider version lock** — always pin `~> 6.0` (not `>= 6.0`) to avoid breaking changes. OCI provider has frequent releases.
- **Availability Domain names** are tenancy-specific (e.g., `kWVD:US-ASHBURN-AD-1`) — always fetch via data source, never hardcode.
- **Eventual consistency** — OCI IAM changes (policies, groups) can take 30–60 seconds to propagate. Add `time_sleep` resources after IAM changes if downstream resources need them.
- **`freeform_tags` vs `defined_tags`** — freeform tags are key-value strings; defined tags require a namespace and are schema-validated. Prefer freeform for simple cost allocation.
- **Import existing resources**: `terraform import oci_core_vcn.main <vcn-ocid>` — OCID is the import ID for all OCI resources.
