---
name: oci-networking
description: OCI networking — VCN, subnets, security, load balancers, gateways, DRG, VPN, DNS, and peering
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

# OCI Networking

## VCN Fundamentals

A **Virtual Cloud Network (VCN)** is OCI's private network. Each VCN belongs to one compartment and one region.

```bash
# Create VCN
oci network vcn create \
  --compartment-id $C \
  --cidr-blocks '["10.0.0.0/16"]' \
  --display-name "prod-vcn" \
  --dns-label "prodvcn"

# List VCNs
oci network vcn list --compartment-id $C --all --output table \
  --query 'data[*].{"Name":"display-name","CIDR":"cidr-blocks","State":"lifecycle-state"}'
```

### VCN CIDR Sizing Guide
- `/16` = 65,536 IPs — recommended for production
- `/24` = 256 IPs — acceptable for small/dev VCNs
- OCI reserves 3 IPs per subnet (first, last, and broadcast equivalent)
- Subnets must be within the VCN CIDR; can't overlap with each other

## Subnets: Public vs Private

```bash
# Public subnet (instances can have public IPs)
oci network subnet create \
  --compartment-id $C \
  --vcn-id $VCN_ID \
  --cidr-block "10.0.1.0/24" \
  --display-name "public-subnet" \
  --dns-label "pub" \
  --prohibit-public-ip-on-vnic false \
  --route-table-id $PUBLIC_RT_ID \
  --security-list-ids "[\"$PUBLIC_SL_ID\"]"

# Private subnet (no public IPs allowed)
oci network subnet create \
  --compartment-id $C \
  --vcn-id $VCN_ID \
  --cidr-block "10.0.2.0/24" \
  --display-name "private-subnet" \
  --dns-label "prv" \
  --prohibit-public-ip-on-vnic true \
  --route-table-id $PRIVATE_RT_ID \
  --security-list-ids "[\"$PRIVATE_SL_ID\"]"
```

## Gateways

### Internet Gateway (IGW) — public subnet egress/ingress
```bash
oci network internet-gateway create \
  --compartment-id $C --vcn-id $VCN_ID --is-enabled true --display-name "igw"
```

### NAT Gateway — private subnet outbound only
```bash
oci network nat-gateway create \
  --compartment-id $C --vcn-id $VCN_ID --display-name "nat-gw" --block-traffic false
```

### Service Gateway — private access to OCI services (Object Storage, etc.)
```bash
# Get the service OCID for your region
oci network service list  # Shows available services

oci network service-gateway create \
  --compartment-id $C \
  --vcn-id $VCN_ID \
  --display-name "svc-gw" \
  --services '[{"serviceId":"<oci-services-ocid>"}]'
```

### Route Tables
```bash
# Public route table: 0.0.0.0/0 → IGW
oci network route-table create \
  --compartment-id $C --vcn-id $VCN_ID --display-name "public-rt" \
  --route-rules '[{
    "networkEntityId": "'$IGW_ID'",
    "destination": "0.0.0.0/0",
    "destinationType": "CIDR_BLOCK"
  }]'

# Private route table: 0.0.0.0/0 → NAT, OCI services → Service GW
oci network route-table create \
  --compartment-id $C --vcn-id $VCN_ID --display-name "private-rt" \
  --route-rules '[
    {"networkEntityId": "'$NAT_GW_ID'", "destination": "0.0.0.0/0", "destinationType": "CIDR_BLOCK"},
    {"networkEntityId": "'$SVC_GW_ID'", "destination": "all-iad-services-in-oracle-services-network", "destinationType": "SERVICE_CIDR_BLOCK"}
  ]'
```

## Security Lists vs Network Security Groups (NSGs)

### Security Lists — subnet-level, stateful (prefer NSGs for new designs)
```bash
oci network security-list create \
  --compartment-id $C --vcn-id $VCN_ID --display-name "web-sl" \
  --ingress-security-rules '[
    {"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"min":443,"max":443}},"isStateless":false},
    {"protocol":"6","source":"0.0.0.0/0","tcpOptions":{"destinationPortRange":{"min":80,"max":80}},"isStateless":false},
    {"protocol":"1","source":"0.0.0.0/0","icmpOptions":{"type":3,"code":4},"isStateless":false}
  ]' \
  --egress-security-rules '[
    {"protocol":"all","destination":"0.0.0.0/0","isStateless":false}
  ]'
```

### NSGs — resource-level, attach to individual VNICs (recommended)
```bash
# Create NSG
NSG_ID=$(oci network nsg create \
  --compartment-id $C --vcn-id $VCN_ID --display-name "web-nsg" \
  --query 'data.id' --raw-output)

# Add rules
oci network nsg rules add --nsg-id $NSG_ID \
  --security-rules '[
    {
      "direction": "INGRESS",
      "protocol": "6",
      "source": "0.0.0.0/0",
      "sourceType": "CIDR_BLOCK",
      "tcpOptions": {"destinationPortRange": {"min": 443, "max": 443}},
      "isStateless": false,
      "description": "HTTPS from internet"
    },
    {
      "direction": "EGRESS",
      "protocol": "all",
      "destination": "0.0.0.0/0",
      "destinationType": "CIDR_BLOCK",
      "isStateless": false
    }
  ]'

# Reference NSG when creating instances
oci compute instance launch ... --nsg-ids "[\"$NSG_ID\"]"
```

**NSG vs Security List decision:**
- NSG: attach to individual VNICs/resources — more granular, easier to manage at scale
- Security List: attached to subnet — all resources in subnet get same rules
- NSGs allow NSG-to-NSG rules (reference another NSG as source/destination) — great for tiered app patterns

## Load Balancer

### Flexible Load Balancer (Layer 7 / HTTP/HTTPS)
```bash
# Create load balancer
LB_ID=$(oci lb load-balancer create \
  --compartment-id $C \
  --display-name "web-lb" \
  --shape-name "flexible" \
  --shape-details '{"minimumBandwidthInMbps":10,"maximumBandwidthInMbps":100}' \
  --subnet-ids "[\"$PUBLIC_SUBNET_ID\"]" \
  --is-private false \
  --query 'data.id' --raw-output)

# Wait for it to become ACTIVE
oci lb load-balancer get --load-balancer-id $LB_ID \
  --wait-for-state ACTIVE

# Create backend set
oci lb backend-set create \
  --load-balancer-id $LB_ID \
  --name "web-backends" \
  --policy "ROUND_ROBIN" \
  --health-checker '{"protocol":"HTTP","urlPath":"/health","port":8080,"returnCode":200,"intervalInMillis":10000,"timeoutInMillis":3000,"retries":3}'

# Add backends
oci lb backend create \
  --load-balancer-id $LB_ID \
  --backend-set-name "web-backends" \
  --ip-address "10.0.2.10" \
  --port 8080 \
  --weight 1

# Create HTTPS listener with SSL certificate
oci lb listener create \
  --load-balancer-id $LB_ID \
  --name "https-listener" \
  --default-backend-set-name "web-backends" \
  --port 443 \
  --protocol "HTTP" \
  --ssl-certificate-name "my-cert"

# Get LB public IP
oci lb load-balancer get --load-balancer-id $LB_ID \
  --query 'data."ip-addresses"[0]."ip-address"' --raw-output
```

### Network Load Balancer (Layer 4 / TCP/UDP — lower latency)
```bash
oci nlb network-load-balancer create \
  --compartment-id $C \
  --display-name "tcp-nlb" \
  --subnet-id $PUBLIC_SUBNET_ID \
  --is-private false \
  --is-preserve-source-destination false
```

## DRG (Dynamic Routing Gateway)

DRG is the hub for connecting VCNs, on-premises, and other cloud networks.

```bash
# Create DRG
DRG_ID=$(oci network drg create --compartment-id $C --display-name "hub-drg" \
  --query 'data.id' --raw-output)

# Attach VCN to DRG
oci network drg-attachment create \
  --drg-id $DRG_ID \
  --network-details '{"type":"VCN","id":"'$VCN_ID'"}'

# Add route to DRG in VCN route table for remote CIDR
oci network route-table update \
  --rt-id $PRIVATE_RT_ID \
  --route-rules '[{"networkEntityId":"'$DRG_ID'","destination":"192.168.0.0/16","destinationType":"CIDR_BLOCK"}]'
```

## IPSec VPN (to on-premises)

```bash
# Create Customer-Premises Equipment (CPE) — your on-prem router
CPE_ID=$(oci network cpe create \
  --compartment-id $C \
  --ip-address "203.0.113.5" \
  --display-name "onprem-router" \
  --query 'data.id' --raw-output)

# Create IPSec connection
oci network ip-sec-connection create \
  --compartment-id $C \
  --cpe-id $CPE_ID \
  --drg-id $DRG_ID \
  --static-routes '["192.168.1.0/24"]' \
  --display-name "onprem-vpn"

# Get tunnel config (IKE shared secret, OCI IP)
oci network ip-sec-tunnel list --ipsc-id <ipsec-connection-id>
```

## Local Peering (same region, different VCNs)

```bash
# In VCN A: create Local Peering Gateway
LPG_A=$(oci network local-peering-gateway create \
  --compartment-id $C --vcn-id $VCN_A_ID --display-name "lpg-a" \
  --query 'data.id' --raw-output)

# In VCN B: create Local Peering Gateway
LPG_B=$(oci network local-peering-gateway create \
  --compartment-id $C --vcn-id $VCN_B_ID --display-name "lpg-b" \
  --query 'data.id' --raw-output)

# Connect them (only one side initiates)
oci network local-peering-gateway connect \
  --local-peering-gateway-id $LPG_A \
  --peer-id $LPG_B
```

## DNS

```bash
# Create private DNS zone (resolves within VCN)
oci dns zone create \
  --compartment-id $C \
  --name "internal.mycompany.com" \
  --zone-type "PRIMARY" \
  --scope "PRIVATE" \
  --view-id <private-view-id>

# Create A record
oci dns record domain patch \
  --zone-name-or-id "internal.mycompany.com" \
  --domain "db.internal.mycompany.com" \
  --scope "PRIVATE" \
  --items '[{"domain":"db.internal.mycompany.com","rdata":"10.0.2.50","rtype":"A","ttl":300}]'
```

## Network Firewall

OCI Network Firewall is a managed NGFW (powered by Palo Alto).

```bash
# Create firewall policy
oci network-firewall firewall-policy create \
  --compartment-id $C \
  --display-name "web-policy"

# Create firewall
oci network-firewall firewall create \
  --compartment-id $C \
  --display-name "main-firewall" \
  --subnet-id $FIREWALL_SUBNET_ID \
  --network-firewall-policy-id <policy-ocid>
```

## Gotchas & Tips

- **Security List rules are additive** — all security lists attached to a subnet combine. The default security list allows all egress and stateful ingress for existing connections; removing it can break instances.
- **NSG vs SL for same resource** — if both are attached, BOTH must allow traffic. They are AND logic, not OR.
- **VCN DNS hostnames** — VCN must have `dns_label` and subnet must have `dns_label` for private hostnames to work (`hostname.subnet.vcn.oraclevcn.com`).
- **Load Balancer subnets** — public LBs need a public subnet. Regional subnets (span all ADs) are required for HA; AD-specific subnets limit availability.
- **DRG transit routing** — DRG v2 supports transit routing between attached VCNs without needing LPGs. Use DRG route tables and import/export distributions.
- **NAT Gateway is regional** — one NAT GW handles all private subnets' outbound traffic. It's not a single point of failure (OCI manages HA internally).
- **FastConnect bandwidth** — 1 Gbps, 2 Gbps, 5 Gbps, 10 Gbps options. Requires a partner or colocation. Cross-connect groups provide redundancy.
- **ICMP type 3 code 4** — always allow this in security lists for Path MTU Discovery to work correctly. Missing this breaks some application protocols silently.
