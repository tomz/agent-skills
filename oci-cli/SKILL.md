---
name: oci-cli
description: Oracle Cloud Infrastructure CLI patterns, authentication, config, and common commands
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

# OCI CLI — Patterns, Auth, and Common Commands

## Installation & Setup

```bash
# Install via pip (cross-platform)
pip install oci-cli

# Or the official install script (Linux/macOS)
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

# Verify
oci --version

# Interactive setup wizard (creates ~/.oci/config and API key pair)
oci setup config
```

## Config File (~/.oci/config)

The OCI CLI config file lives at `~/.oci/config`. It supports **named profiles**.

```ini
[DEFAULT]
user=ocid1.user.oc1..aaaa...
fingerprint=aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99
tenancy=ocid1.tenancy.oc1..aaaa...
region=us-ashburn-1
key_file=~/.oci/oci_api_key.pem

[prod]
user=ocid1.user.oc1..bbbb...
fingerprint=11:22:33:44:55:66:77:88:99:aa:bb:cc:dd:ee:ff:00
tenancy=ocid1.tenancy.oc1..bbbb...
region=eu-frankfurt-1
key_file=~/.oci/prod_key.pem

[dev]
# Inherits DEFAULT tenancy, overrides region
region=us-phoenix-1
```

**Switch profiles:**
```bash
oci iam compartment list --profile prod
export OCI_CLI_PROFILE=prod   # Set for entire shell session
```

## Authentication Methods

### 1. API Key (default)
Generate key pair and upload public key to OCI Console → Identity → Users → API Keys.
```bash
oci setup keys            # Generate key pair interactively
oci setup repair-file-permissions --file ~/.oci/oci_api_key.pem
```

### 2. Instance Principals
For compute instances — no API keys needed. IAM policy required:
```
allow dynamic-group MyInstanceDG to manage all-resources in compartment MyCompartment
```
```bash
oci iam compartment list --auth instance_principal
```

### 3. Resource Principals
For OCI Functions running inside OCI. Auth is automatic from runtime environment.
```bash
oci os object list --bucket-name my-bucket --auth resource_principal
```

### 4. Session Tokens (for MFA/federation)
```bash
oci session authenticate --region us-ashburn-1 --profile-name my-session
oci iam compartment list --auth security_token --profile my-session
# Tokens expire after 1 hour (extendable to 24h with --session-expiry-in-minutes)
oci session refresh --profile my-session
```

### 5. CloudShell
OCI CloudShell in the Console has pre-authenticated access — no config needed.

## Tenancy & Compartment Hierarchy

OCIDs uniquely identify every resource:
```
ocid1.<resource-type>.<realm>.<region>[.<future-use>].<unique-id>
```

```bash
# Get your tenancy OCID
oci iam tenancy get --tenancy-id $(oci iam compartment list --all --query 'data[0]."compartment-id"' --raw-output)

# List all compartments (recursive)
oci iam compartment list --compartment-id <tenancy-ocid> --all --compartment-id-in-subtree true

# Get compartment OCID by name
oci iam compartment list --all --query "data[?name=='MyCompartment'].id | [0]" --raw-output

# Create a sub-compartment
oci iam compartment create \
  --compartment-id <parent-compartment-ocid> \
  --name "MySubCompartment" \
  --description "Development resources"
```

## Output Formats

```bash
# JSON (default) — machine-readable, full detail
oci iam user list --output json

# Table — human-readable
oci iam user list --output table

# Raw output — strips JSON quotes (useful for scripts)
oci iam user list --query 'data[0].id' --raw-output

# Quiet — suppress progress/status messages
oci os object put --bucket-name my-bucket --file ./data.csv --quiet
```

## JMESPath Queries

OCI CLI supports JMESPath with `--query`. Use single quotes on Linux/macOS.

```bash
# Get specific field
oci compute instance list --compartment-id $C --query 'data[0]."display-name"'

# Filter list
oci compute instance list --compartment-id $C \
  --query 'data[?contains("display-name", `web`)]'

# Running instances only
oci compute instance list --compartment-id $C \
  --query 'data[?"lifecycle-state"==`RUNNING`]."display-name"'

# Extract multiple fields
oci compute instance list --compartment-id $C \
  --query 'data[*].{"name":"display-name","state":"lifecycle-state","shape":"shape"}'

# Count items
oci compute instance list --compartment-id $C --all \
  --query 'length(data)'
```

## Common Commands

### IAM
```bash
oci iam user list --all
oci iam group list --all
oci iam policy list --compartment-id <ocid> --all
oci iam region list          # All available regions
oci iam availability-domain list --compartment-id <tenancy-ocid>
```

### Compute
```bash
# List instances
oci compute instance list --compartment-id $C --output table \
  --query 'data[*].{"Name":"display-name","Shape":"shape","State":"lifecycle-state"}'

# Launch instance (minimal)
oci compute instance launch \
  --compartment-id $C \
  --availability-domain "AD-1" \
  --shape "VM.Standard.E4.Flex" \
  --shape-config '{"ocpus":2,"memoryInGBs":16}' \
  --image-id <image-ocid> \
  --subnet-id <subnet-ocid> \
  --display-name "my-vm"

# Start/stop/terminate
oci compute instance action --instance-id <ocid> --action STOP
oci compute instance action --instance-id <ocid> --action START
oci compute instance terminate --instance-id <ocid> --force

# Get public IP
oci compute instance list-vnics --instance-id <ocid> \
  --query 'data[0]."public-ip"' --raw-output
```

### Object Storage
```bash
# List buckets
oci os bucket list --compartment-id $C --all

# Create bucket
oci os bucket create --compartment-id $C --name my-bucket --storage-tier Standard

# Upload file
oci os object put --bucket-name my-bucket --file ./report.pdf

# Upload large file with multipart
oci os object put --bucket-name my-bucket --file ./bigfile.tar.gz \
  --parallel-upload-count 10 --part-size 100

# Download
oci os object get --bucket-name my-bucket --name report.pdf --file ./report.pdf

# List objects
oci os object list --bucket-name my-bucket --all

# Bulk upload directory
oci os object bulk-upload --bucket-name my-bucket --src-dir ./data/ --include "*.csv"

# Delete object
oci os object delete --bucket-name my-bucket --name old-report.pdf --force
```

### Networking
```bash
oci network vcn list --compartment-id $C --all
oci network subnet list --compartment-id $C --all --output table
oci network security-list list --compartment-id $C --all
oci network nsg list --compartment-id $C --all
```

### Database
```bash
oci db autonomous-database list --compartment-id $C --all --output table
oci db autonomous-database create \
  --compartment-id $C \
  --display-name "MyATP" \
  --db-name "MYATP" \
  --cpu-core-count 1 \
  --data-storage-size-in-tbs 1 \
  --admin-password 'Str0ng#Pass!'
```

## Pagination & Large Result Sets

```bash
# --all fetches all pages automatically
oci compute instance list --compartment-id $C --all

# Manual pagination
oci compute instance list --compartment-id $C --limit 100 --page <next-page-token>

# Get page token from response
oci compute instance list --compartment-id $C --limit 5 | jq -r '.["opc-next-page"]'
```

## Useful CLI Flags

```bash
--debug          # Full HTTP request/response debug output
--request-id     # Set a custom request ID for tracing
--no-retry       # Disable automatic retries
--max-retries 5  # Override default retry count
--wait-for-state ACTIVE --max-wait-seconds 300  # Block until resource reaches state
--dry-run        # Preview without executing (not all commands support this)
```

## Shell Helpers / Patterns

```bash
# Set compartment ID from env
export C=$(oci iam compartment list --all \
  --query "data[?name=='MyCompartment'].id | [0]" --raw-output)

# Wait for instance to start
oci compute instance launch ... --wait-for-state RUNNING --max-wait-seconds 600

# Delete all objects in a bucket before deleting the bucket
oci os object bulk-delete --bucket-name my-bucket --force
oci os bucket delete --name my-bucket --force

# Get all running instances across all compartments (requires jq)
oci compute instance list --compartment-id <tenancy-ocid> \
  --compartment-id-in-subtree true --all \
  --query 'data[?"lifecycle-state"==`RUNNING`]' | jq '.[].{"name","shape"}'
```

## Gotchas & Tips

- **OCIDs are region-scoped** — a resource OCID from `us-ashburn-1` won't work in API calls targeting another region. Always match `--region` to your resource.
- **Rate limiting** — OCI APIs throttle at ~60 req/min for most endpoints. Use `--all` rather than looping with pagination when possible.
- **Tenancy root compartment** — the tenancy OCID IS the root compartment OCID. Use it for top-level `--compartment-id` lookups.
- **`--query` runs client-side** — it filters after fetching all results. For large result sets, use server-side filters (`--display-name`, `--lifecycle-state`) first.
- **Key file permissions must be 600** — OCI CLI rejects keys with loose permissions. Run `chmod 600 ~/.oci/oci_api_key.pem`.
- **CloudShell region** — CloudShell targets the Console's currently selected region. Switch the Console region to change it.
- **`--force`** — many destructive commands require `--force` to skip confirmation. Use carefully in scripts.
