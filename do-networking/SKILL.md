---
name: do-networking
description: DigitalOcean networking — VPC, Load Balancers, DNS, CDN, and firewall strategies
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

# DigitalOcean Networking Skill

## VPC (Virtual Private Cloud)

VPCs provide isolated private networks. Droplets in the same VPC can communicate
on private IPs without going through the public internet — faster and free.

### Key facts

- Every region has a **default VPC** — new resources land there automatically
- VPCs are **regional** — you can't span regions with one VPC
- VPC CIDR ranges can't overlap (plan ahead for multi-VPC setups)
- Private DNS resolves `droplet-name.internal` within a VPC

### VPC operations

```bash
# List VPCs
doctl networking vpc list

# Create a VPC with custom IP range
doctl networking vpc create \
  --name prod-network \
  --region nyc3 \
  --ip-range 10.10.0.0/16

# Get VPC details (note the ID for use in resource creation)
doctl networking vpc get $VPC_ID

# Create droplet in a specific VPC
doctl compute droplet create web-01 \
  --image ubuntu-24-04-x64 \
  --size s-2vcpu-4gb \
  --region nyc3 \
  --vpc-uuid $VPC_ID \
  --ssh-keys $SSH_KEY_ID

# List members of a VPC
doctl networking vpc members $VPC_ID

# Update VPC name
doctl networking vpc update $VPC_ID --name new-name

# Delete VPC (must be empty)
doctl networking vpc delete $VPC_ID
```

### VPC peering (connecting multiple VPCs)

```bash
# Create a peering between two VPCs
doctl networking vpc-peering create \
  --name prod-to-staging \
  --vpc-ids $VPC_ID_1,$VPC_ID_2

# List peerings
doctl networking vpc-peering list
```

---

## Load Balancers

DO load balancers distribute traffic across Droplets or Kubernetes nodes.
They terminate SSL, check health, and support sticky sessions.

### Creating a load balancer

```bash
# Basic HTTP load balancer
doctl compute load-balancer create \
  --name web-lb \
  --region nyc3 \
  --forwarding-rules "entry_protocol:http,entry_port:80,target_protocol:http,target_port:8080" \
  --health-check "protocol:http,port:8080,path:/health,check_interval_seconds:10,response_timeout_seconds:5,healthy_threshold:3,unhealthy_threshold:3" \
  --droplet-ids $DROPLET_ID_1,$DROPLET_ID_2

# HTTPS with SSL termination (DO manages the cert)
doctl compute load-balancer create \
  --name web-lb-tls \
  --region nyc3 \
  --forwarding-rules "entry_protocol:https,entry_port:443,target_protocol:http,target_port:8080,certificate_id:$CERT_ID" \
  --redirect-http-to-https \
  --health-check "protocol:http,port:8080,path:/health,check_interval_seconds:10,response_timeout_seconds:5,healthy_threshold:2,unhealthy_threshold:5"

# Target by tag (new droplets with this tag auto-join — preferred for autoscaling)
doctl compute load-balancer create \
  --name web-lb \
  --region nyc3 \
  --forwarding-rules "entry_protocol:http,entry_port:80,target_protocol:http,target_port:8080" \
  --tag-name role:web \
  --health-check "protocol:http,port:8080,path:/healthz,check_interval_seconds:10,response_timeout_seconds:5,healthy_threshold:2,unhealthy_threshold:3"
```

### SSL certificates

```bash
# Upload a custom certificate
doctl compute certificate create \
  --name my-cert \
  --type custom \
  --certificate-chain-path ./fullchain.pem \
  --leaf-certificate-path ./cert.pem \
  --private-key-path ./privkey.pem

# Let's Encrypt managed cert (DO auto-renews — requires domain on DO DNS)
doctl compute certificate create \
  --name le-cert \
  --type lets_encrypt \
  --dns-names "example.com,www.example.com"

# List certificates
doctl compute certificate list
```

### TCP passthrough (for non-HTTP protocols)

```bash
# TCP load balancing (no SSL termination — raw TCP proxy)
doctl compute load-balancer create \
  --name tcp-lb \
  --region nyc3 \
  --forwarding-rules "entry_protocol:tcp,entry_port:5432,target_protocol:tcp,target_port:5432" \
  --health-check "protocol:tcp,port:5432,check_interval_seconds:10,response_timeout_seconds:5,healthy_threshold:2,unhealthy_threshold:3"
```

### Sticky sessions

```bash
# Enable cookie-based sticky sessions
doctl compute load-balancer update $LB_ID \
  --sticky-sessions "type:cookies,cookie_name:DO-LB,cookie_ttl_seconds:300"
```

### Load balancer management

```bash
# List load balancers
doctl compute load-balancer list

# Get details (includes IP)
doctl compute load-balancer get $LB_ID

# Add droplets
doctl compute load-balancer add-droplets $LB_ID --droplet-ids $NEW_DROPLET_ID

# Remove droplets
doctl compute load-balancer remove-droplets $LB_ID --droplet-ids $OLD_DROPLET_ID

# Delete
doctl compute load-balancer delete $LB_ID
```

---

## Domains and DNS

DO's DNS is free and fast. Point your registrar's nameservers to DO, then manage
records via doctl or the API.

### DO nameservers

```
ns1.digitalocean.com
ns2.digitalocean.com
ns3.digitalocean.com
```

### Domain management

```bash
# Add a domain to your account
doctl compute domain create example.com

# List domains
doctl compute domain list

# Get domain records
doctl compute domain records list example.com

# Create an A record
doctl compute domain records create example.com \
  --record-type A \
  --record-name "@" \
  --record-data 203.0.113.10 \
  --record-ttl 300

# Create a CNAME
doctl compute domain records create example.com \
  --record-type CNAME \
  --record-name "www" \
  --record-data "@" \
  --record-ttl 300

# Create MX record
doctl compute domain records create example.com \
  --record-type MX \
  --record-name "@" \
  --record-data "mail.example.com." \
  --record-priority 10 \
  --record-ttl 300

# Create TXT record (for SPF, DKIM, verification)
doctl compute domain records create example.com \
  --record-type TXT \
  --record-name "@" \
  --record-data "v=spf1 include:_spf.example.com ~all" \
  --record-ttl 300

# Update an existing record
doctl compute domain records update example.com \
  --record-id $RECORD_ID \
  --record-data 203.0.113.20

# Delete a record
doctl compute domain records delete example.com $RECORD_ID
```

---

## CDN

CDN endpoints cache content from a Spaces bucket at edge locations globally.

```bash
# Create CDN endpoint (origin must be a Spaces bucket domain)
doctl compute cdn create \
  --origin my-bucket.nyc3.digitaloceanspaces.com \
  --ttl 3600 \
  --custom-domain static.example.com \
  --certificate-id $CERT_ID

# List CDN endpoints
doctl compute cdn list

# Update TTL
doctl compute cdn update $CDN_ID --ttl 86400

# Purge cache (after deploying new assets)
doctl compute cdn flush $CDN_ID --files "js/*,css/*"

# Purge everything
doctl compute cdn flush $CDN_ID --files "*"

# Delete CDN
doctl compute cdn delete $CDN_ID
```

---

## Firewall Strategy: Cloud vs OS-level

### Cloud Firewalls (preferred for production)

Applied at the **network edge** — traffic is dropped before it reaches the droplet.
Zero overhead on the droplet. Managed centrally. Apply by tag.

Best for: multi-droplet setups, load-balanced tiers, anything at scale.

```bash
# Typical 3-tier firewall setup:

# Web tier (accepts HTTP/HTTPS from internet, SSH from bastion only)
doctl compute firewall create \
  --name fw-web \
  --inbound-rules "protocol:tcp,ports:80,address:0.0.0.0/0 protocol:tcp,ports:443,address:0.0.0.0/0 protocol:tcp,ports:22,address:$BASTION_IP/32" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0 protocol:udp,ports:all,address:0.0.0.0/0 protocol:icmp,address:0.0.0.0/0"
doctl compute firewall add-tags fw-web --tag-names role:web

# App tier (only accepts from LB and web tier)
doctl compute firewall create \
  --name fw-app \
  --inbound-rules "protocol:tcp,ports:8080,tag:role:web protocol:tcp,ports:22,address:$BASTION_IP/32" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0 protocol:udp,ports:all,address:0.0.0.0/0"
doctl compute firewall add-tags fw-app --tag-names role:app
```

### UFW / iptables (OS-level)

Useful for dev environments, single droplets, or rules the cloud firewall can't express.

```bash
# Basic UFW setup on a new droplet
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 'Nginx Full'
ufw enable
ufw status verbose
```

### Key difference

| | Cloud Firewall | UFW/iptables |
|---|---|---|
| **Where** | Network edge | Inside the OS |
| **Overhead** | None | Minimal |
| **Management** | Centralized | Per-droplet |
| **Best for** | Multi-droplet production | Single droplet, dev |

---

## Guardrails

- **Always set cloud firewall rules** — rely on network-edge controls, not just OS firewall
- **Use tag-based firewall assignment** — rules apply automatically to new droplets
- **Load balancer health check path** must return 2xx or the backend is marked unhealthy
- **LB target by tag, not droplet IDs** — lets you scale without updating the LB config
- **Let's Encrypt certs require DO DNS** — domain must be on DO nameservers
- **TTL 300 (5 min)** when changing DNS; bump to 3600 after propagation confirms
- **VPC private IPs are free** — always use them for backend-to-backend traffic
- **CDN cache invalidation is eventual** — flush before announcing a deployment
