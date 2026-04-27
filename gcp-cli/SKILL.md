---
name: gcp-cli
description: gcloud CLI patterns, authentication, configurations, output formats, gsutil, bq CLI
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP CLI Skill

Master the `gcloud`, `gsutil`, and `bq` command-line tools for Google Cloud Platform.

---

## Configurations (Named Profiles)

gcloud supports named configurations — like AWS profiles. Always use them; never
rely on a single implicit active config.

```bash
# List all configurations
gcloud config configurations list

# Create a new named configuration
gcloud config configurations create prod
gcloud config configurations create dev

# Activate a configuration
gcloud config configurations activate prod

# Set properties in a configuration
gcloud config set project my-prod-project
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
gcloud config set account user@example.com

# View current configuration
gcloud config list

# View a specific configuration
gcloud config configurations describe prod

# Run a single command with a different config (without activating it)
gcloud compute instances list --configuration=dev

# Delete a configuration
gcloud config configurations delete old-config
```

**Gotcha:** `gcloud config set` modifies the *active* configuration. Always verify
which config is active before running mutating commands.

---

## Authentication

### Interactive Login (human users)

```bash
# Browser-based login — sets Application Default Credentials (ADC)
gcloud auth login

# Login AND set ADC in one step (recommended for local dev)
gcloud auth login --update-adc

# Set ADC separately (what SDKs use)
gcloud auth application-default login

# Revoke credentials
gcloud auth revoke user@example.com
gcloud auth application-default revoke
```

### Service Accounts

```bash
# Create a service account
gcloud iam service-accounts create my-sa \
  --display-name="My Service Account" \
  --project=my-project

# Create and download a key (avoid if possible — use Workload Identity instead)
gcloud iam service-accounts keys create ~/sa-key.json \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com

# Activate service account for gcloud
gcloud auth activate-service-account \
  --key-file=~/sa-key.json

# Impersonate a service account (requires roles/iam.serviceAccountTokenCreator)
gcloud config set auth/impersonate_service_account my-sa@my-project.iam.gserviceaccount.com
# Clear impersonation
gcloud config unset auth/impersonate_service_account

# List keys on a service account
gcloud iam service-accounts keys list \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com

# Delete old keys
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com
```

**Gotcha:** Service account JSON keys are long-lived credentials. Rotate them or,
better, use Workload Identity Federation / metadata server instead.

---

## Projects

```bash
# List projects you have access to
gcloud projects list

# Set default project
gcloud config set project PROJECT_ID

# Describe a project
gcloud projects describe PROJECT_ID

# Create a project (under an org or folder)
gcloud projects create my-new-project \
  --name="My New Project" \
  --organization=123456789

# Create under a folder
gcloud projects create my-new-project \
  --folder=FOLDER_ID

# Move a project to a different folder
gcloud projects move PROJECT_ID --folder=NEW_FOLDER_ID

# Enable an API
gcloud services enable container.googleapis.com

# List enabled APIs
gcloud services list --enabled

# Disable an API (careful!)
gcloud services disable container.googleapis.com --force
```

---

## Output Formats

The `--format` flag controls output. Always use machine-readable formats in scripts.

```bash
# Human-readable table (default)
gcloud compute instances list

# JSON — full resource representation
gcloud compute instances list --format=json

# YAML
gcloud compute instances list --format=yaml

# CSV with specific fields
gcloud compute instances list \
  --format="csv(name,zone,status,machineType)"

# Table with custom columns
gcloud compute instances list \
  --format="table(name,zone.basename(),status,networkInterfaces[0].networkIP)"

# Value only (single field, good for scripting)
gcloud config get-value project

# Extract a field with --format=value
gcloud compute instances describe my-vm --zone=us-central1-a \
  --format="value(networkInterfaces[0].accessConfigs[0].natIP)"

# Get multiple values tab-separated
gcloud compute instances list \
  --format="value(name,status)"

# Flatten repeated fields
gcloud compute instances list \
  --format="table(name,networkInterfaces[].networkIP)"
```

### Filters

```bash
# Filter by field value
gcloud compute instances list --filter="status=RUNNING"

# Multiple conditions
gcloud compute instances list \
  --filter="status=RUNNING AND zone:us-central1"

# Label filters
gcloud compute instances list \
  --filter="labels.env=production"

# Negation
gcloud compute instances list \
  --filter="NOT status=TERMINATED"

# Glob patterns
gcloud compute instances list --filter="name:web-*"

# Combine with format for scriptable output
gcloud compute instances list \
  --filter="status=RUNNING" \
  --format="value(name)" \
  | while read inst; do echo "Processing $inst"; done
```

---

## gsutil (Cloud Storage)

```bash
# Copy local → GCS
gsutil cp file.txt gs://my-bucket/path/file.txt

# Copy GCS → local
gsutil cp gs://my-bucket/path/file.txt ./file.txt

# Recursive copy
gsutil cp -r ./local-dir gs://my-bucket/prefix/

# Parallel composite upload (faster for large files)
gsutil -o GSUtil:parallel_composite_upload_threshold=150M cp large-file.tar.gz gs://my-bucket/

# Sync directories (like rsync)
gsutil rsync -r ./local-dir gs://my-bucket/prefix/
gsutil rsync -r -d ./local-dir gs://my-bucket/prefix/  # -d deletes extra remote files

# List bucket contents
gsutil ls gs://my-bucket/
gsutil ls -la gs://my-bucket/  # with sizes and timestamps

# Remove objects
gsutil rm gs://my-bucket/file.txt
gsutil rm -r gs://my-bucket/prefix/  # recursive

# Set object metadata
gsutil setmeta -h "Cache-Control:public, max-age=3600" gs://my-bucket/static/**

# Make object public
gsutil acl ch -u AllUsers:R gs://my-bucket/public-file.txt

# Create a bucket
gsutil mb -l us-central1 gs://my-new-bucket

# Enable versioning
gsutil versioning set on gs://my-bucket

# View bucket IAM policy
gsutil iam get gs://my-bucket

# Grant IAM on bucket
gsutil iam ch user:someone@example.com:objectAdmin gs://my-bucket

# Signed URL (7 days, requires SA key or impersonation)
gsutil signurl -d 7d ~/sa-key.json gs://my-bucket/private-file.zip
```

**Gotcha:** `gsutil` uses boto config, not gcloud config. Use
`gcloud storage` commands (newer, GA) if you want consistent auth.

```bash
# gcloud storage (modern replacement for gsutil)
gcloud storage cp file.txt gs://my-bucket/
gcloud storage ls gs://my-bucket/
gcloud storage rm -r gs://my-bucket/prefix/
```

---

## bq CLI (BigQuery)

```bash
# List datasets
bq ls --project_id=my-project

# Show dataset details
bq show my-project:my_dataset

# Create a dataset
bq mk --dataset --location=US my-project:my_dataset

# Run a query (interactive)
bq query --use_legacy_sql=false \
  'SELECT COUNT(*) FROM `my-project.my_dataset.my_table`'

# Run a query and write to a table
bq query --use_legacy_sql=false \
  --destination_table=my-project:my_dataset.output_table \
  --replace \
  'SELECT * FROM `my-project.my_dataset.source` WHERE date = CURRENT_DATE()'

# Load data from GCS
bq load --source_format=CSV \
  --autodetect \
  my-project:my_dataset.my_table \
  gs://my-bucket/data/*.csv

# Load newline-delimited JSON
bq load --source_format=NEWLINE_DELIMITED_JSON \
  my-project:my_dataset.my_table \
  gs://my-bucket/data.jsonl \
  schema.json

# Export table to GCS
bq extract \
  --destination_format=CSV \
  my-project:my_dataset.my_table \
  gs://my-bucket/export/data-*.csv

# Describe table schema
bq show --schema --format=prettyjson my-project:my_dataset.my_table

# Copy a table
bq cp my-project:my_dataset.source_table my-project:my_dataset.dest_table

# Delete a table
bq rm -f my-project:my_dataset.my_table

# Delete a dataset (and all contents)
bq rm -r -f my-project:my_dataset

# Check job status
bq show -j my-job-id

# Cancel a running job
bq cancel my-job-id
```

---

## Useful Patterns

### Script-safe project targeting

Always pass `--project` explicitly in scripts — never rely on active config:

```bash
PROJECT="my-project"
gcloud compute instances list --project="$PROJECT" --format=json
```

### Quiet/non-interactive mode

```bash
# Suppress prompts
gcloud compute instances delete my-vm --zone=us-central1-a --quiet

# Or set globally (useful in CI)
gcloud config set core/disable_prompts true
```

### Getting help

```bash
gcloud help                    # Top-level help
gcloud compute instances --help
gcloud topic filters           # Filter syntax reference
gcloud topic formats           # Format syntax reference
gcloud topic projections       # Projection syntax reference
```

### Check current auth/identity

```bash
gcloud auth list
gcloud config list account
gcloud auth describe $(gcloud auth list --format="value(account)" --filter="status=ACTIVE")
```

---

## Guardrails

- **Never run `gcloud` mutating commands without `--project` in CI/scripts.** Active configs drift.
- **Avoid SA JSON keys.** Prefer Workload Identity Federation or metadata server ADC.
- **Test `--filter` and `--format` interactively before embedding in scripts.**
- **`gsutil rm -r` with `-d` in rsync is destructive** — double-check source/dest order.
- **`bq mk --dataset` location is permanent** — choose region carefully upfront.
- **Don't store credentials in source control.** Never commit `sa-key.json` or ADC files.
