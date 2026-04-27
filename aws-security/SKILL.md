---
name: aws-security
description: AWS security — IAM, KMS, Secrets Manager, GuardDuty, Security Hub, WAF, Config, CloudTrail, SCPs, Organizations
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

# AWS Security — Comprehensive Reference

## IAM

### Policy Types (Precedence Order)

1. **SCPs** (Organizations) — max permissions boundary for entire account
2. **Permission boundaries** — max permissions for a principal
3. **Identity-based policies** — attached to user/role/group
4. **Resource-based policies** — attached to S3, KMS, etc.
5. **Session policies** — passed at AssumeRole time

Effective permissions = intersection of all applicable allow policies, minus any explicit deny.

### Least-Privilege Policy Example

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowS3ReadInPrefix",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/data/*"
    ],
    "Condition": {
      "StringEquals": {"s3:prefix": ["data/"]},
      "Bool": {"aws:SecureTransport": "true"}
    }
  }]
}
```

### IAM Commands

```bash
# Who am I
aws sts get-caller-identity

# Create role with trust policy
aws iam create-role \
  --role-name AppRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach managed policy
aws iam attach-role-policy \
  --role-name AppRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create and attach inline policy
aws iam put-role-policy \
  --role-name AppRole \
  --policy-name DynamoReadOnly \
  --policy-document file://dynamo-policy.json

# List what a role can do
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123:role/AppRole \
  --action-names dynamodb:GetItem s3:PutObject \
  --resource-arns arn:aws:dynamodb:us-east-1:123:table/orders

# Generate credential report
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d

# List unused access keys (>90d)
aws iam list-users --query 'Users[].UserName' --output text | \
  xargs -I{} aws iam list-access-keys --user-name {} \
  --query 'AccessKeyMetadata[?Status==`Active`].[UserName,AccessKeyId,CreateDate]' --output text
```

### Permission Boundaries

```bash
# Create boundary policy (max permissions for delegated admins)
aws iam create-policy \
  --policy-name DevBoundary \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:*", "dynamodb:*", "lambda:*", "logs:*"],
      "Resource": "*"
    }]
  }'

# Attach boundary when creating role
aws iam create-role \
  --role-name DevRole \
  --assume-role-policy-document file://trust.json \
  --permissions-boundary arn:aws:iam::123:policy/DevBoundary
```

### IAM Identity Center (SSO)

```bash
# List permission sets
aws sso-admin list-permission-sets \
  --instance-arn arn:aws:sso:::instance/ssoins-xxx

# Assign permission set to user in account
aws sso-admin create-account-assignment \
  --instance-arn arn:aws:sso:::instance/ssoins-xxx \
  --target-id 123456789012 \
  --target-type AWS_ACCOUNT \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-xxx/ps-xxx \
  --principal-type USER \
  --principal-id <user-id>
```

---

## KMS

### Key Types

- **AWS Managed Keys** (`aws/s3`, `aws/rds`) — free, no direct control
- **Customer Managed Keys (CMK)** — $1/month, full control, auditable
- **External Key Material** — BYOK, you hold the key material

```bash
# Create CMK
KEY_ID=$(aws kms create-key \
  --description "Production app encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --query 'KeyMetadata.KeyId' --output text)

# Create alias
aws kms create-alias \
  --alias-name alias/prod-app-key \
  --target-key-id $KEY_ID

# Encrypt
aws kms encrypt \
  --key-id alias/prod-app-key \
  --plaintext fileb://secret.txt \
  --query 'CiphertextBlob' --output text | base64 -d > secret.enc

# Decrypt
aws kms decrypt \
  --ciphertext-blob fileb://secret.enc \
  --query 'Plaintext' --output text | base64 -d

# Key policy — allow specific role
aws kms get-key-policy --key-id $KEY_ID --policy-name default
# Edit and put-key-policy

# Key rotation (annual automatic)
aws kms enable-key-rotation --key-id $KEY_ID
aws kms get-key-rotation-status --key-id $KEY_ID

# Encryption context (integrity + auditability)
aws kms encrypt \
  --key-id alias/prod-app-key \
  --plaintext fileb://data.txt \
  --encryption-context Purpose=BackupEncryption,App=myapp
# MUST provide same context on decrypt
```

---

## Secrets Manager vs Parameter Store

| | Secrets Manager | Parameter Store |
|---|---|---|
| Cost | $0.40/secret/month | Free (standard) / $0.05/param/month (advanced) |
| Rotation | Built-in automatic rotation | Manual only |
| Max size | 65 KB | 4 KB (standard) / 8 KB (advanced) |
| X-account | Yes (resource policy) | No |
| Use for | DB passwords, API keys | Config, non-sensitive strings |

```bash
# Create secret
aws secretsmanager create-secret \
  --name prod/myapp/db-password \
  --description "Production DB password" \
  --secret-string "$(openssl rand -base64 32)"

# Get secret
aws secretsmanager get-secret-value \
  --secret-id prod/myapp/db-password \
  --query SecretString --output text

# Enable rotation (Lambda-based)
aws secretsmanager rotate-secret \
  --secret-id prod/myapp/db-password \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123:function:SecretsRotator \
  --rotation-rules AutomaticallyAfterDays=30

# Parameter Store
aws ssm put-parameter \
  --name /prod/myapp/db-host \
  --value "db.prod.internal" \
  --type SecureString \
  --key-id alias/aws/ssm

aws ssm get-parameter --name /prod/myapp/db-host --with-decryption \
  --query 'Parameter.Value' --output text

# Get all params under a path
aws ssm get-parameters-by-path --path /prod/myapp/ \
  --recursive --with-decryption \
  --query 'Parameters[].[Name,Value]' --output table
```

---

## GuardDuty

```bash
# Enable (each region independently)
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

DETECTOR=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)

# List findings
aws guardduty list-findings --detector-id $DETECTOR \
  --finding-criteria '{"Criterion":{"severity":{"Gte":7}}}'

# Get finding details
aws guardduty get-findings --detector-id $DETECTOR \
  --finding-ids <finding-id> \
  --query 'Findings[].[Title,Severity,Region,Service.Action.ActionType]' --output table

# Enable S3 protection, EKS protection
aws guardduty update-detector --detector-id $DETECTOR \
  --features '[{"Name":"S3_DATA_EVENTS","Status":"ENABLED"},{"Name":"EKS_AUDIT_LOGS","Status":"ENABLED"}]'
```

---

## Security Hub

```bash
# Enable
aws securityhub enable-security-hub --enable-default-standards

# Enable specific standards
aws securityhub batch-enable-standards \
  --standards-subscription-requests \
    StandardsArn=arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0 \
    StandardsArn=arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.2.0

# Get failed controls
aws securityhub get-findings \
  --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}],"RecordState":[{"Value":"ACTIVE","Comparison":"EQUALS"}]}' \
  --query 'Findings[].[Title,Severity.Label,ProductFields.ControlId]' --output table
```

---

## WAF (Web Application Firewall)

```bash
# Create WAF Web ACL (WAFv2)
aws wafv2 create-web-acl \
  --name prod-web-acl \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules file://waf-rules.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=prod-waf \
  --region us-east-1

# Associate with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn <acl-arn> \
  --resource-arn <alb-arn>

# Add AWS Managed Rules (common rule group)
# In rules JSON:
# {
#   "Name": "AWSManagedRulesCommonRuleSet",
#   "Priority": 1,
#   "OverrideAction": {"None": {}},
#   "Statement": {
#     "ManagedRuleGroupStatement": {
#       "VendorName": "AWS",
#       "Name": "AWSManagedRulesCommonRuleSet"
#     }
#   },
#   "VisibilityConfig": {...}
# }

# Rate-based rule (block >2000 req/5min per IP)
# "Statement": {"RateBasedStatement": {"Limit": 2000, "AggregateKeyType": "IP"}}
```

---

## AWS Config

```bash
# Enable Config recorder
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::123:role/ConfigRole \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

aws configservice put-delivery-channel \
  --delivery-channel name=default,s3BucketName=my-config-bucket

aws configservice start-configuration-recorder --configuration-recorder-name default

# Add managed rules
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "encrypted-volumes",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "ENCRYPTED_VOLUMES"
  }
}'

# Check compliance
aws configservice describe-compliance-by-config-rule \
  --query 'ComplianceByConfigRules[].[ConfigRuleName,Compliance.ComplianceType]' --output table

# Get non-compliant resources
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name encrypted-volumes \
  --compliance-types NON_COMPLIANT \
  --query 'EvaluationResults[].[EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId,ComplianceType]' \
  --output table
```

---

## CloudTrail

```bash
# Create organization trail (covers all accounts)
aws cloudtrail create-trail \
  --name org-trail \
  --s3-bucket-name my-cloudtrail-bucket \
  --is-organization-trail \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation

aws cloudtrail start-logging --name org-trail

# Look up recent events (last 90d, free)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket \
  --query 'Events[].[EventTime,Username,CloudTrailEvent]' --output table

# CloudTrail Lake — SQL queries on events
aws cloudtrail create-event-data-store \
  --name my-events \
  --retention-period 90 \
  --organization-enabled \
  --multi-region-enabled

aws cloudtrail start-query \
  --query-statement "SELECT eventName, userIdentity.arn, eventTime FROM my-events WHERE eventName = 'ConsoleLogin' AND errorCode = 'Failed authentication' ORDER BY eventTime DESC LIMIT 25"
```

---

## AWS Organizations & SCPs

```bash
# List accounts in org
aws organizations list-accounts \
  --query 'Accounts[].[Id,Name,Status]' --output table

# Create SCP (deny all in specific regions)
aws organizations create-policy \
  --name DenyNonApprovedRegions \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Deny",
      "NotAction": ["cloudfront:*","iam:*","route53:*","support:*"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
        }
      }
    }]
  }'

# Attach SCP to OU
aws organizations attach-policy \
  --policy-id p-xxxx \
  --target-id ou-xxxx-xxxxxxxx
```

---

## Guardrails & Gotchas

- **Root account** — enable MFA, disable access keys, don't use day-to-day. Period.
- **Explicit deny always wins** — even if identity policy allows, an SCP or resource policy deny blocks it.
- **KMS key deletion** — minimum 7-day waiting period. Lost key = lost data forever.
- **GuardDuty is regional** — must enable in every region you use (and us-east-1 for global services).
- **WAF Managed Rules count** — each rule group counts toward the 1500 WCU limit per ACL.
- **CloudTrail data events** — not enabled by default; costs $0.10/100k events. Enable for S3 and Lambda in security-sensitive environments.
- **IAM Access Analyzer** — run it; it finds resource-based policies granting external access you didn't intend.
- **Secrets Manager rotation** — test the rotation Lambda before enabling! Failed rotation locks out your app.
- **SCP FullAWSAccess** — must be attached to root/OU or all access is denied (deny by default at org level).
