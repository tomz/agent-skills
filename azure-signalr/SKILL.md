---
name: azure-signalr
description: Azure SignalR Service & Web PubSub — real-time messaging, hubs, groups, serverless mode, upstream, WebSocket at scale, live dashboards, notifications
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure SignalR Service & Web PubSub

## Overview

Azure provides two managed real-time messaging services:

| Service | Best For |
|---------|----------|
| **Azure SignalR Service** | ASP.NET Core SignalR apps, hub/client model, chat, collaboration |
| **Azure Web PubSub** | Pure WebSocket/MQTT, event-driven pub/sub, non-.NET clients |

Both scale to millions of concurrent connections without managing WebSocket servers.

---

## Azure SignalR Service

### Service Modes

Set at resource creation or updated later (causes brief disconnect):

| Mode | Description |
|------|-------------|
| **Default** | Traditional ASP.NET Core SignalR hub server required; SignalR routes traffic |
| **Serverless** | No hub server; Azure Functions or REST API send messages; clients connect directly |
| **Classic** | Legacy — tries hub server first, falls back to serverless (not recommended) |

```bash
az signalr create \
  --name mySignalR \
  --resource-group myRG \
  --sku Standard_S1 \
  --unit-count 1 \
  --service-mode Serverless \
  --location eastus
```

### Hubs, Groups, and Users

- **Hub** — logical namespace (e.g., `chatHub`). Clients connect to a hub.
- **Group** — named subset of connections within a hub. One connection can be in multiple groups.
- **User** — identity-based targeting; one user can have multiple connections.

```csharp
// ASP.NET Core hub — Default mode
public class ChatHub : Hub
{
    // Send to all clients in hub
    public async Task SendAll(string message) =>
        await Clients.All.SendAsync("ReceiveMessage", message);

    // Send to a group
    public async Task SendGroup(string group, string message) =>
        await Clients.Group(group).SendAsync("ReceiveMessage", message);

    // Add caller to group
    public async Task JoinGroup(string group) =>
        await Groups.AddToGroupAsync(Context.ConnectionId, group);

    // Send to specific user
    public async Task SendUser(string userId, string message) =>
        await Clients.User(userId).SendAsync("ReceiveMessage", message);
}
```

```csharp
// Startup / Program.cs registration
builder.Services.AddSignalR();
builder.Services.AddAzureSignalR(connectionString);

app.MapHub<ChatHub>("/chatHub");
```

### Azure Functions Bindings (Serverless Mode)

**Negotiate function** — mandatory; returns client connection token:

```csharp
[FunctionName("negotiate")]
public static SignalRConnectionInfo Negotiate(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequest req,
    [SignalRConnectionInfo(HubName = "chat", UserId = "{headers.x-ms-client-principal-id}")] SignalRConnectionInfo connectionInfo)
    => connectionInfo;
```

**Send messages from a function:**

```csharp
[FunctionName("SendMessage")]
public static async Task SendMessage(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequest req,
    [SignalR(HubName = "chat")] IAsyncCollector<SignalRMessage> signalRMessages)
{
    var body = await new StreamReader(req.Body).ReadToEndAsync();
    await signalRMessages.AddAsync(new SignalRMessage
    {
        Target = "newMessage",
        Arguments = new[] { body }
        // GroupName = "room1"   // optional — target a group
        // UserId = "user42"     // optional — target a user
    });
}
```

**host.json** extension bundle requirement:

```json
{
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

### Scaling (Units)

Each unit supports ~1,000 concurrent connections and ~1,000,000 messages/day.
Scale up/out via the portal or CLI:

```bash
az signalr update --name mySignalR --resource-group myRG --unit-count 5
```

Premium_P1 SKU adds private endpoints, custom domains, and 100 units max.

### JavaScript Client

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/8.0.0/signalr.min.js"></script>
```

```javascript
const connection = new signalR.HubConnectionBuilder()
  .withUrl("/chatHub", {
    accessTokenFactory: () => fetchToken()  // for authenticated hubs
  })
  .withAutomaticReconnect()
  .configureLogging(signalR.LogLevel.Information)
  .build();

connection.on("ReceiveMessage", (user, message) => {
  console.log(`${user}: ${message}`);
});

await connection.start();
await connection.invoke("SendAll", "Hello world");
```

### Authentication

**Connection string (development):**

```bash
az signalr key list --name mySignalR --resource-group myRG --query primaryConnectionString -o tsv
```

**Entra ID (production) — managed identity:**

```csharp
builder.Services.AddAzureSignalR(options =>
{
    options.Endpoints = new ServiceEndpoint[]
    {
        new ServiceEndpoint(new Uri("https://mysignalr.service.signalr.net"),
                            new DefaultAzureCredential())
    };
});
```

Assign **SignalR Service Owner** role to the managed identity on the SignalR resource.

### REST API (Serverless — send without SDK)

```bash
# Send to all clients in a hub
POST https://{hostname}/api/v1/hubs/{hub}
Authorization: Bearer <token>    # HMAC-signed JWT from connection string
Content-Type: application/json

{"target": "newMessage", "arguments": ["Hello from REST"]}

# Send to a group
POST https://{hostname}/api/v1/hubs/{hub}/groups/{group}

# Send to a user
POST https://{hostname}/api/v1/hubs/{hub}/users/{userId}
```

Generate the HMAC JWT with the `Microsoft.Azure.SignalR.Management` SDK or manually
using the access key from the connection string.

---

## Azure Web PubSub

### Core Concepts

- **Hub** — logical channel; clients subscribe to a hub.
- **Group** — named subset within a hub; clients join/leave dynamically.
- **Event handler** — HTTP webhook called for `connect`, `connected`, `disconnected`, and custom message events.
- **Event listener** — fan-out to Event Hubs (no reply needed; for analytics/logging).
- **Upstream** — umbrella term for event handlers + event listeners.

```bash
az webpubsub create \
  --name myWPS \
  --resource-group myRG \
  --sku Standard_S1 \
  --unit-count 1 \
  --location eastus
```

### Client Protocols

| Protocol | Use Case |
|----------|----------|
| **Raw WebSocket** | Any language; no built-in group management |
| **`json.webpubsub.azure.v1`** | JSON subprotocol; client can join groups, send events |
| **`protobuf.webpubsub.azure.v1`** | Binary Protobuf subprotocol |
| **MQTT 3.1.1 / 5.0** | IoT devices; topics map to groups |

**Get client URL:**

```bash
az webpubsub client token generate \
  --name myWPS --resource-group myRG \
  --hub-name chat --user-id user42
```

### JSON Subprotocol (`json.webpubsub.azure.v1`)

```javascript
const ws = new WebSocket(clientUrl, "json.webpubsub.azure.v1");

ws.onopen = () => {
  // Join a group
  ws.send(JSON.stringify({ type: "joinGroup", group: "room1" }));

  // Send event to server
  ws.send(JSON.stringify({ type: "event", event: "message", dataType: "text", data: "Hello" }));

  // Send to group (if granted sendToGroup permission)
  ws.send(JSON.stringify({ type: "sendToGroup", group: "room1", dataType: "text", data: "Hi room" }));
};

ws.onmessage = ({ data }) => {
  const msg = JSON.parse(data);
  console.log(msg.data);
};
```

### Event Handlers (Upstream Webhooks)

Configure in portal or CLI — Web PubSub POSTs events to your endpoint:

```bash
az webpubsub hub create \
  --name myWPS --resource-group myRG --hub-name chat \
  --event-handler url-template="https://myapp.azurewebsites.net/api/{event}" \
    user-event-pattern="*" \
    system-events="connect" "connected" "disconnected"
```

**Webhook handler (ASP.NET Core):**

```csharp
app.UseEndpoints(endpoints =>
{
    endpoints.MapWebPubSubHub<SampleHub>("/eventhandler/{*path}");
});

public class SampleHub : WebPubSubHub
{
    public override async Task OnConnectedAsync(ConnectedEventRequest request)
    {
        // client connected
    }

    public override async ValueTask<UserEventResponse> OnMessageReceivedAsync(
        UserEventRequest request, CancellationToken ct)
    {
        // echo back
        return request.CreateResponse(request.Data, request.DataType);
    }
}
```

**Webhook validation** — Web PubSub sends an `OPTIONS` preflight with `WebHook-Request-Origin`
header; your handler must respond with `WebHook-Allowed-Origin: *` or the specific origin.

### Azure Functions Bindings

```csharp
// Negotiate
[FunctionName("negotiate")]
public static WebPubSubConnection Negotiate(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequest req,
    [WebPubSubConnection(Hub = "chat", UserId = "{query.userid}")] WebPubSubConnection conn)
    => conn;

// Send message to a group
[FunctionName("broadcast")]
public static async Task Broadcast(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequest req,
    [WebPubSub(Hub = "chat")] IAsyncCollector<WebPubSubAction> actions)
{
    var body = await new StreamReader(req.Body).ReadToEndAsync();
    await actions.AddAsync(WebPubSubAction.CreateSendToAllAction(body, WebPubSubDataType.Text));
}

// Event trigger — handle upstream events
[FunctionName("OnMessage")]
public static MessageResponse OnMessage(
    [WebPubSubTrigger("chat", WebPubSubEventType.User, "message")] UserEventRequest request)
{
    return new MessageResponse(request.Data, request.DataType);
}
```

### Server SDK — Send Messages Programmatically

```csharp
var serviceClient = new WebPubSubServiceClient(connectionString, hubName);

// Broadcast to all
await serviceClient.SendToAllAsync("Hello everyone", ContentType.TextPlain);

// Send to group
await serviceClient.SendToGroupAsync("room1", "Hello room", ContentType.TextPlain);

// Send to user
await serviceClient.SendToUserAsync("user42", "Hello user", ContentType.TextPlain);

// Add user to group
await serviceClient.AddUserToGroupAsync("room1", "user42");

// Close a connection
await serviceClient.CloseClientConnectionAsync(connectionId, "Kicked");
```

**Python SDK:**

```python
from azure.messaging.webpubsubservice import WebPubSubServiceClient

client = WebPubSubServiceClient.from_connection_string(conn_str, hub="chat")
client.send_to_all(message="Hello", content_type="text/plain")
client.send_to_group(group="room1", message={"event": "update"}, content_type="application/json")
```

---

## Cross-Cutting Concerns

### CLI Quick Reference

```bash
# SignalR
az signalr list -g myRG -o table
az signalr show -n mySignalR -g myRG
az signalr key list -n mySignalR -g myRG
az signalr key renew -n mySignalR -g myRG --key-type primary
az signalr delete -n mySignalR -g myRG

# Web PubSub
az webpubsub list -g myRG -o table
az webpubsub show -n myWPS -g myRG
az webpubsub key list -n myWPS -g myRG
az webpubsub hub list -n myWPS -g myRG
az webpubsub hub delete -n myWPS -g myRG --hub-name chat
az webpubsub client token generate -n myWPS -g myRG --hub-name chat --user-id alice
```

### Serverless Patterns

**Live dashboard:** Functions timer trigger → query DB → `SendToAll` → browser updates chart.

**Notifications:** Event Grid / Service Bus → Function trigger → `SendToUser` → targeted alert.

**IoT telemetry:** MQTT clients → Web PubSub → event listener → Event Hubs → Stream Analytics.

**Collaborative editing:** SignalR hub → broadcast diffs to group → clients apply OT/CRDT.

### Private Endpoints

```bash
# Create private endpoint for SignalR
az network private-endpoint create \
  --name myPE --resource-group myRG \
  --vnet-name myVNet --subnet mySubnet \
  --private-connection-resource-id $(az signalr show -n mySignalR -g myRG --query id -o tsv) \
  --group-id signalr \
  --connection-name myPEConn

# Disable public network access
az signalr update -n mySignalR -g myRG --set publicNetworkAccess=Disabled
```

Same pattern applies to Web PubSub (`--group-id webpubsub`).

### Custom Domains

```bash
az signalr custom-domain create \
  --name myDomain --resource-group myRG \
  --resource-name mySignalR \
  --domain-name chat.example.com \
  --certificate-resource-id /subscriptions/.../certificates/myCert
```

Custom domains require **Premium_P1** SKU and a Key Vault certificate.

### Diagnostic Logs

```bash
az monitor diagnostic-settings create \
  --name signalr-diag \
  --resource $(az signalr show -n mySignalR -g myRG --query id -o tsv) \
  --workspace /subscriptions/.../workspaces/myLAW \
  --logs '[{"category":"MessagingLogs","enabled":true},{"category":"ConnectivityLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

Key metrics: `ConnectionCount`, `MessageCount`, `InboundTraffic`, `OutboundTraffic`, `UserErrors`, `SystemErrors`.

### Bicep

```bicep
resource signalr 'Microsoft.SignalRService/signalR@2023-02-01' = {
  name: 'mySignalR'
  location: location
  sku: { name: 'Standard_S1', capacity: 1 }
  properties: {
    features: [{ flag: 'ServiceMode', value: 'Serverless' }]
    cors: { allowedOrigins: ['https://myapp.com'] }
    publicNetworkAccess: 'Enabled'
  }
}

resource webpubsub 'Microsoft.SignalRService/webPubSub@2023-02-01' = {
  name: 'myWPS'
  location: location
  sku: { name: 'Standard_S1', capacity: 1 }
  properties: {
    publicNetworkAccess: 'Enabled'
  }
}

resource wpsHub 'Microsoft.SignalRService/webPubSub/hubs@2023-02-01' = {
  parent: webpubsub
  name: 'chat'
  properties: {
    eventHandlers: [{
      urlTemplate: 'https://myapp.azurewebsites.net/api/{event}'
      userEventPattern: '*'
      systemEvents: ['connect', 'connected', 'disconnected']
    }]
  }
}
```

### Terraform

```hcl
resource "azurerm_signalr_service" "main" {
  name                = "mySignalR"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku { name = "Standard_S1"; capacity = 1 }
  service_mode        = "Serverless"
  cors { allowed_origins = ["https://myapp.com"] }
}

resource "azurerm_web_pubsub" "main" {
  name                = "myWPS"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Standard_S1"
  capacity            = 1
}

resource "azurerm_web_pubsub_hub" "chat" {
  name        = "chat"
  web_pubsub_id = azurerm_web_pubsub.main.id
  event_handler {
    url_template       = "https://myapp.azurewebsites.net/api/{event}"
    user_event_pattern = "*"
    system_events      = ["connect", "connected", "disconnected"]
  }
}
```

---

## Decision Guide

| Need | Use |
|------|-----|
| Existing ASP.NET Core SignalR app | **SignalR Service** (Default mode) |
| Serverless / Functions-only backend | **SignalR Service** (Serverless) or **Web PubSub** |
| Non-.NET clients, raw WebSocket control | **Web PubSub** |
| MQTT / IoT devices | **Web PubSub** (MQTT support) |
| Client-to-client messaging without server | **Web PubSub** (subprotocol + group permissions) |
| >1M concurrent connections | Both support it — scale units accordingly |

## References

- [SignalR Service docs](https://learn.microsoft.com/azure/azure-signalr/)
- [Web PubSub docs](https://learn.microsoft.com/azure/azure-web-pubsub/)
- [SignalR Functions bindings](https://learn.microsoft.com/azure/azure-functions/functions-bindings-signalr-service)
- [Web PubSub Functions bindings](https://learn.microsoft.com/azure/azure-functions/functions-bindings-azure-web-pubsub)
- [SignalR REST API](https://learn.microsoft.com/rest/api/signalr/)
- [Web PubSub subprotocols](https://learn.microsoft.com/azure/azure-web-pubsub/concept-service-internals)
