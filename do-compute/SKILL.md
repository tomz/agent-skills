---
name: do-compute
description: DigitalOcean Compute — Droplets, volumes, snapshots, firewalls, reserved IPs, SSH keys, and tags
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

# DigitalOcean Compute Skill

## Droplets

Droplets are DO's virtual machines. Everything is defined at creation time — you can resize
later but can't change the base image or region.

### Key concepts

| Term | Meaning |
|------|---------|
| **slug** | URL-safe identifier (`s-1vcpu-1gb`, `nyc3`, `ubuntu-24-04-x64`) |
| **size** | CPU + RAM + disk + transfer bundle |
| **image** | OS image or snapshot or backup |
| **region** | Datacenter location |
| **user data** | Cloud-init script runs on first boot |

### Creating droplets

```bash
# Minimal — smallest, cheapest (512MB RAM, 1 vCPU, $4/mo)
doctl compute droplet create my-droplet \
  --image ubuntu-24-04-x64 \
  --size s-1vcpu-512mb-10gb \
  --region nyc3

# Production-ready with SSH key, monitoring, IPv6
doctl compute droplet create web-01 \
  --image ubuntu-24-04-x64 \
  --size s-2vcpu-4gb \
  --region nyc3 \
  --ssh-keys $(doctl compute ssh-key list --format ID --no-header | tr '\n' ',') \
  --enable-monitoring \
  --enable-ipv6 \
  --tag-names "env:prod,role:web" \
  --wait

# With user data (cloud-init script)
doctl compute droplet create app-01 \
  --image ubuntu-24-04-x64 \
  --size s-2vcpu-4gb \
  --region nyc3 \
  --user-data-file ./cloud-init.yaml \
  --ssh-keys $SSH_KEY_ID

# Create multiple at once (appended with -1, -2, ...)
doctl compute droplet create web \
  --image ubuntu-24-04-x64 \
  --size s-1vcpu-1gb \
  --region nyc3 \
  --count 3 \
  --ssh-keys $SSH_KEY_ID
```

### User data (cloud-init) example

```yaml
#cloud-config
package_update: true
packages:
  - nginx
  - ufw
runcmd:
  - ufw allow 'Nginx Full'
  - ufw enable
  - systemctl enable nginx
  - systemctl start nginx
```

### Common droplet operations

```bash
# List with useful columns
doctl compute droplet list --format ID,Name,PublicIPv4,Status,Region,Size,Tags

# SSH into a droplet by name
doctl compute ssh web-01

# SSH with a specific user
doctl compute ssh web-01 --ssh-user ubuntu

# Power off / on
doctl compute droplet-action power-off $DROPLET_ID
doctl compute droplet-action power-on $DROPLET_ID

# Reboot
doctl compute droplet-action reboot $DROPLET_ID

# Resize (power off first for permanent disk resize)
doctl compute droplet-action resize $DROPLET_ID --size s-4vcpu-8gb --wait

# Rebuild from image (wipes data!)
doctl compute droplet-action rebuild $DROPLET_ID --image ubuntu-24-04-x64

# Delete
doctl compute droplet delete $DROPLET_ID --force
```

### Snapshots and backups

```bash
# Take a snapshot (droplet must be powered off for consistency)
doctl compute droplet-action snapshot $DROPLET_ID --snapshot-name "pre-upgrade-$(date +%Y%m%d)"

# List snapshots
doctl compute snapshot list --resource-type droplet

# Create droplet from snapshot
doctl compute droplet create restored \
  --image $SNAPSHOT_ID \
  --size s-2vcpu-4gb \
  --region nyc3

# Enable automatic weekly backups (20% of droplet cost)
doctl compute droplet-action enable-backups $DROPLET_ID

# List backups
doctl compute image list --type backup
```

---

## Sizes (slug patterns)

```bash
doctl compute size list  # full list with pricing

# Common slugs:
# s-1vcpu-512mb-10gb   — $4/mo  (shared CPU, dev/test)
# s-1vcpu-1gb          — $6/mo
# s-2vcpu-2gb          — $18/mo
# s-4vcpu-8gb          — $48/mo
# c-2                  — CPU-optimized
# m-2vcpu-16gb         — Memory-optimized
# g-2vcpu-8gb          — General purpose (NVMe)
```

---

## SSH Keys

```bash
# Add your public key to DO account
doctl compute ssh-key create my-laptop \
  --public-key "$(cat ~/.ssh/id_rsa.pub)"

# List keys (note the fingerprint and ID)
doctl compute ssh-key list

# Get key ID for use in droplet creation
doctl compute ssh-key list --format ID,Name,FingerPrint
```

---

## Volumes (Block Storage)

Volumes attach to droplets in the same region. They persist independently of the droplet.

```bash
# Create a 50GB volume
doctl compute volume create my-data \
  --region nyc3 \
  --size 50 \
  --filesystem-type ext4

# Attach to a droplet
doctl compute volume-action attach $VOLUME_ID $DROPLET_ID

# List volumes
doctl compute volume list

# Resize (can only grow, not shrink)
doctl compute volume-action resize $VOLUME_ID --size 100 --region nyc3

# Detach
doctl compute volume-action detach $VOLUME_ID $DROPLET_ID

# Delete (must be detached)
doctl compute volume delete $VOLUME_ID
```

After attaching, format and mount on the droplet:

```bash
# On the droplet (usually appears as /dev/sda or /dev/disk/by-id/scsi-...)
lsblk
mkfs.ext4 /dev/sda
mkdir -p /mnt/data
mount /dev/sda /mnt/data
echo '/dev/sda /mnt/data ext4 defaults,nofail,discard 0 2' >> /etc/fstab
```

---

## Reserved IPs (formerly Floating IPs)

Reserved IPs let you move a public IP between droplets — useful for failover.

```bash
# Reserve an IP in a region
doctl compute reserved-ip create --region nyc3

# Assign to a droplet
doctl compute reserved-ip-action assign $RESERVED_IP $DROPLET_ID

# Unassign
doctl compute reserved-ip-action unassign $RESERVED_IP

# List
doctl compute reserved-ip list
```

---

## Cloud Firewalls

Cloud firewalls are applied at the network edge — traffic is blocked before reaching the droplet.

```bash
# Create firewall with SSH + web rules
doctl compute firewall create \
  --name web-fw \
  --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:80,address:0.0.0.0/0 protocol:tcp,ports:443,address:0.0.0.0/0" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0 protocol:udp,ports:all,address:0.0.0.0/0 protocol:icmp,address:0.0.0.0/0"

# Add droplets to firewall
doctl compute firewall add-droplets $FIREWALL_ID --droplet-ids $DROPLET_ID

# Add by tag (better for scale — new droplets with tag get rules automatically)
doctl compute firewall add-tags $FIREWALL_ID --tag-names "role:web"

# List firewalls
doctl compute firewall list

# Update rules (replace existing)
doctl compute firewall update $FIREWALL_ID \
  --name web-fw \
  --inbound-rules "protocol:tcp,ports:22,address:10.0.0.0/8"
```

---

## Tags

Tags are key:value labels for grouping and bulk operations.

```bash
# Create tag
doctl compute tag create env:production

# Tag a droplet
doctl compute droplet tag $DROPLET_ID --tag-names "env:production,role:web"

# List droplets by tag
doctl compute droplet list --tag-name env:production

# Delete all tagged droplets (dangerous — verify first!)
doctl compute droplet list --tag-name env:staging --output json | \
  jq -r '.[].id' | \
  xargs doctl compute droplet delete --force
```

---

## Guardrails

- **Always `--wait`** when scripting sequential operations (create → attach → configure)
- **Power off before resizing disk** — live resize for CPU/RAM only (`--resize-disk=false`)
- **Snapshots on powered-off droplets** are consistent; live snapshots may have dirty state
- **Cloud firewalls beat UFW** for multi-droplet setups — manage centrally, apply by tag
- **Volumes don't move between regions** — back up data before recreating in a new region
- **Reserved IPs stay in region** — plan your region strategy before reserving
- **`--no-wait` + poll** for long operations in CI rather than blocking the pipeline
