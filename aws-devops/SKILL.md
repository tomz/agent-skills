---
name: aws-devops
description: AWS DevOps — CodePipeline, CodeBuild, CodeDeploy, ECR, GitHub Actions OIDC, CodeArtifact, deployment strategies
license: MIT
version: 1.0.0
allowed-tools:
  - shell
  - read_file
  - write_file
  - glob
  - grep
---

# AWS DevOps — Comprehensive Reference

## CodeBuild

### buildspec.yml

```yaml
version: 0.2
updated: 2026-04-24

env:
  variables:
    IMAGE_NAME: my-api
  secrets-manager:
    SONAR_TOKEN: prod/sonar:token
  parameter-store:
    ECR_REPO: /prod/ecr-repo-url

phases:
  install:
    runtime-versions:
      python: 3.12
      nodejs: 20
    commands:
      - pip install -r requirements-dev.txt

  pre_build:
    commands:
      - echo "Running tests..."
      - pytest tests/ --junitxml=test-results/junit.xml --cov=src --cov-report=xml
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION |
          docker login --username AWS --password-stdin $ECR_REPO

  build:
    commands:
      - IMAGE_TAG=${CODEBUILD_RESOLVED_SOURCE_VERSION::8}
      - docker build -t $ECR_REPO/$IMAGE_NAME:$IMAGE_TAG -t $ECR_REPO/$IMAGE_NAME:latest .
      - docker push $ECR_REPO/$IMAGE_NAME:$IMAGE_TAG
      - docker push $ECR_REPO/$IMAGE_NAME:latest
      - printf '[{"name":"api","imageUri":"%s"}]' "$ECR_REPO/$IMAGE_NAME:$IMAGE_TAG" > imagedefinitions.json

  post_build:
    commands:
      - echo "Build complete"

reports:
  pytest-coverage:
    files: ['coverage.xml']
    file-format: COBERTURAXML
  test-results:
    files: ['test-results/junit.xml']
    file-format: JUNITXML

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yaml
    - taskdef.json
  discard-paths: yes

cache:
  paths:
    - '/root/.cache/pip/**/*'
    - '/usr/local/lib/node_modules/**/*'
```

### CodeBuild CLI

```bash
# Start build
aws codebuild start-build --project-name my-api-build

# List recent builds
aws codebuild list-builds-for-project --project-name my-api-build \
  --query 'ids[:5]' --output text

# Get build status and logs
BUILD_ID="my-api-build:abc123"
aws codebuild batch-get-builds --ids $BUILD_ID \
  --query 'builds[].[id,buildStatus,phases[].phaseStatus]' --output table

# Tail logs
aws logs tail /aws/codebuild/my-api-build --follow
```

---

## CodeDeploy

### appspec.yaml (ECS)

```yaml
version: 0.0
updated: 2026-04-24
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: api
          ContainerPort: 8080
        PlatformVersion: LATEST
Hooks:
  - BeforeInstall: arn:aws:lambda:us-east-1:123:function:pre-deploy-hook
  - AfterInstall:  arn:aws:lambda:us-east-1:123:function:smoke-test
  - AfterAllowTestTraffic: arn:aws:lambda:us-east-1:123:function:integration-test
```

### appspec.yaml (EC2/On-Premises)

```yaml
version: 0.0
updated: 2026-04-24
os: linux
files:
  - source: /
    destination: /var/www/myapp
hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 60
  AfterInstall:
    - location: scripts/install_deps.sh
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 30
  ValidateService:
    - location: scripts/health_check.sh
      timeout: 60
```

### Deployment Types

| Strategy | Downtime | Rollback | Use case |
|---|---|---|---|
| In-place | Brief (rolling) | Slow (redeploy) | EC2, non-critical |
| Blue/Green | Zero | Instant (reroute) | ECS, Lambda, prod |
| Canary | Zero | Instant | Lambda aliases, gradual rollout |
| Linear | Zero | Instant | ECS, controlled ramp |

```bash
# Create deployment
aws deploy create-deployment \
  --application-name my-api \
  --deployment-group-name prod \
  --deployment-config-name CodeDeployDefault.ECSCanary10Percent5Minutes \
  --revision revisionType=S3,s3Location="{bucket=my-bucket,key=deploy.zip,bundleType=zip}"

# List deployments
aws deploy list-deployments \
  --application-name my-api \
  --deployment-group-name prod \
  --include-only-statuses Succeeded Failed \
  --query 'deployments' --output text

# Rollback to last successful
aws deploy stop-deployment --deployment-id d-ABC123 --auto-rollback-enabled
```

---

## CodePipeline

```bash
# Get pipeline state (see which stage is failing)
aws codepipeline get-pipeline-state --name my-pipeline \
  --query 'stageStates[].[stageName,latestExecution.status,actionStates[].actionName,actionStates[].latestExecution.status]' \
  --output table

# Start pipeline
aws codepipeline start-pipeline-execution --name my-pipeline

# Approve a manual approval action
TOKEN=$(aws codepipeline get-pipeline-state --name my-pipeline \
  --query 'stageStates[?stageName==`Approval`].actionStates[0].latestExecution.token' \
  --output text)

aws codepipeline put-approval-result \
  --pipeline-name my-pipeline \
  --stage-name Approval \
  --action-name ManualApproval \
  --token $TOKEN \
  --result summary="LGTM",status=Approved

# Retry failed stage
aws codepipeline retry-stage-execution \
  --pipeline-name my-pipeline \
  --stage-name Build \
  --pipeline-execution-id <exec-id> \
  --retry-mode FAILED_ACTIONS
```

---

## ECR (Elastic Container Registry)

```bash
# Create repo
aws ecr create-repository \
  --repository-name my-api \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS

# Login
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Push
docker tag my-api:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:latest

# Lifecycle policy (keep last 10 tagged, delete untagged after 7d)
aws ecr put-lifecycle-policy \
  --repository-name my-api \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Delete untagged after 7 days",
        "selection": {"tagStatus": "untagged", "countType": "sinceImagePushed", "countUnit": "days", "countNumber": 7},
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 2,
        "description": "Keep last 10 tagged images",
        "selection": {"tagStatus": "tagged", "tagPrefixList": ["v"], "countType": "imageCountMoreThan", "countNumber": 10},
        "action": {"type": "expire"}
      }
    ]
  }'

# Get scan results
aws ecr describe-image-scan-findings \
  --repository-name my-api \
  --image-id imageTag=latest \
  --query 'imageScanFindings.findings[?severity==`CRITICAL`].[name,severity,uri]' \
  --output table
```

---

## GitHub Actions for AWS (OIDC)

Prefer OIDC over long-lived access keys — no secrets to rotate.

### Setup IAM OIDC Provider

```bash
# One-time setup per account
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Create IAM role with trust policy
aws iam create-role \
  --role-name GitHubActionsDeployRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Federated": "arn:aws:iam::123:oidc-provider/token.actions.githubusercontent.com"},
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"},
        "StringLike": {"token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"}
      }
    }]
  }'
```

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write   # required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: us-east-1

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        run: |
          docker build -t $ECR_REGISTRY/my-api:${{ github.sha }} .
          docker push $ECR_REGISTRY/my-api:${{ github.sha }}
        env:
          ECR_REGISTRY: 123456789012.dkr.ecr.us-east-1.amazonaws.com

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: task-def.json
          service: my-api
          cluster: prod
          wait-for-service-stability: true
          codedeploy-appspec: appspec.yaml
          codedeploy-application: my-api
          codedeploy-deployment-group: prod
```

---

## CodeArtifact

```bash
# Create domain and repo
aws codeartifact create-domain --domain myorg
aws codeartifact create-repository \
  --domain myorg --repository pypi-store \
  --description "PyPI proxy"

# Create upstream connection to public PyPI
aws codeartifact associate-external-connection \
  --domain myorg --repository pypi-store \
  --external-connection public:pypi

# Login (configures pip)
aws codeartifact login --tool pip --domain myorg --repository pypi-store

# Publish private package
aws codeartifact login --tool twine --domain myorg --repository pypi-store
twine upload dist/*
```

---

## Deployment Strategies — Decision Guide

```
Need zero downtime?
├── No  → In-place rolling (EC2, EB) — simplest
└── Yes
    ├── Lambda/API Gateway? → Canary aliases (CodeDeploy)
    ├── ECS?                → Blue/Green (CodeDeploy + ALB)
    └── EC2 fleet?          → Blue/Green with ASG swap
            └── Gradual rollout needed?
                ├── Yes → Linear10PercentEvery1Minute
                └── No  → AllAtOnce (with instant rollback)
```

---

## Guardrails & Gotchas

- **CodeBuild privileged mode** — required for Docker builds. Only enable when needed; it grants root on the build host.
- **GitHub Actions OIDC condition** — always restrict `sub` to your repo. `repo:myorg/*:*` is dangerously broad.
- **CodeDeploy ECS blue/green** — you need two target groups and a test listener port on the ALB. Forgetting these is the #1 setup error.
- **ECR image mutability** — set `imageTagMutability=IMMUTABLE` for prod to prevent tag overwriting. Breaks pipelines that push `:latest` — fix pipelines first.
- **CodePipeline artifact S3** — artifacts bucket must be in the same region as the pipeline.
- **CodeBuild VPC mode** — needed to access RDS, ElastiCache, internal services. Requires NAT or VPC endpoints for ECR/S3.
- **Lambda canary rollback** — CodeDeploy rolls back automatically if the CloudWatch alarm you attach triggers. Wire up your error rate alarm.
- **CodeArtifact token expiry** — tokens last 12h. In long CI jobs, refresh before `pip install` or `npm install`.
