---
name: aws-networking
description: AWS networking — VPC, Security Groups, NACLs, ALB/NLB, CloudFront, Route 53, Transit Gateway, PrivateLink, VPC Flow Logs
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

# AWS Networking — Comprehensive Reference

## VPC Fundamentals

### Creating a VPC

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]' \
  --query 'Vpc.VpcId' --output text)

# Enable DNS hostnames (required for many services)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

# Enable DNS resolution
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
```

### Subnets

```bash
# Public subnets (one per AZ)
aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]'

# Private subnets
aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.11.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]'

# Enable auto-assign public IP for public subnet
aws ec2 modify-subnet-attribute \
  --subnet-id <public-subnet-id> --map-public-ip-on-launch
```

**CIDR planning tip:** Reserve /24 per AZ per tier: `10.0.{tier}{az}.0/24`
- 10.0.10.0/24 = public AZ-a, 10.0.20.0/24 = private AZ-a, 10.0.30.0/24 = data AZ-a

### Internet Gateway & NAT Gateway

```bash
# Internet Gateway (for public subnets)
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Public route table
RTB_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RTB_PUB --subnet-id <public-subnet-id>

# NAT Gateway (for private subnet outbound — costs ~$32/month + data)
EIP=$(aws ec2 allocate-address --query 'AllocationId' --output text)
NAT_ID=$(aws ec2 create-nat-gateway --subnet-id <public-subnet-id> \
  --allocation-id $EIP --query 'NatGateway.NatGatewayId' --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID

# Private route table (default route via NAT)
RTB_PRIV=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_PRIV \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_ID
aws ec2 associate-route-table --route-table-id $RTB_PRIV --subnet-id <private-subnet-id>
```

**Cost gotcha:** NAT Gateway charges per hour AND per GB. Use VPC endpoints for S3/DynamoDB to avoid NAT costs for those services.

---

## Security Groups vs NACLs

| Feature | Security Groups | NACLs |
|---|---|---|
| Level | Instance/ENI | Subnet |
| Stateful | Yes | No |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules | In order (lowest number first) |
| Default | Deny all in, allow all out | Allow all |

```bash
# Security Group
SG=$(aws ec2 create-security-group --group-name web-sg \
  --description "Web tier" --vpc-id $VPC_ID --query 'GroupId' --output text)

# Inbound rules
aws ec2 authorize-security-group-ingress --group-id $SG \
  --ip-permissions IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges=[{CidrIp=0.0.0.0/0}]

# Allow from another SG (preferred over CIDR for internal traffic)
aws ec2 authorize-security-group-ingress --group-id $APP_SG \
  --ip-permissions IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=$WEB_SG}]

# List rules
aws ec2 describe-security-group-rules --filters Name=group-id,Values=$SG \
  --query 'SecurityGroupRules[].[SecurityGroupRuleId,IsEgress,IpProtocol,FromPort,ToPort,CidrIpv4]' \
  --output table
```

**NACLs:** Number rules in increments of 10 (100, 110...) to leave room for insertion. Remember NACLs are stateless — you need both inbound and outbound rules for TCP.

---

## Load Balancers

### ALB (Application Load Balancer — Layer 7)

```bash
# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name prod-alb --type application \
  --subnets <pub-subnet-1a> <pub-subnet-1b> \
  --security-groups $SG \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

# Target group
TG_ARN=$(aws elbv2 create-target-group \
  --name web-targets --protocol HTTP --port 80 --vpc-id $VPC_ID \
  --health-check-path /health --health-check-interval-seconds 30 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Listener with HTTPS redirect
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=redirect,RedirectConfig="{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}"

aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=<acm-cert-arn> \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Register targets
aws elbv2 register-targets --target-group-arn $TG_ARN \
  --targets Id=i-0abc123 Id=i-0def456
```

### NLB (Network Load Balancer — Layer 4)

- Use NLB for: static IPs, ultra-low latency, TCP/UDP, PrivateLink source.
- NLB passes source IP by default; ALB requires X-Forwarded-For.
- NLB does not support security groups (use NACLs/target instance SGs).

---

## CloudFront

```bash
# Create distribution (CLI is verbose — prefer CDK/CF template)
aws cloudfront create-distribution --distribution-config file://cf-config.json

# Invalidate cache (costs after 1000 paths/month)
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"

# List distributions
aws cloudfront list-distributions \
  --query 'DistributionList.Items[].[Id,DomainName,Origins.Items[0].DomainName,Status]' \
  --output table
```

**Key behaviors:**
- **Origins:** S3, ALB, API Gateway, custom HTTP
- **Cache behaviors:** path-pattern matched, ordered (specific → default `*`)
- **OAC (Origin Access Control):** replaces OAI — use for private S3 origins
- **Response headers policy:** add HSTS, CORS headers at edge
- **Functions vs Lambda@Edge:** CloudFront Functions (<1ms, Viewer req/resp only), Lambda@Edge (heavier, all 4 events)

**Gotcha:** CloudFront caches 5xx from origin. Set `error-caching-minimum-ttl=0` for ALB origins if you want errors to pass through.

---

## Route 53

```bash
# List hosted zones
aws route53 list-hosted-zones --query 'HostedZones[].[Name,Id,Config.PrivateZone]' --output table

# Create record (upsert = create or update)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "<ALB-hosted-zone-id>",
          "DNSName": "<alb-dns-name>",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Health check
aws route53 create-health-check --caller-reference $(date +%s) \
  --health-check-config Type=HTTPS,FullyQualifiedDomainName=api.example.com,Port=443,ResourcePath=/health
```

### Routing Policies

| Policy | Use case |
|---|---|
| Simple | Single resource |
| Weighted | A/B testing, gradual migration (weight 0–255) |
| Latency | Route to lowest-latency region |
| Failover | Active/passive with health check |
| Geolocation | Country/continent based routing |
| Geoproximity | Lat/long with bias (Traffic Flow required) |
| Multivalue | Up to 8 healthy random records |

**Private hosted zones:** Associate with VPC for internal DNS (overlaps public zone).

---

## Transit Gateway

Use TGW when you have 3+ VPCs or hybrid connectivity (VPN/Direct Connect).

```bash
# Create TGW
TGW=$(aws ec2 create-transit-gateway \
  --description "Central hub" \
  --options AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable \
  --query 'TransitGateway.TransitGatewayId' --output text)

# Attach VPC
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW \
  --vpc-id $VPC_ID \
  --subnet-ids <subnet-1a> <subnet-1b>

# Route to TGW from spoke VPC route table
aws ec2 create-route --route-table-id $RTB \
  --destination-cidr-block 10.0.0.0/8 \
  --transit-gateway-id $TGW
```

---

## VPC Peering

Simpler than TGW for 2–3 VPCs in same or different accounts/regions. **Not transitive.**

```bash
# Create peering
PEER=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_A --peer-vpc-id $VPC_B \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# Accept (in peer account/region)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEER

# Add routes on both sides
aws ec2 create-route --route-table-id $RTB_A \
  --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PEER
```

---

## PrivateLink (VPC Endpoints)

```bash
# Gateway endpoint — S3 and DynamoDB only, free
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $RTB_PRIV --vpc-endpoint-type Gateway

# Interface endpoint — most services, costs ~$7/AZ/month
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids <priv-subnet-1a> <priv-subnet-1b> \
  --security-group-ids $SG --vpc-endpoint-type Interface \
  --private-dns-enabled  # lets code use normal SDK endpoints
```

---

## VPC Flow Logs

```bash
# Enable flow logs to CloudWatch
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123:role/FlowLogsRole

# Enable to S3 (cheaper for analytics)
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flowlogs-bucket/vpc/

# Query in CloudWatch Logs Insights
# fields @timestamp, srcAddr, dstAddr, dstPort, action
# | filter action = "REJECT"
# | stats count(*) by srcAddr
# | sort count desc | limit 20
```

---

## Guardrails & Gotchas

- **NACL rule ordering is critical** — rules evaluated by number, first match wins. Explicit DENY must come before any broader ALLOW.
- **Security groups are stateful** — return traffic is always allowed. NACLs are not.
- **NAT Gateway in each AZ** — one NAT per AZ for HA; cross-AZ NAT adds latency and cost.
- **Default VPC** — usable for dev, never for prod (public subnets with public IPs by default).
- **ALB idle timeout** — default 60s. Increase if your app has long-running requests.
- **CloudFront signed URLs vs signed cookies** — signed URLs for individual files, cookies for multiple files.
- **Route 53 TTL** — set low (60s) before migrations, restore after.
- **PrivateLink DNS** — `--private-dns-enabled` requires `enableDnsSupport` and `enableDnsHostnames` on VPC.
- **TGW is regional** — inter-region TGW peering exists but adds latency; consider Global Accelerator for user-facing.
