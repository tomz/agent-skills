---
name: gcp-security
description: GCP security — IAM, Secret Manager, SCC, VPC Service Controls, Binary Authorization, Certificate Authority Service, BeyondCorp, Org Policies
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP Security Skill

Defense-in-depth for GCP: identity, secrets, policy, and threat detection.

---

## IAM Fundamentals

### Roles

- **Primitive roles:** `roles/owner`, `roles/editor`, `roles/viewer` — too broad, avoid in prod.
- **Predefined roles:** `roles/storage.objectAdmin`, `roles/container.developer` — use these.
- **Custom roles:** Create your own with exact permissions needed.

```bash
# List predefined roles
gcloud iam roles list

# Describe a role (see exact permissions)
gcloud iam roles describe roles/storage.objectAdmin

# Create a custom role at project level
gcloud iam roles create myCustomRole \
  --project=my-project \
  --title="My Custom Role" \
  --description="Read-only access to Cloud SQL and Secret Manager" \
  --permissions=cloudsql.instances.get,cloudsql.instances.list,secretmanager.versions.access \
  --stage=GA

# Update a custom role (add/remove permissions)
gcloud iam roles update myCustomRole \
  --project=my-project \
  --add-permissions=cloudsql.databases.list

# Disable (soft delete) a custom role
gcloud iam roles update myCustomRole \
  --project=my-project \
  --stage=DISABLED
```

### Bindings

```bash
# Grant a role to a user
gcloud projects add-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# Grant to a group
gcloud projects add-iam-policy-binding my-project \
  --member="group:data-team@example.com" \
  --role="roles/bigquery.dataViewer"

# Grant to a service account
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# Grant to allUsers (public — use with extreme caution)
gcloud projects add-iam-policy-binding my-project \
  --member="allUsers" \
  --role="roles/storage.objectViewer"

# Remove a binding
gcloud projects remove-iam-policy-binding my-project \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# View the full IAM policy
gcloud projects get-iam-policy my-project

# View policy as JSON (scriptable)
gcloud projects get-iam-policy my-project --format=json
```

### IAM Conditions

Grant access only under specific conditions (time, resource, request attributes).

```bash
# Bind with a condition (e.g., only during business hours)
gcloud projects add-iam-policy-binding my-project \
  --member="user:contractor@example.com" \
  --role="roles/compute.viewer" \
  --condition='expression=request.time.getHours("America/New_York") >= 9 && request.time.getHours("America/New_York") < 17,title=BusinessHours,description=Access only during business hours'

# Resource-based condition (only specific bucket)
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="user:alice@example.com" \
  --role="roles/storage.objectAdmin" \
  --condition='expression=resource.name.startsWith("projects/_/buckets/my-bucket/objects/alice/"),title=OwnFolder'
```

### Workload Identity Federation (No SA Keys)

Allow workloads outside GCP (GitHub Actions, AWS, on-prem) to authenticate as GCP SAs.

```bash
# Create a Workload Identity Pool
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --display-name="GitHub Actions Pool"

# Add GitHub as an OIDC provider
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --issuer-uri=https://token.actions.githubusercontent.com \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.actor=assertion.actor" \
  --attribute-condition="assertion.repository=='myorg/myrepo'"

# Grant SA access from the pool
gcloud iam service-accounts add-iam-policy-binding deploy-sa@my-project.iam.gserviceaccount.com \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/myorg/myrepo" \
  --role="roles/iam.workloadIdentityUser"
```

---

## Secret Manager

```bash
# Create a secret
echo -n "my-super-secret-value" | \
  gcloud secrets create my-secret \
    --data-file=- \
    --replication-policy=automatic

# Add a new version
echo -n "new-secret-value" | \
  gcloud secrets versions add my-secret --data-file=-

# Access a secret version
gcloud secrets versions access latest --secret=my-secret

# Access a specific version
gcloud secrets versions access 3 --secret=my-secret

# List versions
gcloud secrets versions list my-secret

# Disable a version (revoke without deleting)
gcloud secrets versions disable 2 --secret=my-secret

# Destroy a version (permanent — can't be undone)
gcloud secrets versions destroy 1 --secret=my-secret

# Delete a secret entirely
gcloud secrets delete my-secret

# Grant access to a secret
gcloud secrets add-iam-policy-binding my-secret \
  --member="serviceAccount:app-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Rotate a secret and add expiration
gcloud secrets create my-db-password \
  --replication-policy=automatic \
  --next-rotation-time=2025-01-01T00:00:00Z \
  --rotation-period=2592000s  # 30 days
```

**In application code:**
```python
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()
name = "projects/my-project/secrets/my-secret/versions/latest"
response = client.access_secret_version(request={"name": name})
secret_value = response.payload.data.decode("UTF-8")
```

---

## Security Command Center (SCC)

SCC aggregates findings from across GCP (misconfigurations, threats, vulnerabilities).

```bash
# List high-severity active findings
gcloud scc findings list my-project \
  --filter="state=ACTIVE AND severity=HIGH" \
  --format="table(name,category,resourceName,eventTime)"

# List findings for a specific category
gcloud scc findings list my-project \
  --filter="category=PUBLIC_BUCKET_ACL AND state=ACTIVE"

# Mark a finding as resolved
gcloud scc findings update FINDING_NAME \
  --state=INACTIVE

# List assets
gcloud scc assets list my-project \
  --filter="resourceProperties.resourceType=google.compute.Firewall"

# Run a security health analytics scan
gcloud scc settings update \
  --organization=ORG_ID \
  --enable-asset-discovery
```

---

## VPC Service Controls

```bash
# List access policies
gcloud access-context-manager policies list

# Create a perimeter in dry-run mode first
gcloud access-context-manager perimeters dry-run create my-perimeter \
  --policy=POLICY_ID \
  --title="Data Perimeter" \
  --resources=projects/PROJECT_NUMBER \
  --restricted-services=bigquery.googleapis.com,storage.googleapis.com

# Check what would be blocked (dry-run logs to Cloud Logging)
gcloud logging read \
  'protoPayload.@type="type.googleapis.com/google.cloud.audit.AuditLog" AND
   protoPayload.metadata.@type="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata" AND
   protoPayload.metadata.vpcServiceControlsUniqueId!=""' \
  --project=my-project \
  --limit=20

# Enforce the perimeter (move from dry-run to enforced)
gcloud access-context-manager perimeters dry-run enforce my-perimeter \
  --policy=POLICY_ID

# Add an ingress rule (allow specific service account from outside perimeter)
gcloud access-context-manager perimeters update my-perimeter \
  --policy=POLICY_ID \
  --add-ingress-policies='ingressFrom.sources.resource=//cloudresourcemanager.googleapis.com/projects/external-project,ingressFrom.identities=serviceAccount:external-sa@external-project.iam.gserviceaccount.com,ingressTo.operations.serviceName=bigquery.googleapis.com,ingressTo.operations.methodSelectors.method=*'
```

---

## Binary Authorization

Enforce that only trusted container images are deployed to GKE.

```bash
# Get the default policy
gcloud container binauthz policy export > policy.yaml

# Import an updated policy
gcloud container binauthz policy import policy.yaml

# Create an attestor
gcloud container binauthz attestors create my-attestor \
  --attestation-authority-note=my-note \
  --attestation-authority-note-project=my-project

# Sign an image (attest it)
gcloud container binauthz attestations sign-and-create \
  --artifact-url=us-docker.pkg.dev/my-project/repo/image@sha256:ABC123 \
  --attestor=my-attestor \
  --attestor-project=my-project \
  --keyversion=projects/my-project/locations/global/keyRings/my-ring/cryptoKeys/my-key/cryptoKeyVersions/1
```

---

## Certificate Authority Service

```bash
# Create a CA pool
gcloud privateca pools create my-ca-pool \
  --location=us-central1 \
  --tier=DEVOPS

# Create a root CA
gcloud privateca roots create my-root-ca \
  --pool=my-ca-pool \
  --location=us-central1 \
  --subject="CN=My Root CA, O=My Org" \
  --key-algorithm=EC_P384_SHA384 \
  --auto-enable

# Issue a certificate
gcloud privateca certificates create my-cert \
  --issuer-pool=my-ca-pool \
  --issuer-location=us-central1 \
  --generate-key \
  --key-output-file=./key.pem \
  --cert-output-file=./cert.pem \
  --dns-san=my-service.internal \
  --validity=365d

# Revoke a certificate
gcloud privateca certificates revoke \
  --certificate=my-cert \
  --issuer-pool=my-ca-pool \
  --issuer-location=us-central1 \
  --reason=key_compromise
```

---

## Organization Policies

```bash
# List effective policies on a project
gcloud resource-manager org-policies list --project=my-project

# Enforce a constraint
gcloud resource-manager org-policies set-policy policy.yaml

# Common policy: require OS Login for all VMs
cat > policy.yaml << 'EOF'
name: projects/my-project/policies/compute.requireOsLogin
spec:
  rules:
    - enforce: true
EOF
gcloud resource-manager org-policies set-policy policy.yaml

# Common policy: restrict public IP on VMs
cat > policy.yaml << 'EOF'
name: projects/my-project/policies/compute.vmExternalIpAccess
spec:
  rules:
    - denyAll: true
EOF
gcloud resource-manager org-policies set-policy policy.yaml

# Common policy: disable SA key creation
cat > policy.yaml << 'EOF'
name: projects/my-project/policies/iam.disableServiceAccountKeyCreation
spec:
  rules:
    - enforce: true
EOF
gcloud resource-manager org-policies set-policy policy.yaml
```

---

## Audit Logging

```bash
# Enable data access audit logs (who read/wrote what data)
# Do this via the IAM policy for the project:
gcloud projects get-iam-policy my-project --format=json > policy.json
# Add auditConfigs for ADMIN_READ, DATA_READ, DATA_WRITE to policy.json, then:
gcloud projects set-iam-policy my-project policy.json

# Query audit logs
gcloud logging read \
  'logName="projects/my-project/logs/cloudaudit.googleapis.com%2Fdata_access" AND
   protoPayload.serviceName="storage.googleapis.com"' \
  --limit=20 \
  --format=json
```

---

## Guardrails

- **Never use primitive roles (`owner`/`editor`) in production.** Use predefined or custom roles.
- **Rotate Secret Manager secrets** — use `--rotation-period` and handle the rotation notification via Pub/Sub.
- **Always test VPC Service Controls in dry-run mode first** — enforcing too soon will break apps.
- **Enable SCC Standard or Premium tier** for threat detection (Event Threat Detection, Container Threat Detection).
- **Enable data access audit logs** for sensitive services (BigQuery, Cloud SQL, Secret Manager) — off by default.
- **Enforce `iam.disableServiceAccountKeyCreation`** at the org level and use Workload Identity instead.
- **Don't grant SA roles/editor or roles/owner** — it's essentially full access to the project.
