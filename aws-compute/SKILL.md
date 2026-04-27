---
name: aws-compute
description: AWS compute services — EC2, ECS (Fargate/EC2), EKS, Lambda, App Runner, Elastic Beanstalk, Lightsail
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

# AWS Compute — Comprehensive Reference

## EC2

### Instance Types (Quick Reference)

| Family | Optimized for | Examples |
|---|---|---|
| t3/t4g | Burstable general purpose | t3.micro, t4g.small |
| m6i/m7g | Balanced compute/memory | m6i.large, m7g.xlarge |
| c6i/c7g | Compute intensive | c6i.xlarge, c7g.2xlarge |
| r6i/r7g | Memory intensive | r6i.2xlarge (16 vCPU/128 GB) |
| g4dn/g5 | GPU (ML inference) | g4dn.xlarge (1x T4) |
| p3/p4d | GPU (ML training) | p3.8xlarge (4x V100) |
| i3en/i4i | Storage optimized NVMe | i3en.xlarge |
| inf2 | AWS Inferentia2 | inf2.xlarge |

```bash
# Launch instance
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.medium \
  --key-name my-keypair \
  --subnet-id <subnet-id> \
  --security-group-ids <sg-id> \
  --iam-instance-profile Name=EC2InstanceProfile \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-01}]' \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":30,"VolumeType":"gp3","Encrypted":true}}]'

# Find latest Amazon Linux 2023 AMI
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' --output text
```

### User Data

```bash
#!/bin/bash
# /etc/userdata.sh — runs as root on first boot
set -euo pipefail
dnf update -y
dnf install -y amazon-cloudwatch-agent aws-cli

# Write app config
cat > /etc/myapp.conf <<EOF
LOG_LEVEL=info
DB_HOST=${db_host}
EOF

systemctl enable --now myapp
```

**Gotcha:** User data runs once at launch. For re-running on each boot, add a `cloud-init` config or use `cloud-init-per` commands.

### Placement Groups

- **Cluster** — same AZ, same rack, <1ms latency (HPC, tightly coupled workloads)
- **Spread** — different hardware, up to 7 instances per AZ (small critical nodes)
- **Partition** — partitions on separate hardware (Hadoop, Cassandra, Kafka)

```bash
aws ec2 create-placement-group --group-name hpc-cluster --strategy cluster
aws ec2 run-instances ... --placement GroupName=hpc-cluster
```

### EBS Volumes

```bash
# gp3 is default: 3000 IOPS baseline, up to 16000 IOPS, $0.08/GB/month
# io2: up to 64000 IOPS, for databases
# st1: throughput optimized HDD, streaming workloads
# sc1: cold HDD, infrequent access

# Attach volume
aws ec2 attach-volume --volume-id vol-0abc --instance-id i-0abc --device /dev/sdf

# Resize (no downtime on Linux 4.x+)
aws ec2 modify-volume --volume-id vol-0abc --size 100
# Then on instance: sudo growpart /dev/xvda 1 && sudo xfs_growfs /

# Snapshot
aws ec2 create-snapshot --volume-id vol-0abc --description "backup $(date)"
```

---

## ECS (Elastic Container Service)

### Fargate vs EC2 Launch Type

| | Fargate | EC2 |
|---|---|---|
| Infrastructure | AWS manages | You manage nodes |
| Pricing | Per task vCPU/memory | Per instance |
| Scaling | Task-level | Instance + task level |
| Use case | Most workloads | GPU, huge tasks, cost optimization |

### Task Definition

```json
{
  "family": "web-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123:role/myTaskRole",
  "containerDefinitions": [{
    "name": "api",
    "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:latest",
    "portMappings": [{"containerPort": 8080}],
    "environment": [{"name": "LOG_LEVEL", "value": "info"}],
    "secrets": [{"name": "DB_PASS", "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:db-pass"}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/web-api",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "api"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
      "interval": 30, "timeout": 5, "retries": 3
    }
  }]
}
```

```bash
# Register task definition
aws ecs register-task-definition --cli-input-json file://task-def.json

# Create service
aws ecs create-service \
  --cluster prod-cluster \
  --service-name web-api \
  --task-definition web-api:3 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<priv-1a>,<priv-1b>],securityGroups=[$SG],assignPublicIp=DISABLED}" \
  --load-balancers targetGroupArn=$TG_ARN,containerName=api,containerPort=8080

# Update service (rolling deploy)
aws ecs update-service --cluster prod-cluster --service web-api \
  --task-definition web-api:4 \
  --force-new-deployment

# Exec into running container
aws ecs execute-command --cluster prod-cluster \
  --task <task-id> --container api \
  --command /bin/sh --interactive
```

**Gotcha:** `execute-command` requires `SSMSessionWorkerPlugin` installed locally and `enableExecuteCommand=true` on the service.

---

## EKS (Elastic Kubernetes Service)

```bash
# Create cluster
eksctl create cluster \
  --name prod \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name workers \
  --node-type m5.large \
  --nodes 3 --nodes-min 1 --nodes-max 10 \
  --managed \
  --with-oidc \
  --alb-ingress-access

# Update kubeconfig
aws eks update-kubeconfig --name prod --region us-east-1

# Add Fargate profile (for specific namespaces)
aws eks create-fargate-profile \
  --cluster-name prod \
  --fargate-profile-name fp-default \
  --pod-execution-role-arn arn:aws:iam::123:role/EKSFargatePodExecutionRole \
  --selectors '[{"namespace":"serverless"}]'

# Install cluster add-ons
aws eks create-addon --cluster-name prod --addon-name vpc-cni
aws eks create-addon --cluster-name prod --addon-name coredns
aws eks create-addon --cluster-name prod --addon-name kube-proxy
aws eks create-addon --cluster-name prod --addon-name aws-ebs-csi-driver

# Scale node group
aws eks update-nodegroup-config --cluster-name prod \
  --nodegroup-name workers \
  --scaling-config minSize=2,maxSize=10,desiredSize=4
```

### EKS IRSA (IAM Roles for Service Accounts)

```bash
# Create IAM role for SA
eksctl create iamserviceaccount \
  --cluster prod \
  --namespace default \
  --name my-app-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

---

## Lambda

### Runtimes & Configuration

```bash
# Deploy from zip
zip -j function.zip lambda/index.py
aws lambda create-function \
  --function-name my-processor \
  --runtime python3.12 \
  --handler index.handler \
  --role arn:aws:iam::123:role/LambdaRole \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 512 \
  --environment Variables={TABLE_NAME=my-table,LOG_LEVEL=info} \
  --architectures arm64    # Graviton2 — 20% cheaper, same performance

# Update code
aws lambda update-function-code \
  --function-name my-processor \
  --zip-file fileb://function.zip

# Update config
aws lambda update-function-configuration \
  --function-name my-processor \
  --memory-size 1024 --timeout 60

# Publish version + alias
aws lambda publish-version --function-name my-processor
aws lambda create-alias --function-name my-processor \
  --name prod --function-version 5

# Weighted alias for canary deploys
aws lambda update-alias --function-name my-processor --name prod \
  --routing-config AdditionalVersionWeights={"4"=0.1}   # 10% to v4
```

### Concurrency

```bash
# Reserved concurrency (limits & guarantees)
aws lambda put-function-concurrency \
  --function-name my-processor --reserved-concurrent-executions 100

# Provisioned concurrency (eliminate cold starts, costs money)
aws lambda put-provisioned-concurrency-config \
  --function-name my-processor \
  --qualifier prod \
  --provisioned-concurrent-executions 10
```

### Layers

```bash
# Publish layer
aws lambda publish-layer-version \
  --layer-name common-deps \
  --compatible-runtimes python3.12 \
  --zip-file fileb://layer.zip

# Attach to function
aws lambda update-function-configuration \
  --function-name my-processor \
  --layers arn:aws:lambda:us-east-1:123:layer:common-deps:3
```

### Destinations (async invocations)

```json
{
  "OnSuccess": {"Destination": "arn:aws:sqs:us-east-1:123:success-queue"},
  "OnFailure": {"Destination": "arn:aws:sqs:us-east-1:123:dlq"}
}
```

```bash
aws lambda put-function-event-invoke-config \
  --function-name my-processor \
  --maximum-retry-attempts 2 \
  --destination-config file://destinations.json
```

---

## App Runner

Fully managed service for containerized apps — no clusters, no task definitions.

```bash
# Deploy from ECR
aws apprunner create-service \
  --service-name my-api \
  --source-configuration '{
    "ImageRepository": {
      "ImageIdentifier": "123.dkr.ecr.us-east-1.amazonaws.com/api:latest",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {"Port": "8080"}
    },
    "AutoDeploymentsEnabled": true
  }' \
  --instance-configuration CpuUnit=1024,MemoryUnit=2048
```

Great for: microservices that need HTTPS auto-provisioning, auto-scaling to zero, no ops overhead.

---

## Elastic Beanstalk

```bash
# Init & deploy (Python example)
pip install awsebcli
eb init my-app --platform python-3.12 --region us-east-1
eb create prod-env --instance-type t3.small --min-instances 2 --max-instances 10
eb deploy
eb status
eb logs
eb ssh  # requires keypair
```

**Gotcha:** EB manages EC2, ALB, ASG for you — but it uses CloudFormation under the hood. Don't modify those CF stacks directly.

---

## Guardrails & Gotchas

- **EC2 termination protection** — enable on long-lived prod instances: `aws ec2 modify-instance-attribute --instance-id i-0abc --disable-api-termination`
- **Lambda cold starts** — arm64 has shorter cold starts than x86_64 for Python/Node. Provisioned concurrency eliminates them (at cost).
- **ECS task IAM vs execution role** — execution role: pull images, write logs. Task role: what your code can call (S3, DynamoDB, etc.).
- **EKS version skew** — kubelets must be within 2 minor versions of control plane. Update managed node groups after control plane.
- **Lambda 15-minute max timeout** — for longer jobs, use ECS/Fargate or Step Functions.
- **ECS Service Connect vs Service Discovery** — Service Connect (newer, built-in L7 LB and retries); prefer it over Cloud Map DNS for new services.
- **t3 unlimited mode** — CPU credits can accumulate costs. Monitor CPUCreditBalance and disable unlimited mode for baseline-only workloads.
- **EC2 instance metadata v2 (IMDSv2)** — enforce it: `aws ec2 modify-instance-metadata-options --instance-id i-0abc --http-tokens required`
