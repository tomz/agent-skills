---
name: aws-iac
description: Infrastructure as Code for AWS — CloudFormation, CDK (TypeScript/Python), Terraform, SAM, and Rain CLI
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

# AWS Infrastructure as Code — Comprehensive Reference

## CloudFormation

### Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: My stack

Parameters:
  Env:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev

Mappings:
  EnvConfig:
    dev:   { InstanceType: t3.micro }
    prod:  { InstanceType: m5.large }

Conditions:
  IsProd: !Equals [!Ref Env, prod]

Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain          # keep on stack delete
    UpdateReplacePolicy: Retain
    Properties:
      BucketName: !Sub "my-app-${Env}-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled

  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !FindInMap [EnvConfig, !Ref Env, InstanceType]
      ImageId: !If [IsProd, ami-prod123, ami-dev456]

Outputs:
  BucketName:
    Value: !Ref MyBucket
    Export:
      Name: !Sub "${AWS::StackName}-BucketName"
```

### Stack Operations

```bash
# Validate template
aws cloudformation validate-template --template-body file://template.yaml

# Deploy (create or update)
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-stack \
  --parameter-overrides Env=prod \
  --capabilities CAPABILITY_NAMED_IAM \
  --tags Project=myapp Environment=prod

# Create change set before applying
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name my-changes \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM

aws cloudformation describe-change-set \
  --stack-name my-stack --change-set-name my-changes

aws cloudformation execute-change-set \
  --stack-name my-stack --change-set-name my-changes

# Drift detection
aws cloudformation detect-stack-drift --stack-name my-stack
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <id>
aws cloudformation describe-stack-resource-drifts --stack-name my-stack \
  --stack-resource-drift-status-filters MODIFIED DELETED

# View stack events (useful for failed deploys)
aws cloudformation describe-stack-events --stack-name my-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`||ResourceStatus==`UPDATE_FAILED`]
           .[LogicalResourceId,ResourceStatusReason]' \
  --output table

# Delete (with retained resources)
aws cloudformation delete-stack --stack-name my-stack
aws cloudformation wait stack-delete-complete --stack-name my-stack

# List stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query 'StackSummaries[].[StackName,StackStatus,LastUpdatedTime]' \
  --output table
```

### Nested Stacks

```yaml
# Parent stack references child templates in S3
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub "https://s3.amazonaws.com/${ArtifactBucket}/network.yaml"
      Parameters:
        VpcCidr: 10.0.0.0/16
      TimeoutInMinutes: 30

  AppStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: !Sub "https://s3.amazonaws.com/${ArtifactBucket}/app.yaml"
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
```

**Gotcha:** Nested stack updates require re-uploading child templates to S3 first.

---

## AWS CDK

### Setup

```bash
npm install -g aws-cdk
cdk --version

# Bootstrap (once per account/region)
cdk bootstrap aws://ACCOUNT_ID/us-east-1

# Init project
cdk init app --language typescript
cdk init app --language python
```

### TypeScript — Construct Levels

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as iam from 'aws-cdk-lib/aws-iam';

export class MyStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // L1 — raw CloudFormation resource (CfnBucket)
    const cfnBucket = new s3.CfnBucket(this, 'RawBucket', {
      versioningConfiguration: { status: 'Enabled' },
    });

    // L2 — opinionated construct with sensible defaults
    const bucket = new s3.Bucket(this, 'AppBucket', {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
    });

    // L3 — pattern construct (high level)
    const fn = new lambda.Function(this, 'MyFn', {
      runtime: lambda.Runtime.PYTHON_3_12,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('./lambda'),
      environment: { BUCKET: bucket.bucketName },
    });

    bucket.grantRead(fn);  // CDK auto-generates least-privilege IAM policy

    new cdk.CfnOutput(this, 'BucketName', { value: bucket.bucketName });
  }
}
```

### Python CDK

```python
import aws_cdk as cdk
from aws_cdk import aws_s3 as s3, aws_lambda as lambda_, Stack
from constructs import Construct

class MyStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        bucket = s3.Bucket(self, "AppBucket",
            versioned=True,
            removal_policy=cdk.RemovalPolicy.DESTROY,
            auto_delete_objects=True,   # custom resource to empty bucket first
        )

        fn = lambda_.Function(self, "Processor",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="index.handler",
            code=lambda_.Code.from_asset("./src"),
        )
        bucket.grant_read_write(fn)
```

### CDK Commands

```bash
cdk synth                         # synthesize CloudFormation template
cdk synth MyStack                 # specific stack
cdk diff                          # compare with deployed stack
cdk diff --security-only          # only show IAM/SG changes
cdk deploy                        # deploy all stacks
cdk deploy MyStack --require-approval never   # CI mode
cdk deploy --hotswap              # skip CF for Lambda/ECS updates (dev only!)
cdk destroy MyStack               # delete stack
cdk ls                            # list stacks
cdk doctor                        # check environment setup
cdk context --clear               # clear cached context (AMI IDs, AZs, etc.)
```

**Gotcha:** `--hotswap` bypasses CloudFormation. Never use in prod — it leaves drift.

---

## Terraform (AWS Provider)

### Basic Pattern

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

provider "aws" {
  region  = var.region
  profile = var.aws_profile
  default_tags {
    tags = { Project = "myapp", ManagedBy = "terraform" }
  }
}

resource "aws_s3_bucket" "app" {
  bucket = "my-app-${var.env}-${data.aws_caller_identity.current.account_id}"
}

data "aws_caller_identity" "current" {}

output "bucket_arn" { value = aws_s3_bucket.app.arn }
```

```bash
terraform init          # download providers
terraform plan          # show changes
terraform apply         # apply (prompts)
terraform apply -auto-approve   # CI mode
terraform destroy
terraform state list    # list managed resources
terraform import aws_s3_bucket.app my-existing-bucket
terraform taint aws_s3_bucket.app   # force recreation
```

---

## AWS SAM (Serverless Application Model)

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.12
    Timeout: 30
    Environment:
      Variables:
        TABLE_NAME: !Ref Table

Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      CodeUri: src/
      Events:
        Api:
          Type: Api
          Properties:
            Path: /items
            Method: get

  Table:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: id
        Type: String
```

```bash
# Install SAM CLI
pip install aws-sam-cli

sam build                          # build/package
sam local invoke ApiFunction       # run locally
sam local start-api                # local API Gateway on :3000
sam local start-lambda             # local Lambda endpoint
sam deploy --guided                # first deploy (creates samconfig.toml)
sam deploy                         # subsequent deploys
sam logs -n ApiFunction --tail     # stream logs
sam delete --stack-name my-sam-app
```

---

## Rain CLI (CloudFormation helper)

```bash
# Install
go install github.com/aws-cloudformation/rain/cmd/rain@latest
# or: brew install rain

# Deploy (Rain handles packaging, S3 upload, waiting)
rain deploy template.yaml my-stack

# Format template (canonicalize YAML)
rain fmt template.yaml

# Diff deployed vs local
rain diff my-stack template.yaml

# Check stack status
rain ls
rain ls my-stack

# Build templates from modules
rain build s3://my-bucket s3-module.yaml > template.yaml
```

---

## IaC Best Practices

1. **Lock provider/CDK versions** — pin minor versions to avoid surprise breaking changes.
2. **Remote state** — always use S3 + DynamoDB for Terraform; never local tfstate in repos.
3. **Separate stacks by lifecycle** — network (rarely changes) vs app (deploys often) vs data (almost never).
4. **Use change sets / `cdk diff` / `terraform plan`** — always review before apply in prod.
5. **Tag everything** — use `default_tags` in Terraform, `Tags` in CDK App props, `aws:cloudformation:stack-name` is auto-added.
6. **DeletionPolicy: Retain** — set on stateful resources (RDS, S3, DynamoDB) to prevent accidental destruction.
7. **Drift detection** — run periodically; manual console changes break IaC trust.
8. **Secrets in Parameter Store/Secrets Manager** — never hardcode in templates.
9. **CDK Aspects** — use for cross-cutting concerns (enforce encryption, tag compliance).
10. **`cdk destroy` in CI** — only on ephemeral envs; production stacks should require human approval.
