---
name: aws-cli
description: AWS CLI v2 patterns, profiles, SSO, JMESPath queries, pagination, waiters, and productivity aliases
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

# AWS CLI v2 — Comprehensive Reference

## Installation & Version Check

```bash
# Install on Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip /tmp/awscliv2.zip -d /tmp && sudo /tmp/aws/install

# Verify
aws --version   # aws-cli/2.x.x Python/3.x.x ...

# Upgrade
sudo /usr/local/bin/aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
```

---

## Profiles & Credentials

### File Locations

```
~/.aws/config       — named profiles, SSO config, region, output
~/.aws/credentials  — access key / secret (legacy; prefer SSO or roles)
```

### Static Credentials (legacy — avoid in production)

```ini
# ~/.aws/credentials
[default]
aws_access_key_id     = AKIA...
aws_secret_access_key = ...

[prod]
aws_access_key_id     = AKIA...
aws_secret_access_key = ...
```

### Named Profiles in config

```ini
# ~/.aws/config
[default]
region = us-east-1
output = json

[profile staging]
region = us-west-2
output = yaml

[profile prod]
role_arn       = arn:aws:iam::123456789012:role/DeployRole
source_profile = default
region         = eu-west-1
```

### Switching Profiles

```bash
# Per command
aws s3 ls --profile prod

# Per shell session
export AWS_PROFILE=staging

# Check current identity
aws sts get-caller-identity
```

---

## SSO (AWS IAM Identity Center)

### Setup

```ini
# ~/.aws/config
[profile sso-dev]
sso_start_url  = https://my-org.awsapps.com/start
sso_region     = us-east-1
sso_account_id = 111122223333
sso_role_name  = DeveloperAccess
region         = us-east-1
output         = json
```

```bash
# Authenticate (opens browser)
aws sso login --profile sso-dev

# Use the profile
aws s3 ls --profile sso-dev

# List all SSO accounts/roles available
aws sso list-accounts --access-token $(cat ~/.aws/sso/cache/*.json | python3 -c "import sys,json; print(json.load(sys.stdin)['accessToken'])")

# Logout
aws sso logout
```

**Gotcha:** SSO tokens expire (default 8h). Add `aws sso login --profile <p>` to your CI pre-step.

---

## Regions & Endpoints

```bash
# One-off override
aws ec2 describe-instances --region ap-southeast-1

# Environment variable
export AWS_DEFAULT_REGION=eu-central-1

# List all regions for a service
aws ec2 describe-regions --query 'Regions[].RegionName' --output text

# Use custom endpoint (e.g., LocalStack)
aws s3 ls --endpoint-url http://localhost:4566
```

---

## Output Formats

```bash
# json (default) — full, machine-parseable
aws ec2 describe-instances --output json

# yaml — human-friendly, less common in scripts
aws ec2 describe-instances --output yaml

# table — ASCII table, great for terminal browsing
aws ec2 describe-instances --output table

# text — tab-delimited, easiest to pipe to awk/cut
aws ec2 describe-instances --output text

# Set default in profile or env
export AWS_DEFAULT_OUTPUT=json
```

---

## --query with JMESPath

JMESPath filters and transforms output without jq.

```bash
# Get a single field
aws sts get-caller-identity --query 'Account' --output text

# Get array of fields
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name]' \
  --output table

# Filter by value
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?State.Name==`running`].InstanceId' \
  --output text

# Filter with tag
aws ec2 describe-instances \
  --query "Reservations[].Instances[?Tags[?Key=='Name'&&Value=='web-server']].InstanceId" \
  --output text

# Sort and limit (JMESPath sort_by)
aws ec2 describe-snapshots --owner-ids self \
  --query 'sort_by(Snapshots, &StartTime)[-5:].SnapshotId' \
  --output text

# Map to key-value object
aws ec2 describe-instances \
  --query 'Reservations[].Instances[]
           .{ID:InstanceId,Type:InstanceType,State:State.Name,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table
```

---

## Pagination

Large result sets are paginated automatically with `--no-paginate` / `--page-size`.

```bash
# Auto-paginate (CLI handles next tokens automatically — default)
aws s3api list-objects-v2 --bucket my-bucket

# Disable autopagination (return one page only)
aws s3api list-objects-v2 --bucket my-bucket --no-paginate

# Control page size (reduces memory, same total results)
aws ec2 describe-instances --page-size 10

# Manual pagination
aws ec2 describe-instances --max-items 5
# Use NextToken from output:
aws ec2 describe-instances --max-items 5 --starting-token <NextToken>

# Paginator via Python SDK (preferred in scripts)
# aws CLI wraps this automatically for most commands
```

---

## Waiter Commands

Waiters poll until a condition is met (up to a timeout).

```bash
# Wait for EC2 instance to be running
aws ec2 wait instance-running --instance-ids i-0abc123def456

# Wait for stack to finish creating
aws cloudformation wait stack-create-complete --stack-name my-stack

# Wait for S3 bucket to exist
aws s3api wait bucket-exists --bucket my-new-bucket

# Wait for ECS service to be stable
aws ecs wait services-stable --cluster prod --services web-api

# Wait for CodeBuild to finish
aws codebuild wait build-complete --ids <build-id>

# Custom polling via CLI loop (when no waiter exists)
until aws lambda get-function --function-name my-fn --query 'Configuration.State' \
      --output text | grep -q Active; do sleep 5; done
```

---

## Common Command Patterns

### EC2

```bash
# List running instances with Name tag
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[]
           .{ID:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,IP:PublicIpAddress}' \
  --output table

# Start/stop instances
aws ec2 start-instances  --instance-ids i-0abc123
aws ec2 stop-instances   --instance-ids i-0abc123

# Get SSM session (no SSH needed)
aws ssm start-session --target i-0abc123

# Get instance metadata from within EC2
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```

### S3

```bash
# List buckets
aws s3 ls

# Copy / sync
aws s3 cp file.txt s3://my-bucket/path/
aws s3 sync ./dist s3://my-bucket/static/ --delete --exclude "*.DS_Store"

# Presigned URL (valid 1h)
aws s3 presign s3://my-bucket/private/doc.pdf --expires-in 3600

# Recursive delete
aws s3 rm s3://my-bucket/old/ --recursive

# Check bucket size
aws s3api list-objects-v2 --bucket my-bucket \
  --query '[sum(Contents[].Size), length(Contents[])]' --output text
```

### IAM

```bash
# Who am I
aws sts get-caller-identity

# Assume a role
aws sts assume-role --role-arn arn:aws:iam::123:role/MyRole \
  --role-session-name my-session

# List policies attached to a role
aws iam list-attached-role-policies --role-name MyRole

# Simulate policy
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123:role/MyRole \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/*
```

---

## Useful Aliases & Shell Functions

```bash
# ~/.bashrc or ~/.zshrc

# Quick identity check
alias whoaws='aws sts get-caller-identity'

# Switch profile
awsp() { export AWS_PROFILE="$1"; echo "Switched to profile: $1"; }

# Login to SSO profile
awslogin() { aws sso login --profile "${1:-default}"; }

# EC2 instance list (table)
alias ec2ls='aws ec2 describe-instances \
  --query "Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==\`Name\`]|[0].Value,State:State.Name,Type:InstanceType,IP:PublicIpAddress}" \
  --output table'

# Tail CloudWatch log group
cwlogs() {
  aws logs tail "$1" --follow --format short
}

# Get secret value
secret() {
  aws secretsmanager get-secret-value --secret-id "$1" \
    --query SecretString --output text
}

# Empty and delete S3 bucket
s3nuke() {
  aws s3 rm "s3://$1" --recursive
  aws s3api delete-bucket --bucket "$1"
}
```

---

## Environment Variables Reference

| Variable | Purpose |
|---|---|
| `AWS_PROFILE` | Named profile to use |
| `AWS_DEFAULT_REGION` | Override region |
| `AWS_DEFAULT_OUTPUT` | Override output format |
| `AWS_ACCESS_KEY_ID` | Static credential (overrides profile) |
| `AWS_SECRET_ACCESS_KEY` | Static credential |
| `AWS_SESSION_TOKEN` | Temporary credential session token |
| `AWS_ROLE_ARN` | Role to assume automatically |
| `AWS_CA_BUNDLE` | Custom CA certificate bundle |
| `AWS_ENDPOINT_URL` | Override all service endpoints (e.g., LocalStack) |
| `AWS_RETRY_MODE` | `legacy`, `standard`, `adaptive` |
| `AWS_MAX_ATTEMPTS` | Max retry attempts |

---

## Guardrails & Gotchas

- **Never commit credentials** to git. Use SSO, IAM roles, or environment variables.
- `aws s3 rm --recursive` is irreversible — double-check the path.
- `--no-paginate` on large data sets can OOM your terminal. Use `--page-size` instead.
- JMESPath backticks (`` ` ``) must be escaped in double-quoted shell strings: use single quotes around the whole `--query` value.
- `aws ec2 terminate-instances` is permanent. Prefer `stop-instances` unless you mean it.
- SSO sessions expire — CI pipelines should use OIDC/role assumption, not SSO.
- `AWS_PROFILE` set in your shell affects all CLI calls including terraform/CDK — be intentional.
- Use `--dry-run` on destructive EC2 commands to verify permissions first: `aws ec2 terminate-instances --dry-run --instance-ids i-0abc`.
