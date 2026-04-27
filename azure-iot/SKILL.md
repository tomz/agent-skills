---
name: azure-iot
description: Azure IoT — IoT Hub, IoT Central, IoT Edge, Device Provisioning Service, Digital Twins, device management, telemetry, edge computing
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure IoT Skills Reference

## Prerequisites & Setup

```bash
# Install Azure CLI + IoT extension
az extension add --name azure-iot
az extension update --name azure-iot

# Login and set subscription
az login
az account set --subscription "<subscription-id>"

# Common variables (set once, reuse throughout)
export RG="rg-iot-prod"
export HUB="my-iothub"
export LOCATION="eastus"
export DEVICE_ID="device-001"
```

---

## IoT Hub

### Provisioning

```bash
# Create resource group + IoT Hub (S1 tier)
az group create --name $RG --location $LOCATION
az iot hub create \
  --name $HUB \
  --resource-group $RG \
  --sku S1 \
  --partition-count 4 \
  --location $LOCATION

# Show connection string (service policy)
az iot hub connection-string show --hub-name $HUB --policy-name service

# Show built-in Event Hub-compatible endpoint
az iot hub show --name $HUB --query "properties.eventHubEndpoints"
```

### Device Registry

```bash
# Register a device (symmetric key auth)
az iot hub device-identity create \
  --hub-name $HUB \
  --device-id $DEVICE_ID \
  --auth-method shared_private_key

# Register with X.509 self-signed cert
az iot hub device-identity create \
  --hub-name $HUB \
  --device-id "device-x509" \
  --auth-method x509_thumbprint \
  --primary-thumbprint "<sha1-thumbprint>" \
  --secondary-thumbprint "<sha1-thumbprint>"

# Get device connection string
az iot hub device-identity connection-string show \
  --hub-name $HUB --device-id $DEVICE_ID

# List all devices
az iot hub device-identity list --hub-name $HUB --output table

# Delete device
az iot hub device-identity delete --hub-name $HUB --device-id $DEVICE_ID
```

### Device Twins

A device twin is a JSON document stored in IoT Hub for each device:

```json
{
  "deviceId": "device-001",
  "etag": "AAAAAAAAAAE=",
  "status": "enabled",
  "tags": {
    "location": {
      "plant": "Redmond43",
      "floor": "1"
    },
    "environment": "production"
  },
  "properties": {
    "desired": {
      "telemetryInterval": 30,
      "fanSpeed": 75,
      "$metadata": { ... },
      "$version": 12
    },
    "reported": {
      "telemetryInterval": 30,
      "fanSpeed": 72,
      "firmware": {
        "version": "1.2.3",
        "lastUpdate": "2026-04-01T10:00:00Z"
      },
      "$metadata": { ... },
      "$version": 9
    }
  }
}
```

```bash
# Show twin
az iot hub device-twin show --hub-name $HUB --device-id $DEVICE_ID

# Update desired properties
az iot hub device-twin update \
  --hub-name $HUB \
  --device-id $DEVICE_ID \
  --desired '{"telemetryInterval": 60, "fanSpeed": 80}'

# Update tags
az iot hub device-twin update \
  --hub-name $HUB \
  --device-id $DEVICE_ID \
  --tags '{"location": {"plant": "Redmond43", "floor": "2"}}'
```

### IoT Hub Query Language

```sql
-- All devices in a location
SELECT * FROM devices WHERE tags.location.plant = 'Redmond43'

-- Devices with firmware older than a version
SELECT deviceId, properties.reported.firmware.version
FROM devices
WHERE properties.reported.firmware.version < '2.0.0'

-- Devices where desired != reported (out of sync)
SELECT deviceId FROM devices
WHERE properties.desired.telemetryInterval != properties.reported.telemetryInterval

-- Module twins
SELECT * FROM devices.modules WHERE properties.reported.status = 'degraded'
```

```bash
# Run a query
az iot hub query \
  --hub-name $HUB \
  --query-command "SELECT * FROM devices WHERE tags.environment = 'production'"
```

### Direct Methods

```bash
# Invoke a direct method on a device (60s connect + response timeout)
az iot hub invoke-device-method \
  --hub-name $HUB \
  --device-id $DEVICE_ID \
  --method-name "reboot" \
  --method-payload '{"delay": 5}' \
  --timeout 60

# Invoke method on an IoT Edge module
az iot hub invoke-module-method \
  --hub-name $HUB \
  --device-id "edge-device-001" \
  --module-id "filterModule" \
  --method-name "SetTemperatureThreshold" \
  --method-payload '{"TemperatureThreshold": 28}'
```

### D2C / C2D Messaging

```bash
# Send a cloud-to-device (C2D) message
az iot device c2d-message send \
  --hub-name $HUB \
  --device-id $DEVICE_ID \
  --data '{"command": "updateConfig"}' \
  --expiry 3600

# Monitor device-to-cloud (D2C) telemetry (all devices)
az iot hub monitor-events --hub-name $HUB --output table

# Monitor single device with timeout
az iot hub monitor-events \
  --hub-name $HUB \
  --device-id $DEVICE_ID \
  --timeout 30

# Simulate a device sending telemetry
az iot device simulate \
  --hub-name $HUB \
  --device-id $DEVICE_ID \
  --data '{"temperature": 22.5, "humidity": 60}' \
  --msg-count 10 \
  --delay 1
```

### Message Routing

```bash
# Add a custom endpoint (Event Hub)
az iot hub routing-endpoint create \
  --hub-name $HUB \
  --endpoint-name "eh-alerts" \
  --endpoint-type eventhub \
  --endpoint-resource-group $RG \
  --endpoint-subscription-id "<sub-id>" \
  --connection-string "<eventhub-connection-string>"

# Add a route to the custom endpoint
az iot hub route create \
  --hub-name $HUB \
  --route-name "high-temp-alerts" \
  --source-type DeviceMessages \
  --endpoint-name "eh-alerts" \
  --condition 'temperature > 30 AND $connectionDeviceGroupId = "sensors"' \
  --enabled true

# Other endpoint types: servicebusqueue, servicebustopic, azurestoragecontainer, cosmosdb
# Built-in fallback route: $default (routes unmatched messages to built-in endpoint)

# List routes
az iot hub route list --hub-name $HUB --output table

# Test a route with a message body
az iot hub route test \
  --hub-name $HUB \
  --route-name "high-temp-alerts" \
  --body '{"temperature": 35}' \
  --app-properties '{"alert": "true"}'
```

### File Upload

```bash
# Link a storage account for file upload
az iot hub update \
  --name $HUB \
  --fileupload-storage-connectionstring "<storage-cs>" \
  --fileupload-storage-container-name "device-files" \
  --fileupload-sas-ttl 1 \
  --fileupload-notification-ttl 1 \
  --fileupload-max-delivery-count 10
```

---

## Device Provisioning Service (DPS)

```bash
# Create DPS instance
az iot dps create \
  --name "my-dps" \
  --resource-group $RG \
  --location $LOCATION

# Link DPS to IoT Hub
az iot dps linked-hub create \
  --dps-name "my-dps" \
  --resource-group $RG \
  --connection-string "<iothub-connection-string>" \
  --location $LOCATION

# --- Individual Enrollments ---

# Symmetric key enrollment
az iot dps enrollment create \
  --dps-name "my-dps" \
  --resource-group $RG \
  --enrollment-id "enroll-device-001" \
  --attestation-type symmetrickey \
  --device-id $DEVICE_ID \
  --iot-hubs "$HUB.azure-devices.net"

# X.509 individual enrollment (leaf cert)
az iot dps enrollment create \
  --dps-name "my-dps" \
  --resource-group $RG \
  --enrollment-id "enroll-device-x509" \
  --attestation-type x509 \
  --certificate-path ./device-cert.pem

# TPM individual enrollment
az iot dps enrollment create \
  --dps-name "my-dps" \
  --resource-group $RG \
  --enrollment-id "enroll-device-tpm" \
  --attestation-type tpm \
  --endorsement-key "<ek-public-key>"

# --- Group Enrollments ---

# X.509 CA group enrollment (all devices signed by this CA auto-enroll)
az iot dps enrollment-group create \
  --dps-name "my-dps" \
  --resource-group $RG \
  --enrollment-id "group-factory-floor" \
  --attestation-type x509 \
  --certificate-path ./ca-cert.pem

# Symmetric key group enrollment
az iot dps enrollment-group create \
  --dps-name "my-dps" \
  --resource-group $RG \
  --enrollment-id "group-symmetric" \
  --attestation-type symmetrickey

# Allocation policies: hashed (default), geoLatency, static, custom (Azure Function)
az iot dps update \
  --name "my-dps" \
  --resource-group $RG \
  --set "properties.allocationPolicy=GeoLatency"
```

---

## IoT Edge

### Runtime & Device Setup

```bash
# Register an IoT Edge device
az iot hub device-identity create \
  --hub-name $HUB \
  --device-id "edge-device-001" \
  --edge-enabled

# Get Edge device connection string
az iot hub device-identity connection-string show \
  --hub-name $HUB \
  --device-id "edge-device-001"

# On the Edge device (Ubuntu 22.04):
# curl https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb > pkg.deb
# sudo dpkg -i pkg.deb
# sudo apt-get install -y moby-engine aziot-edge
# sudo iotedge config mp --connection-string "<conn-str>"
# sudo iotedge config apply
# sudo iotedge check
```

### Deployment Manifest

```json
{
  "modulesContent": {
    "$edgeAgent": {
      "properties.desired": {
        "schemaVersion": "1.1",
        "runtime": {
          "type": "docker",
          "settings": {
            "minDockerVersion": "v1.25",
            "loggingOptions": "",
            "registryCredentials": {
              "myRegistry": {
                "username": "<acr-username>",
                "password": "<acr-password>",
                "address": "myregistry.azurecr.io"
              }
            }
          }
        },
        "systemModules": {
          "edgeAgent": {
            "type": "docker",
            "settings": {
              "image": "mcr.microsoft.com/azureiotedge-agent:1.5",
              "createOptions": {}
            }
          },
          "edgeHub": {
            "type": "docker",
            "status": "running",
            "restartPolicy": "always",
            "settings": {
              "image": "mcr.microsoft.com/azureiotedge-hub:1.5",
              "createOptions": {
                "HostConfig": { "PortBindings": {
                  "5671/tcp": [{"HostPort": "5671"}],
                  "8883/tcp": [{"HostPort": "8883"}],
                  "443/tcp":  [{"HostPort": "443"}]
                }}
              }
            }
          }
        },
        "modules": {
          "filterModule": {
            "version": "1.0",
            "type": "docker",
            "status": "running",
            "restartPolicy": "always",
            "settings": {
              "image": "myregistry.azurecr.io/filtermodule:1.2.0",
              "createOptions": {}
            }
          },
          "SimulatedTemperatureSensor": {
            "version": "1.0",
            "type": "docker",
            "status": "running",
            "restartPolicy": "always",
            "settings": {
              "image": "mcr.microsoft.com/azureiotedge-simulated-temperature-sensor:1.0",
              "createOptions": {}
            }
          }
        }
      }
    },
    "$edgeHub": {
      "properties.desired": {
        "schemaVersion": "1.1",
        "routes": {
          "sensorToFilter": "FROM /messages/modules/SimulatedTemperatureSensor/outputs/temperatureOutput INTO BrokeredEndpoint(\"/modules/filterModule/inputs/input1\")",
          "filterToCloud": "FROM /messages/modules/filterModule/outputs/output1 INTO $upstream",
          "allToCloud": "FROM /messages/* WHERE NOT IS_DEFINED($connectionModuleId) INTO $upstream"
        },
        "storeAndForwardConfiguration": {
          "timeToLiveSecs": 7200
        }
      }
    },
    "filterModule": {
      "properties.desired": {
        "TemperatureThreshold": 25,
        "SendInterval": 5
      }
    }
  }
}
```

```bash
# Apply deployment manifest to a single device
az iot edge set-modules \
  --hub-name $HUB \
  --device-id "edge-device-001" \
  --content ./deployment.json

# Create an automatic deployment (targets devices via twin query)
az iot edge deployment create \
  --hub-name $HUB \
  --deployment-id "prod-deployment-v3" \
  --content ./deployment.json \
  --target-condition "tags.environment='production'" \
  --priority 10

# Layered deployment (incremental — only adds/overrides specific modules)
az iot edge deployment create \
  --hub-name $HUB \
  --deployment-id "asa-overlay" \
  --content ./asa-layer.json \
  --target-condition "tags.environment='production'" \
  --priority 20 \
  --layered

# Monitor Edge device status
az iot hub module-twin show \
  --hub-name $HUB \
  --device-id "edge-device-001" \
  --module-id "\$edgeAgent"

# Nested Edge: set parent device
az iot hub device-identity parent set \
  --hub-name $HUB \
  --device-id "child-edge-device" \
  --parent-device-id "edge-device-001"
```

### Stream Analytics on IoT Edge

```bash
# Create ASA job and configure Edge package
az stream-analytics job create \
  --job-name "asa-edge-job" \
  --resource-group $RG \
  --location $LOCATION \
  --data-locale "en-US" \
  --compatibility-level "1.2" \
  --output-error-policy Drop \
  --sku Standard

# Add Edge input (from edgeHub)
az stream-analytics input create \
  --job-name "asa-edge-job" \
  --resource-group $RG \
  --input-name "telemetry" \
  --type Stream \
  --datasource '{"type":"Microsoft.Devices/IotHubs","properties":{"consumerGroupName":"$Default"}}' \
  --serialization '{"type":"Json","properties":{"encoding":"UTF8"}}'

# Publish ASA job as IoT Edge package, then reference in deployment manifest
# as module image: mcr.microsoft.com/azure-stream-analytics/azureiotedge:1.0
```

---

## IoT Central

```bash
# Create IoT Central application (free tier)
az iot central app create \
  --name "my-iot-central" \
  --resource-group $RG \
  --location $LOCATION \
  --sku F1 \
  --template "iotc-pnp-preview"

# List apps
az iot central app list --resource-group $RG --output table

# List devices in an IoT Central app
az iot central device list --app-id "my-iot-central" --output table

# Create a device in IoT Central
az iot central device create \
  --app-id "my-iot-central" \
  --device-id "central-device-001" \
  --device-name "Sensor Node 1" \
  --template "dtmi:mycompany:Thermostat;1" \
  --simulated false

# Get device twin / properties
az iot central device twin show \
  --app-id "my-iot-central" \
  --device-id "central-device-001"

# Send command to device
az iot central device command run \
  --app-id "my-iot-central" \
  --device-id "central-device-001" \
  --interface-id "thermostat" \
  --command-name "getMaxMinReport" \
  --content '{"since": "2026-01-01T00:00:00Z"}'

# Data export — stream telemetry to Event Hub / Blob / Service Bus
# Configured via IoT Central UI or REST API:
# POST https://{app}.azureiotcentral.com/api/dataExport/exports/{exportId}

# Jobs — bulk operations on device groups
az iot central job create \
  --app-id "my-iot-central" \
  --job-id "firmware-update-batch" \
  --group-id "all-thermostat-devices" \
  --content '[{"type":"cloudProperty","target":"firmwareVersion","path":"firmwareVersion","value":"2.0.0"}]'
```

---

## Azure Digital Twins

### DTDL Model Example

```json
{
  "@context": "dtmi:dtdl:context;3",
  "@id": "dtmi:mycompany:Building;1",
  "@type": "Interface",
  "displayName": "Building",
  "contents": [
    {
      "@type": "Property",
      "name": "occupancy",
      "schema": "integer"
    },
    {
      "@type": "Telemetry",
      "name": "energyUsage",
      "schema": "double"
    },
    {
      "@type": "Relationship",
      "name": "contains",
      "target": "dtmi:mycompany:Floor;1"
    }
  ]
}
```

```bash
# Create ADT instance
az dt create \
  --dt-name "my-digital-twins" \
  --resource-group $RG \
  --location $LOCATION

# Assign data owner role
az dt role-assignment create \
  --dt-name "my-digital-twins" \
  --assignee "<user-or-sp-object-id>" \
  --role "Azure Digital Twins Data Owner"

# Upload DTDL models
az dt model create \
  --dt-name "my-digital-twins" \
  --models ./models/

# List models
az dt model list --dt-name "my-digital-twins" --output table

# Create a twin
az dt twin create \
  --dt-name "my-digital-twins" \
  --dtmi "dtmi:mycompany:Building;1" \
  --twin-id "building-hq" \
  --properties '{"occupancy": 150}'

# Create a relationship
az dt twin relationship create \
  --dt-name "my-digital-twins" \
  --twin-id "building-hq" \
  --target-twin-id "floor-1" \
  --relationship "contains" \
  --relationship-id "hq-contains-floor1"

# Query the twin graph
az dt twin query \
  --dt-name "my-digital-twins" \
  --query-command "SELECT * FROM digitaltwins WHERE IS_OF_MODEL('dtmi:mycompany:Floor;1')"

# Create event route (send twin events to Event Hub)
az dt route create \
  --dt-name "my-digital-twins" \
  --route-name "to-eventhub" \
  --endpoint-name "eh-endpoint" \
  --filter "type = 'Microsoft.DigitalTwins.Twin.Update'"

# ADT → Azure Data Explorer integration:
# Create ADX cluster + database, then set up ADT historization via Event Hub → ADX ingestion
```

---

## Device SDKs — Common Patterns

### Python SDK

```python
import asyncio
from azure.iot.device.aio import IoTHubDeviceClient
from azure.iot.device import Message, MethodResponse

CONNECTION_STRING = "<device-connection-string>"

async def main():
    client = IoTHubDeviceClient.create_from_connection_string(CONNECTION_STRING)
    await client.connect()

    # Send D2C telemetry
    msg = Message('{"temperature": 22.5, "humidity": 60}')
    msg.content_type = "application/json"
    msg.content_encoding = "utf-8"
    await client.send_message(msg)

    # Handle desired property updates (twin patch)
    async def twin_patch_handler(patch):
        print(f"Desired property patch: {patch}")
        reported = {"telemetryInterval": patch.get("telemetryInterval")}
        await client.patch_twin_reported_properties(reported)

    client.on_twin_desired_properties_patch_received = twin_patch_handler

    # Handle direct methods
    async def method_handler(method_request):
        print(f"Method: {method_request.name}, Payload: {method_request.payload}")
        response = MethodResponse.create_from_method_request(method_request, 200, {"result": "ok"})
        await client.send_method_response(response)

    client.on_method_request_received = method_handler

    # Receive C2D messages
    async def message_handler(message):
        print(f"C2D message: {message.data}")

    client.on_message_received = message_handler

    await asyncio.sleep(60)
    await client.disconnect()

asyncio.run(main())
```

### Node.js SDK

```javascript
const { Client, Message } = require('azure-iot-device');
const { Mqtt } = require('azure-iot-device-mqtt');

const client = Client.fromConnectionString(process.env.DEVICE_CONN_STR, Mqtt);

client.open(() => {
  // Send telemetry
  const msg = new Message(JSON.stringify({ temperature: 22.5 }));
  msg.contentType = 'application/json';
  msg.contentEncoding = 'utf-8';
  client.sendEvent(msg, (err) => console.log(err || 'Sent'));

  // Twin update
  client.getTwin((err, twin) => {
    twin.on('properties.desired', (delta) => {
      twin.properties.reported.update({ telemetryInterval: delta.telemetryInterval }, () => {});
    });
  });

  // Direct method
  client.onDeviceMethod('reboot', (req, res) => {
    console.log(`Reboot requested: ${JSON.stringify(req.payload)}`);
    res.send(200, { result: 'rebooting' }, () => {});
  });
});
```

---

## Security

### X.509 Certificate Chain

```bash
# Generate root CA (dev/test only — use proper PKI for production)
openssl genrsa -out root-ca.key 4096
openssl req -x509 -new -key root-ca.key -days 3650 -out root-ca.pem \
  -subj "/CN=IoT-Root-CA/O=MyCompany"

# Generate device certificate signed by root CA
openssl genrsa -out device.key 2048
openssl req -new -key device.key -out device.csr \
  -subj "/CN=device-001/O=MyCompany"
openssl x509 -req -in device.csr -CA root-ca.pem -CAkey root-ca.key \
  -CAcreateserial -days 365 -out device.pem

# Upload root CA to IoT Hub and verify ownership
az iot hub certificate create \
  --hub-name $HUB \
  --name "root-ca" \
  --path ./root-ca.pem \
  --verified  # skip verification for dev; in prod use --etag + verify command

# Register CA with DPS
az iot dps certificate create \
  --dps-name "my-dps" \
  --resource-group $RG \
  --name "factory-root-ca" \
  --path ./root-ca.pem
```

### SAS Tokens

```bash
# Generate SAS token for a device (valid 1 hour)
az iot hub generate-sas-token \
  --hub-name $HUB \
  --device-id $DEVICE_ID \
  --duration 3600

# Generate SAS token for service-level access
az iot hub generate-sas-token \
  --hub-name $HUB \
  --policy-name service \
  --duration 86400
```

### Microsoft Defender for IoT

```bash
# Enable Defender for IoT on IoT Hub (agentless)
az iot hub update \
  --name $HUB \
  --set "properties.enableFileUploadNotifications=true"

az security iot-solution create \
  --solution-name "defender-iot" \
  --resource-group $RG \
  --iot-hubs "/subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.Devices/IotHubs/$HUB"

# The Defender micro-agent can be installed on devices for agent-based monitoring:
# apt-get install -y defender-iot-micro-agent
```

---

## Monitoring & Diagnostics

```bash
# Enable diagnostic logs (send to Log Analytics)
az monitor diagnostic-settings create \
  --name "iothub-diag" \
  --resource "/subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.Devices/IotHubs/$HUB" \
  --workspace "<log-analytics-workspace-id>" \
  --logs '[
    {"category":"Connections","enabled":true},
    {"category":"DeviceTelemetry","enabled":true},
    {"category":"C2DCommands","enabled":true},
    {"category":"DeviceIdentityOperations","enabled":true},
    {"category":"Routes","enabled":true},
    {"category":"TwinQueries","enabled":true},
    {"category":"JobsOperations","enabled":true},
    {"category":"DirectMethods","enabled":true}
  ]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Key metrics to alert on:
# d2c.telemetry.ingress.allProtocol   — total messages received
# d2c.telemetry.egress.dropped        — messages dropped by routing
# d2c.endpoints.latency.builtIn.events — routing latency
# connectedDeviceCount                 — connected devices

# Create metric alert: devices disconnecting
az monitor metrics alert create \
  --name "device-disconnect-spike" \
  --resource-group $RG \
  --scopes "/subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.Devices/IotHubs/$HUB" \
  --condition "avg connectedDeviceCount < 10" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action "<action-group-id>" \
  --severity 2

# KQL query for connection failures (Log Analytics)
# AzureDiagnostics
# | where ResourceType == "IOTHUBS"
# | where Category == "Connections"
# | where ResultType != "Ok"
# | summarize count() by ResultDescription, bin(TimeGenerated, 5m)
# | render timechart
```

---

## Bicep Provisioning

```bicep
// iothub.bicep
param location string = resourceGroup().location
param hubName string
param skuName string = 'S1'
param skuCapacity int = 1

resource iotHub 'Microsoft.Devices/IotHubs@2023-06-30' = {
  name: hubName
  location: location
  sku: {
    name: skuName
    capacity: skuCapacity
  }
  properties: {
    messagingEndpoints: {
      fileNotifications: {
        lockDurationAsIso8601: 'PT1M'
        ttlAsIso8601: 'PT1H'
        maxDeliveryCount: 10
      }
    }
    enableFileUploadNotifications: false
    cloudToDevice: {
      maxDeliveryCount: 10
      defaultTtlAsIso8601: 'PT1H'
      feedback: {
        lockDurationAsIso8601: 'PT1M'
        ttlAsIso8601: 'PT1H'
        maxDeliveryCount: 10
      }
    }
    routing: {
      fallbackRoute: {
        name: '$default'
        source: 'DeviceMessages'
        isEnabled: true
        endpointNames: ['events']
      }
    }
  }
}

output hubHostName string = iotHub.properties.hostName
output hubResourceId string = iotHub.id
```

```bash
az deployment group create \
  --resource-group $RG \
  --template-file iothub.bicep \
  --parameters hubName=$HUB skuName=S1
```

---

## Terraform Provisioning

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~>3.0" }
  }
}

provider "azurerm" { features {} }

resource "azurerm_resource_group" "iot" {
  name     = "rg-iot-prod"
  location = "East US"
}

resource "azurerm_iothub" "hub" {
  name                = "my-iothub-tf"
  resource_group_name = azurerm_resource_group.iot.name
  location            = azurerm_resource_group.iot.location

  sku { name = "S1"; capacity = 1 }

  route {
    name           = "high-temp"
    source         = "DeviceMessages"
    condition      = "temperature > 30"
    endpoint_names = ["eventhub-alerts"]
    enabled        = true
  }

  tags = { environment = "production" }
}

resource "azurerm_iothub_dps" "dps" {
  name                = "my-dps-tf"
  resource_group_name = azurerm_resource_group.iot.name
  location            = azurerm_resource_group.iot.location
  allocation_policy   = "Hashed"
  sku { name = "S1"; capacity = 1 }

  linked_hub {
    connection_string = azurerm_iothub.hub.connection_string
    location          = azurerm_resource_group.iot.location
  }
}
```

---

## Edge Patterns

### Store and Forward

IoT Edge Hub buffers messages locally when upstream connectivity is lost.
Configure `timeToLiveSecs` in `$edgeHub` desired properties (see deployment manifest above).
Messages are persisted to local storage and forwarded when connectivity resumes.

```json
// Mount persistent storage for edgeHub
"createOptions": {
  "HostConfig": {
    "Binds": ["/iotedge/storage:/iotedge/storage"]
  }
},
// Set env var on edgeHub module:
"env": {
  "storageFolder": { "value": "/iotedge/storage" }
}
```

### Local Processing (AI at the Edge)

```bash
# Deploy ONNX model via custom module
# 1. Package model + inference code as Docker image
# 2. Add module to deployment manifest with GPU createOptions if available:
"createOptions": {
  "HostConfig": {
    "DeviceRequests": [{"Count": -1, "Capabilities": [["gpu"]]}],
    "Binds": ["/models:/app/models:ro"]
  }
}

# Azure Machine Learning — publish registered model as IoT Edge module:
# az ml model deploy --name edge-classifier --model-id mymodel:1 \
#   --inference-config inference-config.json --compute-target iot-edge
```

### Nested Edge (Hierarchical IoT)

```
Internet
    │
    ▼
Top-layer Edge device (has internet access, connects to IoT Hub)
    │  (MQTT broker / API proxy module)
    ▼
Lower-layer Edge device (no direct internet, uses parent as gateway)
    │
    ▼
Leaf devices (connect to lower-layer Edge as gateway)
```

```bash
# Configure lower-layer device to use parent as gateway
# In /etc/aziot/config.toml on lower-layer device:
# [parent_hostname]
# value = "top-layer-edge-hostname"

# parent_hostname set during device creation:
az iot hub device-identity parent set \
  --hub-name $HUB \
  --device-id "lower-layer-edge" \
  --parent-device-id "top-layer-edge"
```

---

## Quick Reference: Common az iot Commands

```bash
# Hub management
az iot hub list -o table
az iot hub show -n $HUB
az iot hub stats -n $HUB                          # device count stats
az iot hub monitor-feedback -n $HUB               # C2D delivery feedback

# Device management
az iot hub device-identity list -n $HUB -o table
az iot hub device-identity export -n $HUB --blob-container-uri "<sas-uri>"
az iot hub device-identity import -n $HUB --input-blob-container-uri "<sas-uri>" \
  --output-blob-container-uri "<sas-uri>"

# Bulk jobs
az iot hub job create --hub-name $HUB --job-id "twin-update-job" \
  --job-type scheduleUpdateTwin \
  --twin-patch '{"tags":{"batchUpdated":true}}' \
  --query-condition "SELECT * FROM devices WHERE tags.environment='staging'" \
  --start-time "2026-05-01T00:00:00"

# Edge
az iot edge deployment list -n $HUB -o table
az iot edge deployment show-metric -n $HUB -d "prod-deployment-v3" --metric-id "appliedCount"

# DPS
az iot dps enrollment list --dps-name "my-dps" -g $RG -o table
az iot dps enrollment-group list --dps-name "my-dps" -g $RG -o table

# Digital Twins
az dt twin list --dt-name "my-digital-twins"
az dt twin relationship list --dt-name "my-digital-twins" --twin-id "building-hq"
az dt model delete --dt-name "my-digital-twins" --dtmi "dtmi:mycompany:OldModel;1"
```
