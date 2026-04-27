---
name: azure-networking
description: Azure networking — VNets, NSGs, ASGs, UDRs, Load Balancer, App Gateway, Front Door, Private Link, DNS, VPN, ExpressRoute, Firewall, Bastion, hub-spoke
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

# Azure Networking Skills

## Virtual Networks (VNets)

```bash
# Create a VNet with address space
az network vnet create \
  --name myVNet \
  --resource-group myRG \
  --location eastus \
  --address-prefixes 10.0.0.0/16

# Add a subnet
az network vnet subnet create \
  --name WebSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --address-prefix 10.0.1.0/24

# List subnets
az network vnet subnet list \
  --vnet-name myVNet \
  --resource-group myRG \
  --output table

# Show effective routes for a NIC
az network nic show-effective-route-table \
  --name myNIC \
  --resource-group myRG \
  --output table

# VNet peering
az network vnet peering create \
  --name hub-to-spoke \
  --resource-group myRG \
  --vnet-name hubVNet \
  --remote-vnet spokeVNet \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --allow-gateway-transit    # set on hub only

az network vnet peering create \
  --name spoke-to-hub \
  --resource-group spokeRG \
  --vnet-name spokeVNet \
  --remote-vnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/hubVNet \
  --allow-vnet-access \
  --use-remote-gateways      # set on spoke only
```

### Subnet Service Endpoints & Delegations
```bash
# Enable service endpoint (allows storage firewall to accept subnet traffic)
az network vnet subnet update \
  --name AppSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --service-endpoints Microsoft.Storage Microsoft.KeyVault Microsoft.Sql

# Delegate subnet to a service (required for Container Apps, Azure SQL MI, etc.)
az network vnet subnet update \
  --name ContainerAppsSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --delegations Microsoft.App/environments
```

## Network Security Groups (NSGs)

```bash
# Create NSG
az network nsg create --name myNSG --resource-group myRG

# Add inbound rule — allow HTTPS from internet
az network nsg rule create \
  --nsg-name myNSG \
  --resource-group myRG \
  --name AllowHTTPS \
  --priority 100 \
  --protocol Tcp \
  --direction Inbound \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 443 \
  --access Allow

# Deny all inbound (put at end with high priority number)
az network nsg rule create \
  --nsg-name myNSG \
  --resource-group myRG \
  --name DenyAllInbound \
  --priority 4096 \
  --protocol '*' \
  --direction Inbound \
  --source-address-prefixes '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges '*' \
  --access Deny

# Associate NSG to subnet
az network vnet subnet update \
  --name WebSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --network-security-group myNSG

# Associate NSG to NIC
az network nic update \
  --name myNIC \
  --resource-group myRG \
  --network-security-group myNSG

# View effective NSG rules on a NIC
az network nic list-effective-nsg --name myNIC --resource-group myRG
```

### NSG Service Tags (use instead of IP ranges)
```
Internet              → all internet traffic
VirtualNetwork        → VNet address space + peered VNets + VPN/ExpressRoute
AzureLoadBalancer     → Azure health probes (always allow this on backends)
AzureCloud            → all Azure datacenter IPs (wide)
AppGatewayBackendHealthProbe → App Gateway health probes
Storage               → Azure Storage service
Sql                   → Azure SQL
AzureMonitor          → Azure Monitor
GatewayManager        → VPN/App Gateway management plane
```

## Application Security Groups (ASGs)

ASGs let you group VMs logically and reference groups in NSG rules instead of IPs.

```bash
# Create ASGs
az network asg create --name WebServers --resource-group myRG
az network asg create --name AppServers --resource-group myRG
az network asg create --name DbServers  --resource-group myRG

# Associate a NIC's IP config with an ASG
az network nic ip-config update \
  --nic-name myWebNIC \
  --resource-group myRG \
  --name ipconfig1 \
  --application-security-groups WebServers

# NSG rule using ASGs (not IPs)
az network nsg rule create \
  --nsg-name myNSG \
  --resource-group myRG \
  --name WebToApp \
  --priority 200 \
  --protocol Tcp \
  --direction Inbound \
  --source-asgs WebServers \
  --destination-asgs AppServers \
  --destination-port-ranges 8080 \
  --access Allow
```

## User-Defined Routes (UDRs)

```bash
# Create route table
az network route-table create \
  --name myRouteTable \
  --resource-group myRG \
  --disable-bgp-route-propagation true   # prevents VPN/ExpressRoute routes propagating

# Add route — force all internet traffic through Azure Firewall
az network route-table route create \
  --route-table-name myRouteTable \
  --resource-group myRG \
  --name DefaultToFirewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.0.4   # Azure Firewall private IP

# Associate route table with subnet
az network vnet subnet update \
  --name AppSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --route-table myRouteTable
```

## Azure Load Balancer (Layer 4)

```bash
# Create public IP for LB
az network public-ip create \
  --name myLBPublicIP \
  --resource-group myRG \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3   # zone-redundant

# Create Standard LB
az network lb create \
  --name myLB \
  --resource-group myRG \
  --sku Standard \
  --frontend-ip-name FrontEnd \
  --public-ip-address myLBPublicIP \
  --backend-pool-name BackendPool

# Add health probe
az network lb probe create \
  --lb-name myLB \
  --resource-group myRG \
  --name HealthProbe \
  --protocol Tcp \
  --port 80 \
  --interval 5 \
  --threshold 2

# Add load balancing rule
az network lb rule create \
  --lb-name myLB \
  --resource-group myRG \
  --name HTTPRule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name FrontEnd \
  --backend-pool-name BackendPool \
  --probe-name HealthProbe \
  --idle-timeout 15 \
  --enable-tcp-reset true
```

## Application Gateway (Layer 7)

```bash
# Create App Gateway (WAF_v2 sku — recommended for production)
az network application-gateway create \
  --name myAppGW \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet AppGWSubnet \
  --capacity 2 \
  --sku WAF_v2 \
  --http-settings-cookie-based-affinity Disabled \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --frontend-port 443 \
  --public-ip-address myAppGWPubIP \
  --cert-file cert.pfx \
  --cert-password "password"

# App Gateway needs a /24 or larger dedicated subnet — cannot share with other resources
# Minimum subnet: /26 (for small deployments); /24 recommended for autoscale
```

## Azure Front Door (Global CDN + LB)

```bash
# Create Front Door profile (Standard or Premium)
az afd profile create \
  --profile-name myAFD \
  --resource-group myRG \
  --sku Standard_AzureFrontDoor

# Add endpoint
az afd endpoint create \
  --profile-name myAFD \
  --resource-group myRG \
  --endpoint-name myendpoint \
  --enabled-state Enabled

# Add origin group and origin
az afd origin-group create \
  --profile-name myAFD \
  --resource-group myRG \
  --origin-group-name myOriginGroup \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 30 \
  --sample-size 4 \
  --successful-samples-required 3

az afd origin create \
  --profile-name myAFD \
  --resource-group myRG \
  --origin-group-name myOriginGroup \
  --origin-name myOrigin \
  --host-name myapp.azurewebsites.net \
  --https-port 443 \
  --priority 1 \
  --weight 1000
```

## Private Link & Private Endpoints

```bash
# Create private endpoint for a Storage Account
az network private-endpoint create \
  --name myPE \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet PrivateEndpointSubnet \
  --private-connection-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/myaccount \
  --group-id blob \
  --connection-name myPEConnection

# Create Private DNS Zone for blob storage
az network private-dns zone create \
  --resource-group myRG \
  --name "privatelink.blob.core.windows.net"

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.blob.core.windows.net" \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false

# Create DNS A record for private endpoint
az network private-endpoint dns-zone-group create \
  --endpoint-name myPE \
  --resource-group myRG \
  --name myDNSZoneGroup \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name blob
```

### Private DNS Zone Names (Common)
```
Azure Storage Blob:    privatelink.blob.core.windows.net
Azure Storage File:    privatelink.file.core.windows.net
Azure SQL:             privatelink.database.windows.net
Azure Key Vault:       privatelink.vaultcore.azure.net
Azure Container Reg:   privatelink.azurecr.io
Azure Monitor:         privatelink.monitor.azure.com
Azure Cosmos DB:       privatelink.documents.azure.com
```

## Azure DNS

```bash
# Create public DNS zone
az network dns zone create \
  --resource-group myRG \
  --name contoso.com

# Add A record
az network dns record-set a add-record \
  --resource-group myRG \
  --zone-name contoso.com \
  --record-set-name www \
  --ipv4-address 1.2.3.4

# Add CNAME
az network dns record-set cname set-record \
  --resource-group myRG \
  --zone-name contoso.com \
  --record-set-name app \
  --cname myapp.azurewebsites.net

# List all records
az network dns record-set list \
  --resource-group myRG \
  --zone-name contoso.com \
  --output table

# Get nameservers (delegate from registrar)
az network dns zone show \
  --resource-group myRG \
  --name contoso.com \
  --query nameServers \
  --output tsv
```

## VPN Gateway

```bash
# Create gateway subnet (must be named GatewaySubnet)
az network vnet subnet create \
  --name GatewaySubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --address-prefix 10.0.255.0/27   # /27 minimum

# Create VPN Gateway (takes 20-45 minutes)
az network vnet-gateway create \
  --name myVPNGW \
  --resource-group myRG \
  --vnet myVNet \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2 \
  --public-ip-address myVPNGWPubIP \
  --no-wait

# Create local network gateway (represents on-prem)
az network local-gateway create \
  --name onPremGW \
  --resource-group myRG \
  --gateway-ip-address <on-prem-public-ip> \
  --local-address-prefixes 192.168.0.0/16

# Create the VPN connection
az network vpn-connection create \
  --name myVPNConnection \
  --resource-group myRG \
  --vnet-gateway1 myVPNGW \
  --local-gateway2 onPremGW \
  --shared-key "SuperSecretKey123!"
```

## Azure Firewall

```bash
# Create Firewall subnet (must be named AzureFirewallSubnet, /26 minimum)
az network vnet subnet create \
  --name AzureFirewallSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --address-prefix 10.0.254.0/26

# Create Firewall Policy
az network firewall policy create \
  --name myFWPolicy \
  --resource-group myRG \
  --sku Standard

# Create Azure Firewall
az network firewall create \
  --name myFirewall \
  --resource-group myRG \
  --vnet-name myVNet \
  --firewall-policy myFWPolicy \
  --public-ip myFirewallPubIP

# Add application rule (FQDN-based)
az network firewall policy rule-collection-group create \
  --policy-name myFWPolicy \
  --resource-group myRG \
  --name AppRuleGroup \
  --priority 200

az network firewall policy rule-collection-group collection add-filter-collection \
  --policy-name myFWPolicy \
  --resource-group myRG \
  --rule-collection-group-name AppRuleGroup \
  --name AllowWindowsUpdate \
  --collection-priority 100 \
  --action Allow \
  --rule-name WindowsUpdate \
  --rule-type ApplicationRule \
  --protocols Http=80 Https=443 \
  --fqdn-tags WindowsUpdate \
  --source-addresses '*'
```

## Azure Bastion

```bash
# Create AzureBastionSubnet (must be /26 minimum)
az network vnet subnet create \
  --name AzureBastionSubnet \
  --vnet-name myVNet \
  --resource-group myRG \
  --address-prefix 10.0.253.0/26

# Deploy Bastion (Standard SKU for native client support)
az network bastion create \
  --name myBastion \
  --resource-group myRG \
  --vnet-name myVNet \
  --public-ip-address myBastionPubIP \
  --sku Standard \
  --enable-tunneling true   # enables native SSH/RDP without browser

# SSH to a VM via Bastion (requires Standard SKU + tunneling)
az network bastion ssh \
  --name myBastion \
  --resource-group myRG \
  --target-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --auth-type ssh-key \
  --username azureuser \
  --ssh-key ~/.ssh/id_rsa
```

## Hub-Spoke Topology Pattern

```
Hub VNet (10.0.0.0/16)
├── GatewaySubnet (10.0.255.0/27) → VPN/ExpressRoute Gateway
├── AzureFirewallSubnet (10.0.254.0/26) → Azure Firewall
├── AzureBastionSubnet (10.0.253.0/26) → Azure Bastion
└── ManagementSubnet (10.0.1.0/24)

Spoke VNet A — Prod (10.1.0.0/16)   ← peered to hub (use-remote-gateways)
├── WebSubnet (10.1.1.0/24)
├── AppSubnet (10.1.2.0/24)
└── DbSubnet  (10.1.3.0/24)

Spoke VNet B — Dev (10.2.0.0/16)    ← peered to hub (use-remote-gateways)
```

**Key rules:**
- Hub has `allow-gateway-transit = true` on peerings to spokes
- Spokes have `use-remote-gateways = true` on peering to hub
- All traffic between spokes routes through Azure Firewall (via UDRs)
- BGP route propagation disabled on spoke route tables (UDRs take precedence)

## Gotchas

- **NSG priority**: Lower number = higher priority (100 evaluated before 200).
- **AzureLoadBalancer service tag**: Always allow this in NSG rules on backend VMs — it carries health probes.
- **App Gateway subnet**: Must be dedicated (no other resources), needs /24 for autoscale.
- **Private endpoint + DNS**: Must configure Private DNS Zone + VNet link; otherwise name resolves to public IP.
- **VNet peering is not transitive**: Spoke A cannot reach Spoke B through Hub without routing via NVA/Firewall.
- **ExpressRoute vs VPN**: ExpressRoute doesn't traverse public internet; VPN does. ER latency is predictable; VPN isn't.
- **Azure Firewall**: Outbound SNAT by default. If you have >512 concurrent connections from same source, use multiple public IPs or SNAT port exhaustion occurs.
- **`--disable-bgp-route-propagation`**: Set to `true` on spoke route tables to prevent VPN/ER routes overriding UDRs.
- **Bastion Standard** is required for native client (SSH/RDP without browser). Basic SKU = browser only.
