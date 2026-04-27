---
name: sf-integrations
description: Salesforce integrations — REST API, Bulk API, Streaming API (Platform Events, CDC), Connected Apps, OAuth flows, Named Credentials, External Services, Outbound Messages, Heroku Connect, MuleSoft
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

# Salesforce Integrations Skill

## Overview

Salesforce supports multiple integration patterns. Choose based on volume, direction,
latency requirements, and whether you need real-time or batch processing:

| Pattern | Use Case | Latency | Volume |
|---|---|---|---|
| REST API | CRUD on individual records, external → SF | Low | Medium |
| Bulk API 2.0 | Mass data load/export | High | Very High |
| Platform Events | Async event-driven, SF → external | Medium | High |
| Change Data Capture | Replicate SF changes externally | Medium | High |
| Outbound Messages | SOAP notify on field change | Medium | Low |
| Heroku Connect | DB sync (Postgres ↔ SF) | Medium | High |
| Named Credentials + External Services | Declarative external API calls | Low | Medium |

---

## REST API

### Authentication — get an access token first
```bash
# Username/Password OAuth flow (not recommended for production)
curl -X POST https://login.salesforce.com/services/oauth2/token \
  -d "grant_type=password" \
  -d "client_id=YOUR_CONSUMER_KEY" \
  -d "client_secret=YOUR_CONSUMER_SECRET" \
  -d "username=user@example.com" \
  -d "password=passwordPLUSsecuritytoken"
# Returns: {"access_token":"00D...","instance_url":"https://myorg.my.salesforce.com",...}

export ACCESS_TOKEN="00D..."
export INSTANCE_URL="https://myorg.my.salesforce.com"
```

### SObject REST API
```bash
# Get record
curl "$INSTANCE_URL/services/data/v61.0/sobjects/Account/001..." \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# Create record
curl -X POST "$INSTANCE_URL/services/data/v61.0/sobjects/Account" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Name":"Acme Corp","Industry":"Technology"}'

# Update record (PATCH — partial update)
curl -X PATCH "$INSTANCE_URL/services/data/v61.0/sobjects/Account/001..." \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Rating":"Hot"}'

# Upsert by external ID
curl -X PATCH "$INSTANCE_URL/services/data/v61.0/sobjects/Account/ExternalId__c/EXT-001" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Name":"Acme Corp","Industry":"Technology"}'

# Delete record
curl -X DELETE "$INSTANCE_URL/services/data/v61.0/sobjects/Account/001..." \
  -H "Authorization: Bearer $ACCESS_TOKEN"

# SOQL via REST
curl "$INSTANCE_URL/services/data/v61.0/query?q=SELECT+Id,Name+FROM+Account+LIMIT+5" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### Composite API (multiple requests in one roundtrip)
```bash
curl -X POST "$INSTANCE_URL/services/data/v61.0/composite" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "allOrNone": true,
    "compositeRequest": [
      {
        "method": "POST",
        "url": "/services/data/v61.0/sobjects/Account",
        "referenceId": "newAccount",
        "body": {"Name": "Acme Corp"}
      },
      {
        "method": "POST",
        "url": "/services/data/v61.0/sobjects/Contact",
        "referenceId": "newContact",
        "body": {
          "LastName": "Smith",
          "AccountId": "@{newAccount.id}"
        }
      }
    ]
  }'
```

### Composite Graph API (multiple composite requests, multiple transaction boundaries)
Use for complex multi-object creates where you need reference chaining across > 25 requests.

### Calling REST API from Apex
```apex
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:MyExternalSystem/api/accounts');  // Named Credential
req.setMethod('POST');
req.setHeader('Content-Type', 'application/json');
req.setBody(JSON.serialize(new Map<String, String>{'name' => acc.Name}));
req.setTimeout(30000);  // 30 seconds max

Http http = new Http();
HttpResponse res = http.send(req);

if (res.getStatusCode() == 200 || res.getStatusCode() == 201) {
    Map<String, Object> result = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
    String externalId = (String) result.get('id');
} else {
    throw new CalloutException('External API error: ' + res.getStatus() + ' ' + res.getBody());
}
```

---

## OAuth 2.0 Flows

### Web Server Flow (3-legged, user-context)
```
1. Redirect user to:
   https://login.salesforce.com/services/oauth2/authorize
   ?response_type=code
   &client_id=CONSUMER_KEY
   &redirect_uri=https://yourapp.com/callback
   &scope=api+refresh_token

2. User logs in → Salesforce redirects to:
   https://yourapp.com/callback?code=AUTH_CODE

3. Exchange code for tokens:
   POST https://login.salesforce.com/services/oauth2/token
   grant_type=authorization_code&code=AUTH_CODE
   &client_id=CONSUMER_KEY&client_secret=CONSUMER_SECRET
   &redirect_uri=https://yourapp.com/callback
```

### JWT Bearer Flow (server-to-server, no user interaction)
```bash
# 1. Generate RSA key pair
openssl genrsa -out server.key 2048
openssl req -new -x509 -key server.key -out server.crt -days 365

# 2. Upload server.crt to Connected App → Use Digital Signatures

# 3. Build and sign JWT
# Header: {"alg":"RS256"}
# Claims: {"iss":"CONSUMER_KEY","sub":"sf_username","aud":"https://login.salesforce.com","exp":1234567890}

# 4. Exchange JWT for access token
curl -X POST https://login.salesforce.com/services/oauth2/token \
  -d "grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer" \
  -d "assertion=$JWT_TOKEN"
```

### Refresh Token Flow
```bash
curl -X POST https://login.salesforce.com/services/oauth2/token \
  -d "grant_type=refresh_token" \
  -d "client_id=CONSUMER_KEY" \
  -d "client_secret=CONSUMER_SECRET" \
  -d "refresh_token=$REFRESH_TOKEN"
```

---

## Named Credentials

Named Credentials store endpoint URL + auth for callouts. Use instead of hardcoding.

**Setup → Security → Named Credentials → New**

```apex
// Reference in Apex:
req.setEndpoint('callout:MyAPI/v2/accounts');
// Salesforce automatically injects auth headers defined in Named Credential

// Or with a per-user Named Credential (User External Credentials):
req.setEndpoint('callout:MyUserAPI/endpoint');
```

**Types:**
- **Legacy**: URL + auth in one record
- **New (Salesforce 51+)**: Separate External Credential (auth config) + Named Credential (URL)
  — supports OAuth, JWT, AWS, custom headers per principal

---

## Platform Events (Pub/Sub)

Platform Events are Salesforce's event bus. Publish from anywhere (Apex, Flow, REST),
subscribe from anywhere (Apex triggers, LWC, external via CometD/Pub-Sub API).

### Define event (Setup → Integrations → Platform Events → New)
Or deploy as metadata:
```xml
<!-- OrderShipped__e.object-meta.xml -->
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>Order Shipped</label>
    <pluralLabel>Orders Shipped</pluralLabel>
    <deploymentStatus>Deployed</deploymentStatus>
    <eventType>StandardVolume</eventType>
    <fields>
        <fullName>OrderId__c</fullName>
        <type>Text</type><length>18</length>
    </fields>
    <fields>
        <fullName>TrackingNumber__c</fullName>
        <type>Text</type><length>50</length>
    </fields>
</CustomObject>
```

### Publish from Apex
```apex
OrderShipped__e event = new OrderShipped__e(
    OrderId__c = orderId,
    TrackingNumber__c = trackingNum
);
Database.SaveResult result = EventBus.publish(event);
if (!result.isSuccess()) {
    System.debug('Failed: ' + result.getErrors()[0].getMessage());
}

// Bulk publish
List<OrderShipped__e> events = new List<OrderShipped__e>();
for (Order o : shippedOrders) {
    events.add(new OrderShipped__e(OrderId__c = o.Id, TrackingNumber__c = o.TrackingNumber__c));
}
List<Database.SaveResult> results = EventBus.publish(events);
```

### Subscribe via Apex trigger (after insert only)
```apex
trigger OrderShippedTrigger on OrderShipped__e (after insert) {
    for (OrderShipped__e event : Trigger.new) {
        // Process the event
        System.debug('Order shipped: ' + event.OrderId__c);
    }
    // Set resume checkpoint (for event replay)
    EventBus.TriggerContext.currentContext().setResumeCheckpoint('lastSuccessfulReplayId');
}
```

### Subscribe externally via CometD (Node.js example)
```javascript
const jsforce = require('jsforce');
const conn = new jsforce.Connection({ loginUrl: 'https://login.salesforce.com' });
await conn.login(username, password);

conn.streaming.topic('/event/OrderShipped__e').subscribe((message) => {
    console.log('Received:', message.payload);
});
```

---

## Change Data Capture (CDC)

CDC publishes change events (create/update/delete/undelete) for Salesforce records.

**Enable:** Setup → Integrations → Change Data Capture → Select objects

Subscribe to change events exactly like Platform Events:
- Channel: `/data/AccountChangeEvent`
- Fields: `ChangeEventHeader` (operation, changedFields, recordIds), changed field values

```apex
// Subscribe via Apex trigger
trigger AccountCDCTrigger on AccountChangeEvent (after insert) {
    for (AccountChangeEvent event : Trigger.new) {
        EventBus.ChangeEventHeader header = event.ChangeEventHeader;
        System.debug('Operation: ' + header.changeType);     // CREATE, UPDATE, DELETE
        System.debug('Records: ' + header.recordIds);
        System.debug('Changed fields: ' + header.changedFields);
    }
}
```

---

## Outbound Messages (SOAP)

Legacy but still useful for simple webhook-like notifications triggered by workflow rules.

**Setup → Process Automation → Workflow Rules → Add Workflow Action → New Outbound Message**

- Sends a SOAP message to an endpoint when a record is created/updated
- The endpoint must respond with an ACK (otherwise Salesforce retries for 24 hours)
- No Apex required — purely declarative

---

## Streaming API (Legacy PushTopic)

Older pattern using SOQL-based subscriptions. Platform Events are preferred.

```apex
// Create PushTopic
PushTopic topic = new PushTopic();
topic.Name = 'AccountUpdates';
topic.Query = 'SELECT Id, Name, Rating FROM Account';
topic.ApiVersion = 61.0;
topic.NotifyForOperationCreate = true;
topic.NotifyForOperationUpdate = true;
topic.NotifyForFields = 'Referenced';
insert topic;
// Subscribe to: /topic/AccountUpdates
```

---

## Heroku Connect

Bi-directional sync between Heroku Postgres and Salesforce. No code required.

**Setup:** Heroku → Add-ons → Heroku Connect → Map objects/fields

- Sync latency: ~30-60 seconds
- Creates `_sf_id` column in Postgres with Salesforce record ID
- Conflict resolution: configurable (SF wins or Heroku wins)
- Best for: apps built on Heroku that need SF data without REST API calls

---

## External Services (Declarative REST Callouts)

Import an OpenAPI 3.0 spec → use the external service as a Flow action or Apex.

**Setup → Integrations → External Services → Add an External Service**

```json
{
  "openapi": "3.0.0",
  "info": { "title": "Shipping API", "version": "1.0" },
  "servers": [{ "url": "https://api.shipping.com/v2" }],
  "paths": {
    "/shipment/{trackingNumber}": {
      "get": {
        "operationId": "getShipment",
        "parameters": [{ "name": "trackingNumber", "in": "path", "required": true, "schema": { "type": "string" } }],
        "responses": { "200": { "description": "OK" } }
      }
    }
  }
}
```

After import, `getShipment` appears as an invocable action in Flow Builder.

---

## Common Gotchas

- **CORS**: REST API calls from browser JS must use Salesforce CORS allowlist
  (Setup → Security → CORS). Named Credentials bypass this for server-side calls.
- **OAuth IP restrictions**: If your org has IP restrictions on a profile, JWT auth
  will fail unless the Connected App is set to "Relax IP restrictions".
- **Platform Event replay**: By default, events are replayed from `-1` (tip). Set
  `replayId` to `-2` to replay from 72 hours back (standard volume events).
- **CDC and sharing**: CDC events bypass sharing rules — subscribers see all changes.
  Filter by `recordIds` or `changedFields` in the trigger to enforce access control.
- **Composite API limit**: Max 25 subrequests per composite call. For more, use
  Composite Graph (max 500 nodes, 25 graphs) or Bulk API.
- **Callout + DML ordering**: In a single Apex transaction, you cannot make a callout
  after a DML statement. Use `@future(callout=true)` or Queueable to separate them.
- **Named Credential merge fields**: You can use `{!$Credential.Username}` in the
  request body for the authenticated user's details.
- **MuleSoft Anypoint**: Salesforce's enterprise iPaaS. Has pre-built Salesforce
  connector. Use for complex multi-system orchestrations. Separate license required.
- **Bulk API batch limits**: Bulk API 2.0 has no batch concept — it's one job with
  chunking handled automatically. Max 150MB per job, 10,000 batches/day limit per org.
