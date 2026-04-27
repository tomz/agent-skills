---
name: gcp-networking
description: GCP networking — VPC, Load Balancing, Cloud CDN, DNS, Armor, NAT, Interconnect, VPN, VPC Service Controls
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, glob, grep
---

# GCP Networking Skill

Design, deploy, and secure GCP network infrastructure.

---

## VPC Networks

GCP VPCs are **global** — subnets are regional. This is different from AWS (regional VPCs).

```bash
# Create a custom-mode VPC (always use custom — auto-mode creates subnets in every region)
gcloud compute networks create my-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=global

# Create a subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.10.0.0/24 \
  --enable-private-ip-google-access

# Enable Private Google Access (allows VMs without external IPs to reach Google APIs)
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access

# List subnets
gcloud compute networks subnets list --network=my-vpc

# Expand a subnet range (can only grow, never shrink)
gcloud compute networks subnets expand-ip-range my-subnet \
  --region=us-central1 \
  --prefix-length=23
```

### Secondary IP Ranges (for GKE)

```bash
gcloud compute networks subnets create gke-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.10.0.0/24 \
  --secondary-range=pods=10.100.0.0/16,services=10.200.0.0/20
```

---

## Firewall Rules

GCP firewall rules are **network-level** and use **tags** or **service accounts** for targeting.

```bash
# Allow SSH from IAP (Identity-Aware Proxy) — best practice instead of 0.0.0.0/0
gcloud compute firewall-rules create allow-ssh-iap \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=ssh-allowed

# Allow internal traffic within VPC
gcloud compute firewall-rules create allow-internal \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp,udp,icmp \
  --source-ranges=10.10.0.0/16

# Allow health checks from LB probers
gcloud compute firewall-rules create allow-health-checks \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=lb-backend

# Deny all ingress (explicit deny — lower priority wins, lower number = higher priority)
gcloud compute firewall-rules create deny-all-ingress \
  --network=my-vpc \
  --direction=INGRESS \
  --action=DENY \
  --rules=all \
  --priority=65534

# List firewall rules
gcloud compute firewall-rules list --filter="network=my-vpc" \
  --format="table(name,direction,priority,sourceRanges,targetTags,allowed)"

# Enable firewall logging on a rule
gcloud compute firewall-rules update allow-ssh-iap --enable-logging
```

**Gotcha:** GCP has implicit **deny-all ingress** and **allow-all egress**. You don't
need to create a deny-all rule unless you're overriding it with specific allows at
a higher priority.

---

## Shared VPC

Shared VPC lets host project share subnets with service projects.

```bash
# Enable Shared VPC on host project
gcloud compute shared-vpc enable HOST_PROJECT_ID

# Associate a service project
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
  --host-project=HOST_PROJECT_ID

# Grant subnet-level IAM in the host project
gcloud compute networks subnets add-iam-policy-binding my-subnet \
  --region=us-central1 \
  --member=serviceAccount:SERVICE_PROJECT_NUMBER@cloudservices.gserviceaccount.com \
  --role=roles/compute.networkUser \
  --project=HOST_PROJECT_ID
```

---

## Cloud Load Balancing

GCP load balancers are categorized by traffic type and scope.

| LB Type | Use Case |
|---------|---------|
| Global External HTTP(S) | Public web apps, HTTPS, global Anycast |
| Regional External HTTP(S) | Public web apps, regional |
| Internal HTTP(S) | Internal microservices (Layer 7) |
| External TCP/UDP Network | Non-HTTP public traffic |
| Internal TCP/UDP | Internal non-HTTP traffic |
| External SSL Proxy | SSL termination for non-HTTP |

### HTTP(S) Load Balancer (Global)

```bash
# 1. Reserve a global static IP
gcloud compute addresses create web-lb-ip --global

# 2. Create a backend service
gcloud compute backend-services create web-backend \
  --global \
  --protocol=HTTP \
  --health-checks=web-health-check \
  --load-balancing-scheme=EXTERNAL_MANAGED

# 3. Add instance group or NEG to backend
gcloud compute backend-services add-backend web-backend \
  --global \
  --instance-group=my-mig \
  --instance-group-zone=us-central1-a \
  --balancing-mode=UTILIZATION

# 4. Create URL map (routing rules)
gcloud compute url-maps create web-url-map \
  --default-service=web-backend

# 5. Create HTTPS target proxy with SSL cert
gcloud compute ssl-certificates create my-cert \
  --domains=example.com,www.example.com  # Google-managed cert

gcloud compute target-https-proxies create web-proxy \
  --url-map=web-url-map \
  --ssl-certificates=my-cert

# 6. Create forwarding rule
gcloud compute forwarding-rules create web-forwarding-rule \
  --global \
  --target-https-proxy=web-proxy \
  --address=web-lb-ip \
  --ports=443
```

### Internal TCP Load Balancer

```bash
gcloud compute forwarding-rules create internal-lb \
  --load-balancing-scheme=INTERNAL \
  --backend-service=my-backend-service \
  --ports=8080 \
  --region=us-central1 \
  --subnet=my-subnet \
  --network=my-vpc
```

---

## Cloud CDN

```bash
# Enable CDN on a backend service
gcloud compute backend-services update web-backend \
  --global \
  --enable-cdn \
  --cache-mode=CACHE_ALL_STATIC

# Configure cache TTLs
gcloud compute backend-services update web-backend \
  --global \
  --default-ttl=3600 \
  --max-ttl=86400 \
  --client-ttl=600

# Invalidate CDN cache
gcloud compute url-maps invalidate-cdn-cache web-url-map \
  --path="/*" \
  --global

# Invalidate specific path
gcloud compute url-maps invalidate-cdn-cache web-url-map \
  --path="/static/app.js" \
  --global
```

---

## Cloud DNS

```bash
# Create a managed zone (public)
gcloud dns managed-zones create example-zone \
  --dns-name=example.com. \
  --description="Example DNS zone" \
  --visibility=public

# Create a private zone (internal)
gcloud dns managed-zones create internal-zone \
  --dns-name=internal.example.com. \
  --visibility=private \
  --networks=my-vpc

# Add an A record
gcloud dns record-sets create www.example.com. \
  --zone=example-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=34.120.1.1

# Update a record (replace)
gcloud dns record-sets update www.example.com. \
  --zone=example-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=34.120.2.2

# Delete a record
gcloud dns record-sets delete www.example.com. \
  --zone=example-zone \
  --type=A

# List records in a zone
gcloud dns record-sets list --zone=example-zone
```

---

## Cloud Armor (WAF & DDoS Protection)

```bash
# Create a security policy
gcloud compute security-policies create my-armor-policy \
  --description="WAF policy for web frontend"

# Allow traffic only from specific IPs (allowlist)
gcloud compute security-policies rules create 1000 \
  --security-policy=my-armor-policy \
  --description="Allow corporate IP" \
  --src-ip-ranges=203.0.113.0/24 \
  --action=allow

# Block a specific IP
gcloud compute security-policies rules create 900 \
  --security-policy=my-armor-policy \
  --description="Block bad actor" \
  --src-ip-ranges=198.51.100.1/32 \
  --action=deny-403

# Enable pre-configured WAF rules (OWASP rules)
gcloud compute security-policies rules create 2000 \
  --security-policy=my-armor-policy \
  --expression="evaluatePreconfiguredExpr('xss-stable')" \
  --action=deny-403

gcloud compute security-policies rules create 2001 \
  --security-policy=my-armor-policy \
  --expression="evaluatePreconfiguredExpr('sqli-stable')" \
  --action=deny-403

# Rate limiting
gcloud compute security-policies rules create 3000 \
  --security-policy=my-armor-policy \
  --expression="true" \
  --action=throttle \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP

# Attach policy to backend service
gcloud compute backend-services update web-backend \
  --global \
  --security-policy=my-armor-policy
```

---

## Cloud NAT

Allows VMs without external IPs to reach the internet.

```bash
# Create a Cloud Router first (NAT requires it)
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

# Create NAT config
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges

# NAT with specific static IPs (for IP allowlisting at destinations)
gcloud compute addresses create nat-ip-1 --region=us-central1
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --nat-external-ip-pool=nat-ip-1 \
  --nat-all-subnet-ip-ranges
```

---

## Cloud VPN

```bash
# Classic VPN (single tunnel)
gcloud compute vpn-gateways create my-vpn-gw \
  --network=my-vpc \
  --region=us-central1

# HA VPN (recommended — 99.99% SLA)
gcloud compute vpn-gateways create my-ha-vpn \
  --network=my-vpc \
  --region=us-central1 \
  --stack-type=IPV4_ONLY

# Create VPN tunnel
gcloud compute vpn-tunnels create tunnel-1 \
  --peer-address=PEER_IP \
  --shared-secret=MY_SECRET \
  --ike-version=2 \
  --vpn-gateway=my-ha-vpn \
  --vpn-gateway-interface=0 \
  --router=my-router \
  --region=us-central1
```

---

## VPC Service Controls

```bash
# Create an access policy (once per org)
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --title="My Access Policy"

# Create a service perimeter
gcloud access-context-manager perimeters create my-perimeter \
  --policy=POLICY_ID \
  --title="BigQuery Perimeter" \
  --resources=projects/PROJECT_NUMBER \
  --restricted-services=bigquery.googleapis.com,storage.googleapis.com

# Create an access level (IP-based)
gcloud access-context-manager levels create corp-network \
  --policy=POLICY_ID \
  --title="Corporate Network" \
  --basic-level-spec=conditions.yaml
```

```yaml
# conditions.yaml
- ipSubnetworks:
    - 203.0.113.0/24
```

---

## Private Google Access

VMs without external IPs can reach Google APIs via Private Google Access or Private
Service Connect.

```bash
# Enable on subnet (prerequisite)
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access

# Verify: from a VM without external IP, this should work:
# curl -H "Authorization: Bearer $(gcloud auth print-access-token)" \
#   https://storage.googleapis.com/storage/v1/b/my-bucket
```

---

## Guardrails

- **Use custom-mode VPCs.** Auto-mode creates overlapping ranges that block peering.
- **Never open SSH/RDP to 0.0.0.0/0.** Use IAP tunnels (`gcloud compute ssh` uses IAP automatically).
- **Firewall rules are stateful** — you only need ingress rules; return traffic is automatic.
- **GCP has no implicit allow-all between subnets** — you must create firewall rules for subnet-to-subnet traffic.
- **Subnet ranges are permanent** — plan your CIDR blocks before provisioning.
- **Cloud Armor is only for global/regional external HTTP(S) LBs** — not for TCP LBs.
- **HA VPN requires two tunnels** (interfaces 0 and 1) for the 99.99% SLA.
