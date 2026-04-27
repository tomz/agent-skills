---
name: azure-messaging
description: Azure messaging — Event Hubs, Service Bus, Event Grid, Queue Storage, stream processing, pub/sub, dead-letter queues, Schema Registry
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Messaging Services

Comprehensive reference for Azure's messaging portfolio: Event Hubs, Service Bus,
Event Grid, and Queue Storage — covering architecture, CLI, SDK patterns, Bicep/Terraform,
security, monitoring, and decision guidance.

---

## Table of Contents

1. [Service Decision Tree](#decision-tree)
2. [Event Hubs](#event-hubs)
3. [Service Bus](#service-bus)
4. [Event Grid](#event-grid)
5. [Queue Storage](#queue-storage)
6. [Security](#security)
7. [Monitoring](#monitoring)
8. [Performance Tuning](#performance-tuning)
9. [Infrastructure as Code](#infrastructure-as-code)

---

## Decision Tree

```
Need messaging?
│
├─ High-throughput event streaming (telemetry, logs, clickstreams)?
│   └─► Event Hubs  (millions of events/sec, replay, Kafka-compatible)
│
├─ Enterprise messaging with ordering, sessions, transactions?
│   └─► Service Bus  (queues + topics/subscriptions, DLQ, AMQP)
│
├─ React to Azure resource events or HTTP webhooks, fan-out notifications?
│   └─► Event Grid  (push-based, serverless, near-real-time routing)
│
└─ Simple decoupling, no ordering guarantee, low cost?
    └─► Queue Storage  (simple FIFO, cheap, 64 KB messages up to 500 TB)
```

### Quick Comparison

| Feature               | Event Hubs      | Service Bus        | Event Grid       | Queue Storage  |
|-----------------------|-----------------|--------------------|------------------|----------------|
| Pattern               | Stream          | Message broker     | Event router     | Simple queue   |
| Max message size      | 1 MB            | 256 KB / 100 MB*   | 1 MB             | 64 KB          |
| Retention             | 1–90 days       | Until consumed      | N/A (push)       | 7 days         |
| Ordering              | Per partition   | Full (sessions)    | Best-effort      | Approximate    |
| Replay                | ✓               | ✗                  | ✗                | ✗              |
| Dead-letter           | ✗               | ✓                  | ✓                | ✗              |
| Transactions          | ✗               | ✓                  | ✗                | ✗              |
| Kafka protocol        | ✓               | ✗                  | ✗                | ✗              |
| Throughput            | Very high       | High               | Very high        | High           |
| Pricing basis         | TUs/PUs + data  | Operations + data  | Operations       | Transactions   |

*Premium Service Bus supports 100 MB messages.

---

## Event Hubs

### Concepts

- **Namespace** — container for event hubs; billing and network boundary.
- **Event Hub** — a Kafka topic equivalent; one logical stream.
- **Partition** — ordered, immutable sequence within an event hub (1–32 standard, up to 2000 premium).
  Events with the same partition key always land on the same partition.
- **Consumer Group** — independent view of the stream (like a Kafka consumer group).
  Each consumer group maintains its own offset (checkpoint).
- **Throughput Units (TUs)** — Standard/Basic tier: 1 TU = 1 MB/s ingress, 2 MB/s egress.
- **Processing Units (PUs)** — Premium tier: more powerful, dedicated resources, 10 GB/PU.
- **Capture** — automatically saves event data to Azure Blob Storage or ADLS Gen2 in
  Avro format on a configurable time/size window.
- **Schema Registry** — versioned Avro/JSON schema enforcement for producers/consumers.

### Tiers

| Tier     | TUs/PUs       | Max partitions | Retention   | Kafka | Schema Registry |
|----------|---------------|----------------|-------------|-------|-----------------|
| Basic    | 1–20 TUs      | 32             | 1 day       | ✗     | ✗               |
| Standard | 1–20 TUs      | 32             | 1–7 days    | ✓     | ✓ (add-on)      |
| Premium  | 1–16 PUs      | 100–2000       | 1–90 days   | ✓     | ✓ (included)    |
| Dedicated| Capacity units| 2000           | 1–90 days   | ✓     | ✓               |

### az CLI — Event Hubs

```bash
# Create namespace (Standard, 2 TUs, zone-redundant)
az eventhubs namespace create \
  --resource-group rg-messaging \
  --name evhns-myapp-prod \
  --location eastus \
  --sku Standard \
  --capacity 2 \
  --zone-redundant true \
  --enable-kafka true \
  --enable-auto-inflate true \
  --maximum-throughput-units 10

# Create an event hub with 8 partitions, 3-day retention
az eventhubs eventhub create \
  --resource-group rg-messaging \
  --namespace-name evhns-myapp-prod \
  --name evh-telemetry \
  --partition-count 8 \
  --message-retention 3 \
  --cleanup-policy Delete

# Create a consumer group
az eventhubs eventhub consumer-group create \
  --resource-group rg-messaging \
  --namespace-name evhns-myapp-prod \
  --eventhub-name evh-telemetry \
  --name cg-analytics

# Enable Capture to Blob Storage
az eventhubs eventhub update \
  --resource-group rg-messaging \
  --namespace-name evhns-myapp-prod \
  --name evh-telemetry \
  --enable-capture true \
  --capture-interval 300 \
  --capture-size-limit 314572800 \
  --destination-name EventHubArchive.AzureBlockBlob \
  --storage-account /subscriptions/<sub>/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/stmyapp \
  --blob-container capture-archive \
  --archive-name-format "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"

# Schema Registry — create group and schema
az eventhubs namespace schema-registry create \
  --resource-group rg-messaging \
  --namespace-name evhns-myapp-prod \
  --name sg-telemetry \
  --schema-compatibility Backward \
  --schema-type Avro

# List connection strings
az eventhubs namespace authorization-rule keys list \
  --resource-group rg-messaging \
  --namespace-name evhns-myapp-prod \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv

# Scale TUs
az eventhubs namespace update \
  --resource-group rg-messaging \
  --name evhns-myapp-prod \
  --capacity 4
```

### Python SDK — Event Hubs

```python
# pip install azure-eventhub azure-identity

from azure.eventhub import EventHubProducerClient, EventData, EventDataBatch
from azure.eventhub.aio import EventHubConsumerClient
from azure.eventhub.extensions.checkpointstoreblobaio import BlobCheckpointStore
from azure.identity import DefaultAzureCredential

NAMESPACE = "evhns-myapp-prod.servicebus.windows.net"
EVENTHUB  = "evh-telemetry"

# ── Producer ─────────────────────────────────────────────────────────────────
credential = DefaultAzureCredential()

producer = EventHubProducerClient(
    fully_qualified_namespace=NAMESPACE,
    eventhub_name=EVENTHUB,
    credential=credential,
)

with producer:
    # Batching (recommended — respects max batch size)
    batch: EventDataBatch = producer.create_batch()
    for i in range(1000):
        try:
            event = EventData(f'{{"sensor": "T{i}", "value": {i * 0.1}}}')
            event.properties = {"source": "iot-gateway"}
            batch.add(event)
        except ValueError:
            # Batch full — send and start a new one
            producer.send_batch(batch)
            batch = producer.create_batch()
            batch.add(event)
    producer.send_batch(batch)

# Send with partition key (ensures same partition)
with producer:
    batch = producer.create_batch(partition_key="device-001")
    batch.add(EventData('{"temp": 23.5}'))
    producer.send_batch(batch)

# ── Consumer with EventProcessorClient (production pattern) ──────────────────
import asyncio

async def process_event(partition_context, event):
    data = event.body_as_str()
    print(f"Partition {partition_context.partition_id}: {data}")
    # Checkpoint after processing (persists offset to blob storage)
    await partition_context.update_checkpoint(event)

async def process_error(partition_context, error):
    print(f"Error on partition {partition_context.partition_id}: {error}")

async def main():
    checkpoint_store = BlobCheckpointStore.from_connection_string(
        conn_str="DefaultEndpointsProtocol=https;AccountName=stmyapp;...",
        container_name="checkpoints",
    )
    consumer = EventHubConsumerClient(
        fully_qualified_namespace=NAMESPACE,
        eventhub_name=EVENTHUB,
        consumer_group="cg-analytics",
        checkpoint_store=checkpoint_store,
        credential=DefaultAzureCredential(),
    )
    async with consumer:
        await consumer.receive(
            on_event=process_event,
            on_error=process_error,
            starting_position="-1",  # "-1" = beginning, "@latest" = new only
        )

asyncio.run(main())
```

### Kafka Protocol

Event Hubs exposes a Kafka-compatible endpoint on port 9093 (SASL_SSL).
Lift-and-shift existing Kafka producers/consumers with minimal config changes:

```properties
# producer.properties
bootstrap.servers=evhns-myapp-prod.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="$ConnectionString" \
  password="Endpoint=sb://evhns-myapp-prod.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<key>";
```

---

## Service Bus

### Concepts

- **Namespace** — billing and network boundary; contains queues and topics.
- **Queue** — point-to-point FIFO messaging; competing consumers.
- **Topic** — pub/sub; one message fan-out to multiple **subscriptions**.
- **Subscription** — filtered view of topic messages; each has its own DLQ.
- **Session** — groups related messages with a session ID; guaranteed FIFO per session.
- **Dead-Letter Queue (DLQ)** — holds messages that failed processing or expired.
- **Deferral** — postpone processing a message; retrieve later by sequence number.
- **Duplicate Detection** — deduplicate within a configurable time window using MessageId.
- **Auto-Forward** — automatically forward messages from queue/subscription to another entity.
- **Scheduled Messages** — enqueue with a future `ScheduledEnqueueTimeUtc`.
- **Transactions** — atomic send+complete across multiple entities in the same namespace.

### Tiers

| Feature                | Standard          | Premium                 |
|------------------------|-------------------|-------------------------|
| Message size           | 256 KB            | 100 MB                  |
| Max queue/topic size   | 80 GB             | 1–100 GB (configurable) |
| Dedicated resources    | ✗ (shared)        | ✓ (isolated)            |
| VNet integration       | ✗                 | ✓                       |
| Geo-DR                 | ✗                 | ✓                       |
| Pricing                | Per operation     | Per messaging unit      |

### az CLI — Service Bus

```bash
# Create namespace (Premium, 1 messaging unit)
az servicebus namespace create \
  --resource-group rg-messaging \
  --name sbns-myapp-prod \
  --location eastus \
  --sku Premium \
  --premium-messaging-units 1 \
  --zone-redundant true

# Create queue with DLQ, sessions, duplicate detection
az servicebus queue create \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --name q-orders \
  --enable-dead-lettering-on-message-expiration true \
  --enable-session true \
  --enable-duplicate-detection true \
  --duplicate-detection-history-time-window PT10M \
  --default-message-time-to-live P14D \
  --lock-duration PT2M \
  --max-delivery-count 5 \
  --max-size 5120

# Create topic
az servicebus topic create \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --name t-notifications \
  --enable-duplicate-detection true \
  --default-message-time-to-live P7D \
  --max-size 5120

# Create subscription with SQL filter
az servicebus topic subscription create \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --topic-name t-notifications \
  --name sub-email \
  --enable-dead-lettering-on-message-expiration true \
  --max-delivery-count 10 \
  --lock-duration PT1M

az servicebus topic subscription rule create \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --topic-name t-notifications \
  --subscription-name sub-email \
  --name filter-email \
  --filter-sql-expression "channel = 'email'"

# Enable auto-forwarding from queue to topic
az servicebus queue update \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --name q-staging \
  --forward-to t-notifications

# Schedule a message (via CLI send — usually done via SDK)
# Peek at DLQ
az servicebus queue message peek \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --queue-name q-orders/deadLetterQueue
```

### Python SDK — Service Bus

```python
# pip install azure-servicebus azure-identity

from azure.servicebus import ServiceBusClient, ServiceBusMessage
from azure.servicebus.management import ServiceBusAdministrationClient
from azure.identity import DefaultAzureCredential
from datetime import datetime, timedelta, timezone

NAMESPACE = "sbns-myapp-prod.servicebus.windows.net"
credential = DefaultAzureCredential()

# ── Producer ─────────────────────────────────────────────────────────────────
with ServiceBusClient(NAMESPACE, credential) as client:
    sender = client.get_queue_sender("q-orders")
    with sender:
        # Single message
        msg = ServiceBusMessage(
            '{"orderId": "ORD-001", "total": 99.95}',
            message_id="ORD-001",          # for duplicate detection
            session_id="customer-42",      # for session-based ordering
            subject="NewOrder",
            content_type="application/json",
        )
        sender.send_messages(msg)

        # Batch send
        batch = sender.create_message_batch()
        for i in range(100):
            batch.add_message(ServiceBusMessage(f"message-{i}"))
        sender.send_messages(batch)

        # Scheduled message
        scheduled_time = datetime.now(timezone.utc) + timedelta(hours=2)
        sender.schedule_messages(
            ServiceBusMessage("deferred payload"),
            scheduled_time,
        )

# ── Consumer (competing consumers) ───────────────────────────────────────────
with ServiceBusClient(NAMESPACE, credential) as client:
    receiver = client.get_queue_receiver("q-orders", max_wait_time=5)
    with receiver:
        for msg in receiver:
            try:
                process_order(str(msg))
                receiver.complete_message(msg)
            except TransientError:
                receiver.abandon_message(msg)   # returns to queue
            except PoisonError:
                receiver.dead_letter_message(   # sends to DLQ
                    msg,
                    reason="PoisonMessage",
                    error_description="Unrecoverable format error",
                )

# ── Session receiver (ordered processing per customer) ───────────────────────
with ServiceBusClient(NAMESPACE, credential) as client:
    session_receiver = client.get_queue_receiver(
        "q-orders",
        session_id="customer-42",   # or use next_available_session=True
    )
    with session_receiver:
        msgs = session_receiver.receive_messages(max_message_count=10)
        for msg in msgs:
            session_receiver.complete_message(msg)

# ── DLQ processing ───────────────────────────────────────────────────────────
with ServiceBusClient(NAMESPACE, credential) as client:
    dlq_receiver = client.get_queue_receiver(
        "q-orders",
        sub_queue=ServiceBusSubQueue.DEAD_LETTER,
    )
    with dlq_receiver:
        for msg in dlq_receiver:
            print(f"DLQ: {msg.dead_letter_reason} — {str(msg)}")
            dlq_receiver.complete_message(msg)  # remove from DLQ

# ── Topic/Subscription ───────────────────────────────────────────────────────
with ServiceBusClient(NAMESPACE, credential) as client:
    sender = client.get_topic_sender("t-notifications")
    with sender:
        msg = ServiceBusMessage(
            '{"userId": "u1", "message": "Hello"}',
            application_properties={"channel": "email"},
        )
        sender.send_messages(msg)

    receiver = client.get_subscription_receiver("t-notifications", "sub-email")
    with receiver:
        for msg in receiver:
            print(str(msg))
            receiver.complete_message(msg)
```

---

## Event Grid

### Concepts

- **System Topics** — built-in topics for Azure services (Storage, Resource Groups,
  Key Vault, Container Registry, etc.). No publisher config needed.
- **Custom Topics** — your own application events. You POST events to an HTTPS endpoint.
- **Event Domains** — manage thousands of topics under one endpoint; useful for multi-tenant.
- **Event Subscriptions** — routing rule: filter → destination.
- **Filters** — event type, subject prefix/suffix, advanced filters on JSON properties.
- **Dead-Lettering** — undeliverable events land in a blob container.
- **Retry Policy** — exponential backoff; configurable max attempts (1–30) and TTL (1–1440 min).
- **CloudEvents** — open standard schema (preferred over Event Grid schema).
- **Partner Events** — third-party SaaS events (Auth0, Tribal, SAP) natively integrated.

### Destinations

| Destination              | Use case                                    |
|--------------------------|---------------------------------------------|
| Azure Function           | Serverless event processing                 |
| Event Hub                | Stream events for analytics                 |
| Service Bus Queue/Topic  | Reliable ordered delivery                   |
| Storage Queue            | Simple queuing                              |
| Webhook (HTTPS)          | Custom HTTP endpoint                        |
| Logic App                | No-code workflow                            |
| Hybrid Connection        | On-prem relay                               |
| Partner Destination      | Third-party SaaS                            |

### az CLI — Event Grid

```bash
# Create custom topic (CloudEvents schema)
az eventgrid topic create \
  --resource-group rg-messaging \
  --name egt-myapp-events \
  --location eastus \
  --input-schema cloudeventschemav1_0 \
  --public-network-access enabled

# Get topic endpoint and key
az eventgrid topic show \
  --resource-group rg-messaging \
  --name egt-myapp-events \
  --query endpoint -o tsv

az eventgrid topic key list \
  --resource-group rg-messaging \
  --name egt-myapp-events \
  --query key1 -o tsv

# Create event subscription → Azure Function
az eventgrid event-subscription create \
  --name es-process-orders \
  --source-resource-id /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventGrid/topics/egt-myapp-events \
  --endpoint /subscriptions/<sub>/resourceGroups/rg-app/providers/Microsoft.Web/sites/func-myapp/functions/ProcessOrder \
  --endpoint-type azurefunction \
  --included-event-types OrderCreated OrderUpdated \
  --subject-begins-with /orders/ \
  --event-delivery-schema cloudeventschemav1_0 \
  --max-delivery-attempts 10 \
  --event-ttl 1440 \
  --deadletter-endpoint /subscriptions/<sub>/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/stmyapp/blobServices/default/containers/deadletter

# Create system topic for Storage Account events
az eventgrid system-topic create \
  --resource-group rg-messaging \
  --name egst-storage-events \
  --location eastus \
  --topic-type Microsoft.Storage.StorageAccounts \
  --source /subscriptions/<sub>/resourceGroups/rg-storage/providers/Microsoft.Storage/storageAccounts/stmyapp

# Subscribe to blob created events → Event Hub
az eventgrid system-topic event-subscription create \
  --resource-group rg-messaging \
  --system-topic-name egst-storage-events \
  --name es-blob-to-eventhub \
  --endpoint /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventHub/namespaces/evhns-myapp-prod/eventHubs/evh-storage-events \
  --endpoint-type eventhub \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/uploads/

# Advanced filter (only events where size > 1MB)
az eventgrid event-subscription update \
  --name es-process-orders \
  --source-resource-id /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventGrid/topics/egt-myapp-events \
  --advanced-filter data.size NumberGreaterThan 1048576

# Publish events to custom topic
ENDPOINT=$(az eventgrid topic show --resource-group rg-messaging --name egt-myapp-events --query endpoint -o tsv)
KEY=$(az eventgrid topic key list --resource-group rg-messaging --name egt-myapp-events --query key1 -o tsv)

curl -X POST "$ENDPOINT" \
  -H "Content-Type: application/cloudevents+json" \
  -H "aeg-sas-key: $KEY" \
  -d '{
    "specversion": "1.0",
    "type": "com.myapp.order.created",
    "source": "/orders/service",
    "id": "ORD-001",
    "subject": "/orders/ORD-001",
    "datacontenttype": "application/json",
    "data": {"orderId": "ORD-001", "total": 99.95}
  }'
```

### Python SDK — Event Grid

```python
# pip install azure-eventgrid azure-identity

from azure.eventgrid import EventGridPublisherClient
from azure.eventgrid.models import CloudEvent
from azure.core.credentials import AzureKeyCredential
from azure.identity import DefaultAzureCredential
import datetime, uuid

ENDPOINT = "https://egt-myapp-events.eastus-1.eventgrid.azure.net/api/events"
KEY = "<topic-key>"

client = EventGridPublisherClient(ENDPOINT, AzureKeyCredential(KEY))

events = [
    CloudEvent(
        type="com.myapp.order.created",
        source="/orders/service",
        subject=f"/orders/ORD-{i}",
        data={"orderId": f"ORD-{i}", "total": i * 10.0},
        id=str(uuid.uuid4()),
        time=datetime.datetime.now(datetime.timezone.utc),
        datacontenttype="application/json",
    )
    for i in range(10)
]

# Batch publish (recommended)
client.send(events)
```

---

## Queue Storage

### Concepts

Azure Queue Storage provides simple, durable messaging for decoupling components.
Best for high-volume, low-cost scenarios where ordering and transactions aren't needed.

- **Max message size:** 64 KB
- **Max queue size:** 500 TB
- **Retention:** up to 7 days (default)
- **Visibility timeout:** message hidden from other consumers while being processed
- No dead-letter queue, no sessions, no ordering guarantees

### Queue Storage vs Service Bus Queues

| Feature              | Queue Storage     | Service Bus Queue     |
|----------------------|-------------------|-----------------------|
| Message size         | 64 KB             | 256 KB / 100 MB (Prem)|
| Retention            | ≤ 7 days          | Configurable, forever |
| Dead-letter          | ✗                 | ✓                     |
| Sessions             | ✗                 | ✓                     |
| Transactions         | ✗                 | ✓                     |
| Ordering             | Approximate       | Guaranteed (sessions) |
| Max queue depth      | Unlimited         | 80 GB                 |
| Cost                 | Very low          | Higher                |
| Protocol             | HTTP/HTTPS        | AMQP / HTTP           |

**Use Queue Storage when:** you need cheap, simple decoupling, audit logs via storage
diagnostics, or queue depth > 80 GB.

### az CLI — Queue Storage

```bash
# Create storage account and queue
az storage account create \
  --resource-group rg-messaging \
  --name stmessaging \
  --location eastus \
  --sku Standard_LRS

az storage queue create \
  --name q-tasks \
  --account-name stmessaging \
  --auth-mode login

# Send a message (Base64 encoded automatically)
az storage message put \
  --queue-name q-tasks \
  --account-name stmessaging \
  --content '{"taskId": "T-001", "action": "resize"}' \
  --time-to-live 3600 \
  --auth-mode login

# Peek without dequeuing
az storage message peek \
  --queue-name q-tasks \
  --account-name stmessaging \
  --auth-mode login

# Dequeue (get + visibility timeout)
az storage message get \
  --queue-name q-tasks \
  --account-name stmessaging \
  --visibility-timeout 30 \
  --auth-mode login

# Delete after processing
az storage message delete \
  --queue-name q-tasks \
  --account-name stmessaging \
  --id <message-id> \
  --pop-receipt <pop-receipt> \
  --auth-mode login

# Queue approximate message count
az storage queue metadata show \
  --name q-tasks \
  --account-name stmessaging \
  --auth-mode login
```

### Python SDK — Queue Storage

```python
# pip install azure-storage-queue azure-identity

from azure.storage.queue import QueueClient
from azure.identity import DefaultAzureCredential
import json, base64

client = QueueClient(
    account_url="https://stmessaging.queue.core.windows.net",
    queue_name="q-tasks",
    credential=DefaultAzureCredential(),
)

# Send
payload = json.dumps({"taskId": "T-001", "action": "resize"})
client.send_message(payload, time_to_live=3600, visibility_timeout=0)

# Receive and process
messages = client.receive_messages(max_messages=32, visibility_timeout=30)
for msg in messages:
    task = json.loads(msg.content)
    print(f"Processing: {task}")
    client.delete_message(msg)
```

---

## Security

### Managed Identity (Recommended)

Always prefer Managed Identity over connection strings/SAS tokens in production.

```bash
# Assign Event Hubs Data Sender to app's managed identity
az role assignment create \
  --role "Azure Event Hubs Data Sender" \
  --assignee <managed-identity-object-id> \
  --scope /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventHub/namespaces/evhns-myapp-prod

# Assign Service Bus Data Owner to function app
az role assignment create \
  --role "Azure Service Bus Data Owner" \
  --assignee <object-id> \
  --scope /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.ServiceBus/namespaces/sbns-myapp-prod
```

### RBAC Roles

| Service      | Role                              | Permissions                  |
|--------------|-----------------------------------|------------------------------|
| Event Hubs   | Azure Event Hubs Data Sender      | Send events                  |
| Event Hubs   | Azure Event Hubs Data Receiver    | Receive events               |
| Event Hubs   | Azure Event Hubs Data Owner       | Full access                  |
| Service Bus  | Azure Service Bus Data Sender     | Send messages                |
| Service Bus  | Azure Service Bus Data Receiver   | Receive/peek messages        |
| Service Bus  | Azure Service Bus Data Owner      | Full access (incl. DLQ)      |
| Event Grid   | EventGrid Contributor             | Manage topics/subscriptions  |
| Event Grid   | EventGrid Data Sender             | Publish events               |
| Queue Storage| Storage Queue Data Contributor    | Send/receive/delete          |
| Queue Storage| Storage Queue Data Message Sender | Send only                    |

### SAS Tokens

Use SAS for short-lived access when Managed Identity isn't available:

```bash
# Event Hubs namespace-level SAS
az eventhubs namespace authorization-rule create \
  --resource-group rg-messaging \
  --namespace-name evhns-myapp-prod \
  --name sender-only \
  --rights Send

# Service Bus queue-level SAS
az servicebus queue authorization-rule create \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --queue-name q-orders \
  --name app-sender \
  --rights Send
```

### Network Security

```bash
# Event Hubs — restrict to specific VNet subnet
az eventhubs namespace network-rule-set update \
  --resource-group rg-messaging \
  --namespace-name evhns-myapp-prod \
  --default-action Deny

az eventhubs namespace network-rule-set virtual-network-rule add \
  --resource-group rg-messaging \
  --namespace-name evhns-myapp-prod \
  --vnet-name vnet-prod \
  --subnet app-subnet

# Service Bus — add IP firewall rule
az servicebus namespace network-rule-set ip-rule add \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --ip-address 203.0.113.0/24 \
  --action Allow

# Private endpoint for Event Hubs
az network private-endpoint create \
  --resource-group rg-messaging \
  --name pe-eventhubs \
  --vnet-name vnet-prod \
  --subnet pe-subnet \
  --private-connection-resource-id /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventHub/namespaces/evhns-myapp-prod \
  --group-id namespace \
  --connection-name pec-eventhubs
```

---

## Monitoring

### Key Metrics

**Event Hubs:**
- `IncomingMessages` / `OutgoingMessages` — throughput
- `IncomingBytes` / `OutgoingBytes` — data volume
- `ThrottledRequests` — TU exhaustion (scale up if non-zero)
- `CapturedMessages` / `CapturedBytes` — capture progress
- `ActiveConnections` — AMQP connections
- `Errors` — server/user errors split

**Service Bus:**
- `IncomingMessages` / `OutgoingMessages`
- `ActiveMessages` — queue depth (alert on growth)
- `DeadLetteredMessages` — DLQ depth (alert if > 0)
- `ScheduledMessages`
- `ServerErrors` / `UserErrors`
- `ThrottledRequests`

**Event Grid:**
- `PublishSuccessCount` / `PublishFailCount`
- `DeliverySuccessCount` / `DeliveryFailCount`
- `DeadLetteredCount`
- `MatchedEventCount`

### az CLI — Monitoring

```bash
# Enable diagnostic logs — Event Hubs to Log Analytics
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventHub/namespaces/evhns-myapp-prod \
  --name diag-eventhubs \
  --workspace /subscriptions/<sub>/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-myapp \
  --logs '[{"category":"OperationalLogs","enabled":true},{"category":"AutoScaleLogs","enabled":true},{"category":"KafkaCoordinatorLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Alert on Service Bus DLQ depth
az monitor metrics alert create \
  --resource-group rg-messaging \
  --name alert-sb-dlq \
  --scopes /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.ServiceBus/namespaces/sbns-myapp-prod \
  --condition "avg DeadLetteredMessages > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action /subscriptions/<sub>/resourceGroups/rg-monitoring/providers/microsoft.insights/actionGroups/ag-ops \
  --description "Service Bus DLQ has messages"

# Alert on Event Hubs throttling
az monitor metrics alert create \
  --resource-group rg-messaging \
  --name alert-evh-throttle \
  --scopes /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventHub/namespaces/evhns-myapp-prod \
  --condition "total ThrottledRequests > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action /subscriptions/<sub>/resourceGroups/rg-monitoring/providers/microsoft.insights/actionGroups/ag-ops

# Query Log Analytics for Event Hubs errors
az monitor log-analytics query \
  --workspace /subscriptions/<sub>/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-myapp \
  --analytics-query "AzureDiagnostics | where ResourceType == 'EVENTHUBS' and Level == 'Error' | summarize count() by bin(TimeGenerated, 5m)" \
  --timespan P1D
```

---

## Performance Tuning

### Event Hubs

- **Partition count** — set at creation, immutable on Standard (choose 8–32 for throughput).
  More partitions = more parallelism, but also more consumer complexity.
- **Partition keys** — use consistent keys to group related events; avoids cross-partition ordering issues.
- **Batching** — always use `create_batch()` / `EventDataBatch`; reduces API calls by 10–100×.
- **Prefetch** — `EventHubConsumerClient(prefetch=300)` buffers messages locally.
- **Checkpointing frequency** — checkpoint after N events (not every event) to reduce blob calls.
- **TU scaling** — enable auto-inflate; set max TUs to 2–3× your peak.
- **Premium tier** — dedicated resources eliminate noisy-neighbor; prefer for <200 ms P99 latency.
- **Capture format** — Avro is compact; use Schema Registry for schema evolution.

### Service Bus

- **Prefetch count** — `ServiceBusReceiver(prefetch_count=100)` reduces round trips.
- **Batch receive** — `receive_messages(max_message_count=100)` in a loop.
- **Lock duration** — set slightly longer than your processing time; renew if needed with `renew_message_lock()`.
- **Max delivery count** — tune to your retry budget; 5–10 is typical.
- **Partitioned entities** — enable on Standard tier for higher throughput (16 partitions).
  Note: sessions and transactions are incompatible with partitioned entities.
- **Premium scaling** — add messaging units (1, 2, 4, 8, 16) for dedicated capacity.
- **Connection pooling** — reuse `ServiceBusClient`; it manages AMQP connections internally.

### Event Grid

- **Batching on webhook destinations** — configure `maxEventsPerBatch` (up to 5000) and
  `preferredBatchSizeInKilobytes` to reduce HTTP calls.
- **Retry policy** — increase `maxDeliveryAttempts` to 30 and `eventTimeToLiveInMinutes` to 1440
  for critical events.
- **Dead-letter** — always configure a dead-letter blob container in production.
- **Advanced filters** — filter early at Event Grid (not in your function) to reduce invocations.

---

## Infrastructure as Code

### Bicep

```bicep
// messaging.bicep
param location string = resourceGroup().location
param appName string = 'myapp'
param env string = 'prod'

// ── Event Hubs ────────────────────────────────────────────────────────────────
resource eventHubNamespace 'Microsoft.EventHub/namespaces@2023-01-01-preview' = {
  name: 'evhns-${appName}-${env}'
  location: location
  sku: {
    name: 'Standard'
    tier: 'Standard'
    capacity: 2
  }
  properties: {
    isAutoInflateEnabled: true
    maximumThroughputUnits: 10
    zoneRedundant: true
    kafkaEnabled: true
  }
}

resource eventHub 'Microsoft.EventHub/namespaces/eventhubs@2023-01-01-preview' = {
  parent: eventHubNamespace
  name: 'evh-telemetry'
  properties: {
    partitionCount: 8
    messageRetentionInDays: 3
    captureDescription: {
      enabled: true
      encoding: 'Avro'
      intervalInSeconds: 300
      sizeLimitInBytes: 314572800
      destination: {
        name: 'EventHubArchive.AzureBlockBlob'
        properties: {
          storageAccountResourceId: storageAccount.id
          blobContainer: 'capture-archive'
          archiveNameFormat: '{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}'
        }
      }
    }
  }
}

resource consumerGroup 'Microsoft.EventHub/namespaces/eventhubs/consumergroups@2023-01-01-preview' = {
  parent: eventHub
  name: 'cg-analytics'
}

// ── Service Bus ───────────────────────────────────────────────────────────────
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sbns-${appName}-${env}'
  location: location
  sku: {
    name: 'Premium'
    tier: 'Premium'
    capacity: 1
  }
  properties: {
    zoneRedundant: true
  }
}

resource ordersQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: serviceBusNamespace
  name: 'q-orders'
  properties: {
    lockDuration: 'PT2M'
    maxSizeInMegabytes: 5120
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT10M'
    requiresSession: true
    defaultMessageTimeToLive: 'P14D'
    deadLetteringOnMessageExpiration: true
    maxDeliveryCount: 5
  }
}

resource notificationsTopic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: serviceBusNamespace
  name: 't-notifications'
  properties: {
    defaultMessageTimeToLive: 'P7D'
    requiresDuplicateDetection: true
    maxSizeInMegabytes: 5120
  }
}

resource emailSubscription 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: notificationsTopic
  name: 'sub-email'
  properties: {
    lockDuration: 'PT1M'
    maxDeliveryCount: 10
    deadLetteringOnMessageExpiration: true
  }
}

// ── Event Grid ────────────────────────────────────────────────────────────────
resource eventGridTopic 'Microsoft.EventGrid/topics@2023-12-15-preview' = {
  name: 'egt-${appName}-events'
  location: location
  properties: {
    inputSchema: 'CloudEventSchemaV1_0'
    publicNetworkAccess: 'Enabled'
  }
}

// ── Storage Account + Queue ───────────────────────────────────────────────────
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'st${appName}${env}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

resource queueService 'Microsoft.Storage/storageAccounts/queueServices@2023-01-01' = {
  parent: storageAccount
  name: 'default'
}

resource tasksQueue 'Microsoft.Storage/storageAccounts/queueServices/queues@2023-01-01' = {
  parent: queueService
  name: 'q-tasks'
}

// ── RBAC: Managed Identity → Event Hubs Data Sender ──────────────────────────
param appManagedIdentityObjectId string

resource ehSenderRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(eventHubNamespace.id, appManagedIdentityObjectId, 'EventHubs-DataSender')
  scope: eventHubNamespace
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '2b629674-e913-4c01-ae53-ef4638d8f975')
    principalId: appManagedIdentityObjectId
    principalType: 'ServicePrincipal'
  }
}
```

### Terraform

```hcl
# messaging.tf

locals {
  app_name = "myapp"
  env      = "prod"
  location = "East US"
}

# ── Event Hubs ────────────────────────────────────────────────────────────────
resource "azurerm_eventhub_namespace" "this" {
  name                     = "evhns-${local.app_name}-${local.env}"
  location                 = local.location
  resource_group_name      = var.resource_group_name
  sku                      = "Standard"
  capacity                 = 2
  auto_inflate_enabled     = true
  maximum_throughput_units = 10
  zone_redundant           = true
  kafka_enabled            = true
}

resource "azurerm_eventhub" "telemetry" {
  name                = "evh-telemetry"
  namespace_name      = azurerm_eventhub_namespace.this.name
  resource_group_name = var.resource_group_name
  partition_count     = 8
  message_retention   = 3

  capture_description {
    enabled             = true
    encoding            = "Avro"
    interval_in_seconds = 300
    size_limit_in_bytes = 314572800
    destination {
      name                = "EventHubArchive.AzureBlockBlob"
      archive_name_format = "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"
      blob_container_name = "capture-archive"
      storage_account_id  = azurerm_storage_account.this.id
    }
  }
}

resource "azurerm_eventhub_consumer_group" "analytics" {
  name                = "cg-analytics"
  namespace_name      = azurerm_eventhub_namespace.this.name
  eventhub_name       = azurerm_eventhub.telemetry.name
  resource_group_name = var.resource_group_name
}

# ── Service Bus ───────────────────────────────────────────────────────────────
resource "azurerm_servicebus_namespace" "this" {
  name                = "sbns-${local.app_name}-${local.env}"
  location            = local.location
  resource_group_name = var.resource_group_name
  sku                 = "Premium"
  capacity            = 1
  zone_redundant      = true
}

resource "azurerm_servicebus_queue" "orders" {
  name         = "q-orders"
  namespace_id = azurerm_servicebus_namespace.this.id

  lock_duration                           = "PT2M"
  max_size_in_megabytes                   = 5120
  requires_duplicate_detection            = true
  duplicate_detection_history_time_window = "PT10M"
  requires_session                        = true
  default_message_ttl                     = "P14D"
  dead_lettering_on_message_expiration    = true
  max_delivery_count                      = 5
}

resource "azurerm_servicebus_topic" "notifications" {
  name                      = "t-notifications"
  namespace_id              = azurerm_servicebus_namespace.this.id
  default_message_ttl       = "P7D"
  requires_duplicate_detection = true
  max_size_in_megabytes     = 5120
}

resource "azurerm_servicebus_subscription" "email" {
  name               = "sub-email"
  topic_id           = azurerm_servicebus_topic.notifications.id
  lock_duration      = "PT1M"
  max_delivery_count = 10
  dead_lettering_on_message_expiration = true
}

resource "azurerm_servicebus_subscription_rule" "email_filter" {
  name            = "filter-email"
  subscription_id = azurerm_servicebus_subscription.email.id
  filter_type     = "SqlFilter"
  sql_filter      = "channel = 'email'"
}

# ── Event Grid ────────────────────────────────────────────────────────────────
resource "azurerm_eventgrid_topic" "this" {
  name                = "egt-${local.app_name}-events"
  location            = local.location
  resource_group_name = var.resource_group_name
  input_schema        = "CloudEventSchemaV1_0"
}

# ── Storage + Queue ───────────────────────────────────────────────────────────
resource "azurerm_storage_account" "this" {
  name                     = "st${local.app_name}${local.env}"
  resource_group_name      = var.resource_group_name
  location                 = local.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_queue" "tasks" {
  name                 = "q-tasks"
  storage_account_name = azurerm_storage_account.this.name
}

# ── RBAC ─────────────────────────────────────────────────────────────────────
data "azurerm_role_definition" "eh_sender" {
  name = "Azure Event Hubs Data Sender"
}

resource "azurerm_role_assignment" "app_eh_sender" {
  scope                = azurerm_eventhub_namespace.this.id
  role_definition_id   = data.azurerm_role_definition.eh_sender.id
  principal_id         = var.app_managed_identity_object_id
}
```

---

## Quick Reference

```bash
# Check Event Hubs namespace metrics (last hour)
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventHub/namespaces/evhns-myapp-prod \
  --metric ThrottledRequests IncomingMessages \
  --interval PT5M \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)

# Check Service Bus active + DLQ message counts
az servicebus queue show \
  --resource-group rg-messaging \
  --namespace-name sbns-myapp-prod \
  --name q-orders \
  --query "{active:countDetails.activeMessageCount, dlq:countDetails.deadLetterMessageCount}"

# Purge Event Grid subscription dead-letter
az eventgrid event-subscription show --name es-process-orders \
  --source-resource-id /subscriptions/<sub>/resourceGroups/rg-messaging/providers/Microsoft.EventGrid/topics/egt-myapp-events \
  --query deadLetterDestination

# List all Service Bus entities in namespace
az servicebus queue list --resource-group rg-messaging --namespace-name sbns-myapp-prod -o table
az servicebus topic list --resource-group rg-messaging --namespace-name sbns-myapp-prod -o table
```
