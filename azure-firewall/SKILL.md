---
name: azure-firewall
description: Azure Firewall — Standard/Premium/Basic SKUs, firewall policies, DNAT/network/application rules, IDPS, TLS inspection, DNS proxy, forced tunneling, Firewall Manager, hub-spoke
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Firewall Skill

## Overview

Azure Firewall is a cloud-native, stateful, managed network security service protecting
Azure Virtual Network resources. It is highly available, scales automatically, and spans
multiple Availability Zones. Three SKUs exist: **Basic**, **Standard**, and **Premium**,
each adding capabilities on top of the previous tier.

---

## SKU Comparison

| Feature | Basic | Standard | Premium |
|---|---|---|---|
| Stateful firewall | ✓ | ✓ | ✓ |
| DNAT rules | ✓ | ✓ | ✓ |
| Network rules | ✓ | ✓ | ✓ |
| Application rules (FQDN) | Partial* | ✓ | ✓ |
| FQDN tags | ✗ | ✓ | ✓ |
| IP Groups | ✗ | ✓ | ✓ |
| Threat intelligence | Alert only | Alert/Deny | Alert/Deny |
| DNS proxy | ✗ | ✓ | ✓ |
| Forced tunneling | ✗ | ✓ | ✓ |
| Availability Zones | ✗ | ✓ | ✓ |
| IDPS | ✗ | ✗ | ✓ |
| TLS inspection | ✗ | ✗ | ✓ |
| URL filtering (full path) | ✗ | ✗ | ✓ |
| Web categories | ✗ | ✗ | ✓ |
| Max throughput | 250 Mbps | 30 Gbps | 30 Gbps |
| Firewall Policy | Required | Optional† | Required |

\* Basic supports HTTP/S FQDNs but not FQDN tags or web categories.
† Standard can use classic rules OR a Firewall Policy, but not both simultaneously.

**Choose Basic** for small/dev workloads needing simple L4+FQDN filtering at minimum cost.  
**Choose Standard** for production workloads needing full L7 filtering, DNS proxy, and zone redundancy.  
**Choose Premium** for regulated environments needing deep packet inspection, IDPS, and TLS break-inspect.

---

## Firewall Policies vs Classic Rules

| Aspect | Classic Rules | Firewall Policy |
|---|---|---|
| Management scope | Per-firewall | Shareable across firewalls |
| Hierarchy | Flat (3 rule types) | Rule Collection Groups → Collections → Rules |
| Inheritance | None | Child policy inherits parent |
| Firewall Manager support | Limited | Full |
| Required for Premium | Yes (always) | Yes |
| Recommended for new deployments | No | Yes |

**Always prefer Firewall Policies for new deployments.** Classic rules are legacy and
lack support for inheritance, Firewall Manager, and Premium features.

---

## Rule Processing Architecture

### Hierarchy (Policies)

```
Firewall Policy
└── Rule Collection Group  (priority 100–65000)
    ├── DNAT Rule Collection  (priority, action: DNAT)
    │   └── DNAT Rules
    ├── Network Rule Collection  (priority, action: Allow|Deny)
    │   └── Network Rules
    └── Application Rule Collection  (priority, action: Allow|Deny)
        └── Application Rules
```

**Processing order (within a firewall):**
1. Threat Intelligence (if set to Deny — evaluated before all rules)
2. DNAT rules (lowest priority number wins; evaluated first by type)
3. Network rules
4. Application rules

Within each type: Rule Collection Groups → Collections → Rules, evaluated by ascending
priority number. **First match wins and stops processing.**

### DNAT Rules

Translate inbound traffic from a public IP:port to a private IP:port.

Fields: Name, Protocol (TCP/UDP), Source (IP/CIDR/IP Group/`*`), Destination (firewall
public IP), Destination Port, Translated Address, Translated Port.

```bash
az network firewall policy rule-collection-group collection add-nat-collection \
  --resource-group rg-fw \
  --policy-name fw-policy \
  --rule-collection-group-name DefaultDnatRuleCollectionGroup \
  --name InboundSSH \
  --priority 100 \
  --action DNAT \
  --rule-name AllowSSH \
  --rule-type NatRule \
  --protocols TCP \
  --source-addresses '*' \
  --destination-addresses 20.1.2.3 \
  --destination-ports 22 \
  --translated-address 10.0.1.4 \
  --translated-port 22
```

### Network Rules

L3/L4 rules matching IP, port, and protocol. Support IP Groups and Service Tags as sources/destinations.

```bash
az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group rg-fw \
  --policy-name fw-policy \
  --rule-collection-group-name DefaultNetworkRuleCollectionGroup \
  --name AllowDNS \
  --priority 200 \
  --action Allow \
  --rule-name AllowDNSOut \
  --rule-type NetworkRule \
  --protocols UDP \
  --source-addresses '10.0.0.0/8' \
  --destination-addresses '168.63.129.16' \
  --destination-ports 53
```

### Application Rules

L7 rules matching FQDNs, FQDN tags, URLs (Premium), or web categories (Premium).
Terminate TLS to inspect if TLS inspection is enabled (Premium only).

Fields: Name, Source (IP/CIDR/IP Group), Protocol+Port (http:80, https:443, mssql:1433),
Target FQDNs or FQDN Tags or Web Categories.

```bash
az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group rg-fw \
  --policy-name fw-policy \
  --rule-collection-group-name DefaultApplicationRuleCollectionGroup \
  --name AllowInternet \
  --priority 300 \
  --action Allow \
  --rule-name AllowMicrosoftUpdates \
  --rule-type ApplicationRule \
  --protocols 'https=443' \
  --source-addresses '10.0.0.0/8' \
  --fqdn-tags WindowsUpdate MicrosoftActiveProtectionService
```

### FQDN Tags

Pre-built, Microsoft-managed sets of FQDNs for common services (WindowsUpdate,
AzureBackup, AzureKubernetesService, HDInsight, SQL, AppServiceEnvironment, etc.).
Microsoft updates the tag members automatically — no manual maintenance required.
Only available in Standard and Premium SKUs.

### IP Groups

Reusable objects holding one or more IP addresses/CIDRs. Reference them across
multiple rules in multiple firewalls/policies. Changes to an IP Group propagate
automatically to all rules that reference it.

```bash
az network ip-group create \
  --resource-group rg-fw \
  --name ipg-internal-subnets \
  --ip-addresses 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
```

---

## Premium Features

### IDPS (Intrusion Detection and Prevention System)

Signature-based traffic inspection for 50,000+ signatures across attack categories
(exploits, malware C2, scanning, lateral movement, etc.). Requires Premium SKU and
a Firewall Policy.

**Modes:**
- **Off** — disabled
- **Alert** — log matches, do not block
- **Alert and Deny** — log and block matching flows

**Signature overrides:** Suppress false positives or force-enable signatures per ID.

**Bypass list:** Exclude specific src/dst IP:port combinations from IDPS inspection
(e.g., trusted backup servers generating noisy signatures).

```json
// Firewall Policy IDPS settings (Bicep)
idpsSettings: {
  mode: 'Alert'
  signatureOverrides: [
    { id: '2008983', mode: 'Deny' }
    { id: '2009358', mode: 'Off'  }
  ]
  bypassTrafficSettings: [
    {
      name: 'BypassBackup'
      protocol: 'TCP'
      sourceIpGroups: [ ipGroupBackup.id ]
      destinationPorts: [ '1433' ]
    }
  ]
}
```

### TLS Inspection

Decrypts outbound TLS flows, inspects content (IDPS, URL filtering, web categories),
then re-encrypts. Requires:
1. An **intermediate CA certificate** (not a root CA) in Azure Key Vault.
2. A **managed identity** on the Firewall Policy with `Key Vault Certificate User` role.
3. TLS inspection enabled on specific application rule collections (opt-in per collection).

**Certificate chain:** Clients must trust the intermediate CA. Deploy via Group Policy
or Intune for domain-joined machines.

```bash
az network firewall policy update \
  --resource-group rg-fw \
  --name fw-policy \
  --sku Premium \
  --certificate-authority-name fw-tls-ca \
  --key-vault-secret-id https://kv-fw.vault.azure.net/certificates/fw-ca/abc123 \
  --identity "{type:UserAssigned,userAssignedIdentities:{'/subscriptions/.../fw-mi':{}}}"
```

### URL Filtering

Inspect full URL paths (not just FQDN) in application rules when TLS inspection is
active. Allows rules like `*.example.com/allowed-path/*` while blocking
`*.example.com/blocked-path/*`.

### Web Categories

Block or allow entire categories of sites (Social Networking, Gambling, Adult Content,
Streaming Media, etc.) in application rules without listing individual FQDNs.

---

## DNS Proxy

When enabled, Azure Firewall acts as a DNS forwarder for VMs in the VNet:
- VMs send DNS queries to the firewall's private IP.
- Firewall forwards to configured DNS servers (Azure DNS or custom).
- Enables FQDN-based network rules (resolves FQDNs at rule evaluation time using the
  same DNS server as the workloads — avoids split-horizon mismatches).

**Requirement:** Must be enabled to use FQDNs in **network rules** (not just application rules).

```bash
az network firewall policy update \
  --resource-group rg-fw \
  --name fw-policy \
  --dns-servers 168.63.129.16 \
  --enable-dns-proxy true
```

Configure VNet DNS to point to the firewall private IP so all VM lookups flow through it.

---

## Forced Tunneling

By default, Azure Firewall SNATs outbound traffic using its public IP(s). Forced
tunneling sends all firewall-originated management traffic (and optionally all egress)
through an on-premises path via a dedicated **AzureFirewallManagementSubnet** (`/26` minimum).

- Requires a second NIC on the firewall (management NIC) with its own public IP.
- The management subnet must have a UDR with a default route to the internet (management
  traffic must still reach Azure control plane — do not send it on-prem).
- Enable at creation time with `--enable-forced-tunneling`; cannot be changed post-deploy.

---

## Hub-Spoke Topology and UDRs

**Architecture:**
```
On-prem ──VPN/ER──► Hub VNet
                      ├── AzureFirewallSubnet (Azure Firewall)
                      ├── GatewaySubnet
                      └── Peering ──► Spoke VNet A (10.1.0.0/16)
                                 ──► Spoke VNet B (10.2.0.0/16)
```

**Route table on each spoke subnet:**

| Destination | Next Hop Type | Next Hop IP |
|---|---|---|
| 0.0.0.0/0 | VirtualAppliance | Firewall private IP |
| 10.0.0.0/8 | VirtualAppliance | Firewall private IP |

**Route table on GatewaySubnet (for on-prem → spoke):**

| Destination | Next Hop Type | Next Hop IP |
|---|---|---|
| 10.1.0.0/16 | VirtualAppliance | Firewall private IP |
| 10.2.0.0/16 | VirtualAppliance | Firewall private IP |

**Key note:** Do NOT add a UDR to AzureFirewallSubnet. Azure manages routing there.

Enable **IP forwarding** is not needed — the firewall is not a VM NIC; it is fully managed.

---

## Azure Firewall Manager

Centralized management plane for multiple firewalls and policies across subscriptions/regions.

**Concepts:**
- **Security Partner Providers** — integrate third-party SECaaS (Zscaler, Check Point,
  iboss) for branch/internet traffic filtering alongside Azure Firewall.
- **Secured Virtual Hubs** — Virtual WAN hubs with Azure Firewall deployed inside (managed
  by Microsoft; no subnet management required).
- **Hub Virtual Networks** — Classic hub-spoke with Azure Firewall in a hub VNet.
- **Policy inheritance** — Global/base policy → regional child policies → per-firewall policies.
  Child policies inherit parent rules (immutable from child perspective) and can add local rules.

```bash
# Associate a policy with a firewall
az network firewall update \
  --resource-group rg-fw \
  --name fw-hub \
  --firewall-policy fw-policy-global
```

---

## Availability Zones

Deploy across 1, 2, or 3 AZs at creation for zone redundancy. Each zone has dedicated
capacity units. Zone deployment cannot be changed post-creation.

```bash
az network firewall create \
  --resource-group rg-fw \
  --name fw-hub \
  --location eastus2 \
  --zones 1 2 3 \
  --sku AZFW_VNet \
  --tier Standard \
  --firewall-policy fw-policy \
  --vnet-name hub-vnet \
  --public-ip hub-fw-pip
```

Zone-redundant public IPs (`--zone 1 2 3` on the pip resource) are required for full redundancy.

---

## Bicep Example

```bicep
param location string = resourceGroup().location

resource fwPolicy 'Microsoft.Network/firewallPolicies@2023-09-01' = {
  name: 'fw-policy'
  location: location
  properties: {
    sku: { tier: 'Standard' }
    dnsSettings: {
      enableProxy: true
      servers: ['168.63.129.16']
    }
    threatIntelMode: 'Alert'
  }
}

resource ruleCollectionGroup 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-09-01' = {
  parent: fwPolicy
  name: 'DefaultRuleCollectionGroup'
  properties: {
    priority: 300
    ruleCollections: [
      {
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        name: 'AllowInternet'
        priority: 100
        action: { type: 'Allow' }
        rules: [
          {
            ruleType: 'ApplicationRule'
            name: 'AllowWindowsUpdate'
            sourceAddresses: ['10.0.0.0/8']
            protocols: [{ protocolType: 'Https', port: 443 }]
            fqdnTags: ['WindowsUpdate']
          }
        ]
      }
    ]
  }
}

resource firewall 'Microsoft.Network/azureFirewalls@2023-09-01' = {
  name: 'fw-hub'
  location: location
  zones: ['1', '2', '3']
  properties: {
    sku: { name: 'AZFW_VNet', tier: 'Standard' }
    firewallPolicy: { id: fwPolicy.id }
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: { id: resourceId('Microsoft.Network/virtualNetworks/subnets', 'hub-vnet', 'AzureFirewallSubnet') }
          publicIPAddress: { id: resourceId('Microsoft.Network/publicIPAddresses', 'fw-pip') }
        }
      }
    ]
  }
}
```

---

## Terraform Example

```hcl
resource "azurerm_firewall_policy" "main" {
  name                = "fw-policy"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard"

  dns { proxy_enabled = true }
  threat_intelligence_mode = "Alert"
}

resource "azurerm_firewall_policy_rule_collection_group" "main" {
  name               = "DefaultRuleCollectionGroup"
  firewall_policy_id = azurerm_firewall_policy.main.id
  priority           = 300

  application_rule_collection {
    name     = "AllowInternet"
    priority = 100
    action   = "Allow"

    rule {
      name             = "AllowWindowsUpdate"
      source_addresses = ["10.0.0.0/8"]
      protocols { type = "Https"; port = 443 }
      fqdn_tags = ["WindowsUpdate"]
    }
  }
}

resource "azurerm_firewall" "main" {
  name                = "fw-hub"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku_name            = "AZFW_VNet"
  sku_tier            = "Standard"
  firewall_policy_id  = azurerm_firewall_policy.main.id
  zones               = ["1", "2", "3"]

  ip_configuration {
    name                 = "ipconfig1"
    subnet_id            = azurerm_subnet.firewall.id
    public_ip_address_id = azurerm_public_ip.fw.id
  }
}
```

---

## Monitoring

### Diagnostic Logs (Log Analytics)

Enable via Diagnostic Settings → send to Log Analytics Workspace:

| Log Category | Contents |
|---|---|
| AzureFirewallApplicationRule | Application rule match events |
| AzureFirewallNetworkRule | Network rule match events |
| AzureFirewallDnsProxy | DNS proxy query/response log |
| AzureFirewallThreatIntelLog | Threat intel alert/deny events |
| AzureFirewallIDPS | IDPS alert/deny events (Premium) |

Enable **Structured Logs** (Resource Specific tables) rather than the legacy
AzureDiagnostics table for better query performance and schema clarity.

### Key KQL Queries

```kql
// Top denied destinations (application rules)
AZFWApplicationRule
| where Action == "Deny"
| summarize Count = count() by TargetUrl, Fqdn
| top 20 by Count

// IDPS alerts by category (Premium)
AZFWIdpsSignature
| summarize Count = count() by SignatureId, Description, Action
| order by Count desc

// DNAT rule hits
AZFWNatRule
| summarize Count = count() by DestinationIp, DestinationPort, TranslatedIp
| order by Count desc

// Threat intelligence events
AZFWThreatIntel
| where Action == "Deny"
| project TimeGenerated, SourceIp, DestinationIp, DestinationPort, ThreatDescription

// DNS proxy queries (troubleshoot FQDN rules)
AZFWDnsQuery
| where QueryName contains "windows.com"
| project TimeGenerated, QueryName, ResolvedIp
```

### Azure Monitor Workbook

Microsoft publishes an **Azure Firewall Workbook** (available in Monitor → Workbooks →
Public Templates). It provides pre-built charts for: rule hit rates, top talkers,
denied traffic, IDPS events, and SNAT port utilization.

### Metrics (Azure Monitor)

| Metric | Description |
|---|---|
| Firewall health state | 100%=healthy, 50%=degraded, 0%=critical |
| Throughput | Bits/sec processed |
| SNAT port utilization | % of available SNAT ports used (alert at >80%) |
| Application rules hit count | Matches per rule |
| Network rules hit count | Matches per rule |

---

## Performance and Scaling

### SNAT Port Exhaustion

Each firewall public IP provides ~2,496 SNAT ports per backend instance. Azure Firewall
auto-scales horizontally (adds instances). To increase SNAT capacity:

1. **Add more public IPs** — each additional pip adds ~2,496 ports per instance.
2. **Use NAT Gateway** — attach a NAT Gateway to AzureFirewallSubnet for outbound SNAT
   (separate from firewall's own pip usage). NAT Gateway provides 64,512 ports per pip,
   up to 16 pips = ~1M ports.
3. **Monitor SNAT utilization metric** — alert if >80% sustained.

```bash
# Add a second public IP to Azure Firewall
az network firewall ip-config create \
  --resource-group rg-fw \
  --firewall-name fw-hub \
  --name ipconfig2 \
  --public-ip-address fw-pip2
```

### Scale Units

Azure Firewall Standard/Premium auto-scales between 2–20 scale units (each ~1 Gbps).
You can set minimum scale units (cold standby capacity) to avoid scale-out latency
during burst events.

---

## Cost Optimization

| Strategy | Impact |
|---|---|
| Use Basic SKU for dev/test | ~60% cost reduction vs Standard |
| Consolidate firewalls per region | Fixed instance cost amortized across more flows |
| Share a Firewall Policy across multiple firewalls | Policy itself is free; firewall instances are the cost |
| Turn off dev firewalls overnight | Deallocate/allocate via automation (lose public IPs unless reserved) |
| Use IP Groups to reduce rule count | No direct cost saving, but simpler management |
| Right-size minimum scale units | Do not over-provision cold standbys |

**Pricing components:**
- Fixed deployment fee per hour (per firewall instance)
- Data processing fee per GB of traffic processed
- Firewall Policy: free for one policy per firewall; additional child policies have a small fee

---

## Comparison: Azure Firewall vs NSG vs WAF vs NVA

| Capability | Azure Firewall | NSG | App Gateway WAF | Third-party NVA |
|---|---|---|---|---|
| Stateful L4 | ✓ | ✓ (stateful) | ✓ | ✓ |
| FQDN filtering | ✓ | ✗ | ✗ | ✓ |
| L7 HTTP/S inspection | ✓ (Standard+) | ✗ | ✓ | ✓ |
| IDPS | ✓ (Premium) | ✗ | Limited | ✓ |
| TLS break/inspect | ✓ (Premium) | ✗ | ✓ (inbound) | ✓ |
| East-west filtering | ✓ | ✓ | ✗ | ✓ |
| North-south filtering | ✓ | Partial | Inbound only | ✓ |
| Managed/PaaS | ✓ | ✓ | ✓ | ✗ |
| Custom signatures | ✗ | ✗ | ✓ (WAF rules) | ✓ |
| Typical use case | Central egress/east-west | Per-subnet/NIC ACL | Inbound web app | Advanced/custom |

**Layered security recommendation:**
- **NSGs** on every subnet (default deny, allow only what's needed)
- **Azure Firewall** in hub for cross-spoke and internet egress
- **App Gateway + WAF** in front of inbound web workloads
- **Azure DDoS Protection** at VNet level for volumetric attack mitigation

---

## Common CLI Reference

```bash
# Create firewall (Standard, zone-redundant)
az network firewall create \
  --resource-group rg-fw --name fw-hub --location eastus2 \
  --sku AZFW_VNet --tier Standard --zones 1 2 3 \
  --vnet-name hub-vnet --public-ip fw-pip --firewall-policy fw-policy

# Deallocate firewall (stop billing for instances)
az network firewall deallocate --resource-group rg-fw --name fw-hub

# Allocate firewall (restart)
az network firewall allocate --resource-group rg-fw --name fw-hub \
  --vnet hub-vnet --public-ip fw-pip

# Get firewall private IP
az network firewall show --resource-group rg-fw --name fw-hub \
  --query 'ipConfigurations[0].privateIPAddress' -o tsv

# List all rule collection groups in a policy
az network firewall policy rule-collection-group list \
  --resource-group rg-fw --policy-name fw-policy -o table

# Show all rules in a rule collection group
az network firewall policy rule-collection-group show \
  --resource-group rg-fw --policy-name fw-policy \
  --name DefaultApplicationRuleCollectionGroup

# Update threat intelligence mode
az network firewall policy update \
  --resource-group rg-fw --name fw-policy \
  --threat-intel-mode Deny

# Enable DNS proxy
az network firewall policy update \
  --resource-group rg-fw --name fw-policy \
  --enable-dns-proxy true --dns-servers 168.63.129.16

# Add public IP for additional SNAT capacity
az network firewall ip-config create \
  --resource-group rg-fw --firewall-name fw-hub \
  --name ipconfig2 --public-ip-address fw-pip2
```

---

## Troubleshooting Checklist

1. **Traffic not matching application rules** → Verify DNS proxy is enabled and VMs use
   the firewall private IP as their DNS server. Without DNS proxy, FQDN network rules fail.
2. **SNAT port exhaustion** → Check `SNAT port utilization` metric. Add public IPs or
   attach NAT Gateway to AzureFirewallSubnet.
3. **Asymmetric routing drops** → Ensure GatewaySubnet UDR routes spoke prefixes through
   the firewall so return traffic follows the same path.
4. **TLS inspection breaking apps** → Check if client trusts the intermediate CA. Use
   bypass rules for apps that use cert pinning.
5. **IDPS false positives** → Review signature IDs in IDPS logs and add signature overrides
   (mode: Off) for known-good traffic.
6. **Firewall health degraded** → Check Azure Service Health. Degraded state (50%) means
   one AZ is down; traffic continues via remaining zones.
7. **Policy changes not taking effect** → Policy updates can take 1–3 minutes to propagate.
   Use `az network firewall show` to verify associated policy version.
