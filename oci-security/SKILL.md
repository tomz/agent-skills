---
name: oci-security
description: OCI security — IAM with Identity Domains, policies, Vault, Cloud Guard, Bastion, Security Zones, WAF
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

# OCI Security

## IAM with Identity Domains

OCI IAM (Identity and Access Management) uses **Identity Domains** to manage users, groups, and federation.

### Identity Domain Concepts
- **Identity Domain** — replaces legacy IDCS. Default domain created with tenancy.
- **Users** — can be IAM users (local) or federated (SAML/OIDC IdP)
- **Groups** — collection of users
- **Dynamic Groups** — match compute instances, functions, etc. by rules
- **Policies** — grant permissions to groups/dynamic groups in compartments

```bash
# List identity domains
oci iam domain list --compartment-id <tenancy-ocid> --all

# List users in default domain
oci iam user list --all --output table \
  --query 'data[*].{"Name":"name","Email":"email","State":"lifecycle-state"}'

# Create user
oci iam user create \
  --name "jane.doe@example.com" \
  --description "Jane Doe" \
  --email "jane.doe@example.com"

# Create group
oci iam group create --name "NetworkAdmins" --description "Network team"

# Add user to group
oci iam group add-user \
  --group-id $GROUP_ID \
  --user-id $USER_ID

# Reset user password (generates temporary one-time password)
oci iam user create-or-reset-ui-password --user-id $USER_ID
```

### Dynamic Groups
```bash
# Dynamic group matching ALL compute instances in a compartment
oci iam dynamic-group create \
  --name "ComputeInstancesDG" \
  --description "All instances in prod compartment" \
  --matching-rule "ALL {instance.compartment.id = '$C'}"

# Match specific resource type
oci iam dynamic-group create \
  --name "FunctionsDG" \
  --description "OCI Functions" \
  --matching-rule "ALL {resource.type = 'fnfunc', resource.compartment.id = '$C'}"

# Match by tag
oci iam dynamic-group create \
  --name "BackupInstancesDG" \
  --description "Tagged instances" \
  --matching-rule "tag.Environment.Value = 'production'"
```

## IAM Policy Language

OCI policy statements follow a strict syntax:

```
Allow <subject> to <verb> <resource-type> in <location> [where <conditions>]
```

**Verbs (hierarchical — each includes all verbs below):**
- `manage` — full CRUD + admin actions
- `use` — create/update/delete instances/resources (no IAM)
- `read` — list + inspect details
- `inspect` — list only (no sensitive data)

```bash
# Create policy
oci iam policy create \
  --compartment-id $C \
  --name "NetworkAdminPolicy" \
  --description "Allow network admins to manage networking" \
  --statements '[
    "Allow group NetworkAdmins to manage virtual-network-family in compartment MyCompartment",
    "Allow group NetworkAdmins to read instances in compartment MyCompartment"
  ]'
```

### Common Policy Patterns

```hcl
# Allow group to manage all resources in a compartment
"Allow group Admins to manage all-resources in compartment dev"

# Allow instances to write logs (via dynamic group)
"Allow dynamic-group ComputeInstancesDG to use log-content in compartment MyCompartment"

# Allow Functions to read secrets from Vault
"Allow dynamic-group FunctionsDG to read secret-family in compartment security"

# Allow tenancy-level service permissions
"Allow service objectstorage-us-ashburn-1 to manage object-family in tenancy"

# Allow specific users (by OCID) — less common but useful for break-glass accounts
"Allow any-user to manage all-resources in tenancy where request.user.id = 'ocid1.user...'"

# Cross-compartment access
"Allow group DataScientists to use data-science-family in compartment ml-workspace"
"Allow group DataScientists to manage object-family in compartment ml-data"

# Condition-based: only from specific IP
"Allow group Ops to manage instance-family in compartment prod where request.networkSource.name = 'CorporateVPN'"

# MFA required condition
"Allow group Admins to manage all-resources in tenancy where request.user.mfaTotpVerified = 'true'"

# Resource tags condition
"Allow group Devs to manage instance-family in compartment dev where target.resource.tag.Environment = 'dev'"
```

### Network Sources (for IP-restricted policies)
```bash
oci iam network-source create \
  --name "CorporateVPN" \
  --description "Corporate VPN CIDR" \
  --public-source-list '["203.0.113.0/24","198.51.100.0/24"]' \
  --virtual-source-list '[{"vcnId":"'$VCN_ID'","ipRanges":["10.0.0.0/16"]}]'
```

## OCI Vault (Key Management & Secrets)

### Master Encryption Keys
```bash
# Create a Vault
VAULT_ID=$(oci kms management vault create \
  --compartment-id $C \
  --display-name "prod-vault" \
  --vault-type "DEFAULT" \
  --query 'data.id' --raw-output)

# Get management endpoint
MGMT_ENDPOINT=$(oci kms management vault get --vault-id $VAULT_ID \
  --query 'data."management-endpoint"' --raw-output)

# Create AES-256 master encryption key (MEK)
KEY_ID=$(oci kms management key create \
  --compartment-id $C \
  --display-name "prod-aes-key" \
  --key-shape '{"algorithm":"AES","length":32}' \
  --endpoint $MGMT_ENDPOINT \
  --query 'data.id' --raw-output)

# Create RSA key pair (for signing)
oci kms management key create \
  --compartment-id $C \
  --display-name "signing-key" \
  --key-shape '{"algorithm":"RSA","length":2048}' \
  --endpoint $MGMT_ENDPOINT

# Encrypt data
CRYPTO_ENDPOINT=$(oci kms management vault get --vault-id $VAULT_ID \
  --query 'data."crypto-endpoint"' --raw-output)

oci kms crypto encrypt \
  --key-id $KEY_ID \
  --plaintext "$(echo -n 'my secret data' | base64)" \
  --endpoint $CRYPTO_ENDPOINT

# Decrypt
oci kms crypto decrypt \
  --key-id $KEY_ID \
  --ciphertext "<encrypted-base64-blob>" \
  --endpoint $CRYPTO_ENDPOINT \
  --query 'data.plaintext' --raw-output | base64 -d
```

### Secrets
```bash
# Create secret (stores credentials, API keys, etc.)
oci vault secret create-base64 \
  --compartment-id $C \
  --vault-id $VAULT_ID \
  --key-id $KEY_ID \
  --secret-name "db-password" \
  --secret-content-content "$(echo -n 'MyStr0ng#Pass!' | base64)"

# Get secret value
SECRET_ID=$(oci vault secret list --compartment-id $C \
  --query "data[?\"secret-name\"=='db-password'].id | [0]" --raw-output)

oci secrets secretbundle get --secret-id $SECRET_ID \
  --query 'data.secret-bundle-content.content' --raw-output | base64 -d

# Rotate secret (create new version)
oci vault secret update-base64 \
  --secret-id $SECRET_ID \
  --secret-content-content "$(echo -n 'NewStr0ng#Pass!' | base64)"

# List secret versions
oci vault secret-version list --secret-id $SECRET_ID
```

```python
# Python: read secret at runtime
import oci, base64

signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
client = oci.secrets.SecretsClient({}, signer=signer)

bundle = client.get_secret_bundle(secret_id=SECRET_ID)
value = base64.b64decode(bundle.data.secret_bundle_content.content).decode()
```

## Cloud Guard

Cloud Guard detects security threats and can auto-remediate them.

```bash
# Enable Cloud Guard (tenant-level, one time)
oci cloud-guard configuration update \
  --compartment-id <tenancy-ocid> \
  --status ENABLED \
  --reporting-region us-ashburn-1

# Create Cloud Guard target (monitors a compartment)
oci cloud-guard target create \
  --compartment-id $C \
  --display-name "prod-target" \
  --target-resource-id $C \
  --target-resource-type "COMPARTMENT" \
  --target-detector-recipes '[{"detectorRecipeId":"<oracle-managed-detector-recipe-id>"}]' \
  --target-responder-recipes '[{"responderRecipeId":"<oracle-managed-responder-recipe-id>"}]'

# List problems (detected threats)
oci cloud-guard problem list \
  --compartment-id $C \
  --all \
  --lifecycle-state OPEN \
  --output table \
  --query 'data[*].{"Rule":"detector-rule-id","Severity":"risk-level","Resource":"resource-name","Time":"time-first-detected"}'

# Dismiss a problem (false positive)
oci cloud-guard problem update \
  --problem-id $PROBLEM_ID \
  --lifecycle-state DISMISSED \
  --comment "Reviewed — false positive, known test resource"

# List all open critical/high problems
oci cloud-guard problem list --compartment-id $C --all \
  --query 'data[?contains(`["CRITICAL","HIGH"]`, "risk-level")]'
```

## Bastion Service

OCI Bastion provides ephemeral SSH access to private resources without a jump box.

```bash
# Create bastion
BASTION_ID=$(oci bastion bastion create \
  --compartment-id $C \
  --bastion-type "STANDARD" \
  --target-subnet-id $PRIVATE_SUBNET_ID \
  --client-cidr-block-allow-list '["YOUR_IP/32"]' \
  --display-name "prod-bastion" \
  --query 'data.id' --raw-output)

# Create SSH port-forwarding session (tunnel to private instance)
SESSION_ID=$(oci bastion session create-port-forwarding \
  --bastion-id $BASTION_ID \
  --display-name "admin-session" \
  --target-private-ip "10.0.2.50" \
  --target-port 22 \
  --session-ttl-in-seconds 3600 \
  --ssh-public-key-file ~/.ssh/id_rsa.pub \
  --query 'data.id' --raw-output)

# Get SSH tunnel command
oci bastion session get --session-id $SESSION_ID \
  --query 'data."ssh-metadata"."command"' --raw-output

# Typical tunnel command (output from above):
# ssh -i ~/.ssh/id_rsa -N -L 2222:10.0.2.50:22 \
#   -p 22 ocid1.bastionsession...@host.bastion.us-ashburn-1.oci.oraclecloud.com

# Once tunnel is up, connect to private instance
ssh -i ~/.ssh/id_rsa -p 2222 opc@localhost
```

## Security Zones

Security Zones enforce security policies — deny configurations that violate them.

```bash
# List security zone recipes
oci cloud-guard security-zone security-zone-recipe list --compartment-id $C

# Create security zone (prevents insecure configs like public object storage)
oci cloud-guard security-zone security-zone create \
  --compartment-id $C \
  --display-name "prod-security-zone" \
  --security-zone-recipe-id <recipe-ocid> \
  --description "Production compartment with strict security"
```

## Vulnerability Scanning

```bash
# Create host scan recipe
oci vulnerability-scanning host-scan-recipe create \
  --compartment-id $C \
  --display-name "host-scan-weekly" \
  --schedule '{"type":"WEEKLY","dayOfWeek":"SUNDAY"}' \
  --agent-settings '{"scanLevel":"STANDARD","agentConfigurationVendor":"OCI"}' \
  --port-settings '{"scanLevel":"STANDARD"}'

# Create scan target (all instances in compartment)
oci vulnerability-scanning host-scan-target create \
  --compartment-id $C \
  --display-name "all-instances" \
  --host-scan-recipe-id <recipe-ocid> \
  --target-compartment-id $C

# List vulnerabilities found
oci vulnerability-scanning host-vulnerability list \
  --compartment-id $C --all \
  --query 'data[?"severity"==`CRITICAL`]' --output table
```

## Certificates Service

```bash
# Import existing certificate (e.g., Let's Encrypt)
oci certs-mgmt certificate create-certificate-issued-by-internal-ca \
  --compartment-id $C \
  --name "myapp-cert" \
  --certificate-profile-type "TLS_SERVER_OR_CLIENT"

# Or import external certificate
oci certs-mgmt certificate create-certificate-by-importing-config \
  --compartment-id $C \
  --name "imported-cert" \
  --certificate-pem "$(cat fullchain.pem)" \
  --private-key-pem "$(cat privkey.pem)" \
  --cert-chain-pem "$(cat chain.pem)"
```

## Gotchas & Tips

- **Policy inheritance** — policies in a compartment apply to all sub-compartments. Put broad policies high in the hierarchy; restrict with more specific policies below.
- **Policy propagation delay** — new IAM policies can take up to 60 seconds to take effect. Don't immediately test after creating.
- **`manage` vs `use`** — `manage` includes creating and deleting IAM policies. Giving `manage all-resources` means the group can escalate their own privileges. Use sparingly.
- **Dynamic group matching** — rules are OR'd together by default within `ANY{}`, AND'd within `ALL{}`. Test with `oci iam dynamic-group get` to verify instance membership.
- **Vault deletion is delayed** — you can't delete a vault immediately. It enters a "SCHEDULED_FOR_DELETION" state with a configurable delay (min 7 days). Secrets must all be deleted first.
- **Cloud Guard responders** — auto-remediation responders can stop instances, remove public IPs, etc. Test in non-prod before enabling in production.
- **Bastion TTL** — sessions expire after `session-ttl-in-seconds` (max 10800 = 3 hours). Plan for this in automation scripts.
- **Security Zone restrictions** — once applied, you cannot create insecure resources in that compartment. Moving existing non-compliant resources there will fail. Audit first.
