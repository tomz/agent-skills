---
name: azure-communication
description: Azure Communication Services — SMS, email, voice calling, video calling, chat, phone numbers, rooms, job router, Teams interop
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Communication Services (ACS)

## Overview

Azure Communication Services (ACS) is a cloud-based communications platform providing
SMS, email, voice, video, chat, and Teams interoperability via REST APIs and SDKs.

---

## Resource Creation & Connection Strings

```bash
# Create resource group
az group create --name rg-comms --location eastus

# Create ACS resource
az communication create \
  --name myCommsService \
  --resource-group rg-comms \
  --location global \
  --data-location unitedstates

# Get connection string
az communication list-key \
  --name myCommsService \
  --resource-group rg-comms \
  --query primaryConnectionString -o tsv

# Get endpoint
az communication show \
  --name myCommsService \
  --resource-group rg-comms \
  --query hostName -o tsv
```

Connection string format:
```
endpoint=https://<resource>.communication.azure.com/;accesskey=<base64key>
```

---

## SMS

### Send SMS (Python)

```python
from azure.communication.sms import SmsClient

client = SmsClient.from_connection_string("<connection_string>")

response = client.send(
    from_="+18005551234",          # ACS-acquired number
    to=["+12065550100", "+14255550101"],
    message="Hello from Azure ACS!",
    enable_delivery_report=True,
    tag="campaign-spring-2026"
)

for item in response:
    print(f"To: {item.to}, Status: {item.http_status_code}, MessageId: {item.message_id}")
```

### Receive SMS via Event Grid Webhook

```python
# FastAPI webhook handler
from fastapi import FastAPI, Request
import json

app = FastAPI()

@app.post("/sms-webhook")
async def receive_sms(request: Request):
    events = await request.json()
    for event in events:
        # Respond to Event Grid validation challenge
        if event.get("eventType") == "Microsoft.EventGrid.SubscriptionValidationEvent":
            return {"validationResponse": event["data"]["validationCode"]}

        if event["eventType"] == "Microsoft.Communication.SMSReceived":
            data = event["data"]
            print(f"From: {data['from']}, To: {data['to']}, Message: {data['message']}")
    return {"status": "ok"}
```

### Opt-Out Management

ACS automatically handles STOP/START keywords for toll-free numbers. For short codes:

```bash
# List opt-outs
az communication sms optout show \
  --endpoint https://<resource>.communication.azure.com \
  --from "+18005551234" \
  --to "+12065550100"
```

### Toll-Free Verification

```bash
# Submit toll-free verification (required for US/CA production SMS)
az communication tollfree-verification create \
  --resource-group rg-comms \
  --communication-service-name myCommsService \
  --phone-number "+18005551234" \
  --business-name "Contoso Ltd" \
  --use-case-summary "Account notifications and 2FA codes" \
  --use-case-description "We send OTPs and transactional alerts to registered users."
```

---

## Email

### Domain Setup

```bash
# Use Azure-managed domain (instant, e.g. <guid>.azurecomm.net)
az communication email domain create \
  --name AzureManagedDomain \
  --resource-group rg-comms \
  --email-service-name myEmailService \
  --location global \
  --domain-management AzureManaged

# Use custom domain
az communication email domain create \
  --name mail.contoso.com \
  --resource-group rg-comms \
  --email-service-name myEmailService \
  --location global \
  --domain-management CustomerManaged

# Get DNS records to configure (SPF, DKIM, DMARC)
az communication email domain show \
  --name mail.contoso.com \
  --resource-group rg-comms \
  --email-service-name myEmailService \
  --query "verificationRecords"

# Initiate domain verification after DNS records are set
az communication email domain initiate-verification \
  --name mail.contoso.com \
  --resource-group rg-comms \
  --email-service-name myEmailService \
  --verification-type Domain
```

### Send Email (Python)

```python
from azure.communication.email import EmailClient
from azure.communication.email.models import EmailMessage, EmailAddress, EmailContent

client = EmailClient.from_connection_string("<connection_string>")

message = EmailMessage(
    sender_address="DoNotReply@<guid>.azurecomm.net",
    recipients={
        "to": [EmailAddress(address="user@contoso.com", display_name="Jane Doe")],
        "cc": [EmailAddress(address="manager@contoso.com")],
    },
    content=EmailContent(
        subject="Your Invoice #1042",
        plain_text="Please find your invoice attached.",
        html="<h2>Invoice #1042</h2><p>Please find your invoice attached.</p>",
    ),
    attachments=[
        {
            "name": "invoice.pdf",
            "contentType": "application/pdf",
            "contentInBase64": "<base64_encoded_pdf>",
        }
    ],
    user_engagement_tracking_disabled=False,  # enables open tracking
)

poller = client.begin_send(message)
result = poller.result()
print(f"Message ID: {result['id']}, Status: {result['status']}")
```

### Sender Addresses

```bash
# Add a sender username to a domain
az communication email sender-username create \
  --sender-username "notifications" \
  --username "notifications" \
  --display-name "Contoso Notifications" \
  --domain-name mail.contoso.com \
  --resource-group rg-comms \
  --email-service-name myEmailService
```

---

## Voice & PSTN

### Phone Number Management

```bash
# Search available toll-free numbers
az communication phone-number search-plan-create \
  --resource-group rg-comms \
  --communication-service-name myCommsService \
  --phone-plan-group-id "toll-free" \
  --phone-plan-id "Calling" \
  --area-code "800" \
  --quantity 1

# List acquired numbers
az communication phone-number list \
  --resource-group rg-comms \
  --communication-service-name myCommsService

# Purchase a number (after search)
az communication phone-number purchase \
  --resource-group rg-comms \
  --communication-service-name myCommsService \
  --search-id "<search_id>"

# Release a number
az communication phone-number delete \
  --resource-group rg-comms \
  --communication-service-name myCommsService \
  --phone-number "+18005551234"
```

### Call Automation (Python)

```python
from azure.communication.callautomation import CallAutomationClient, CallConnectionClient
from azure.communication.callautomation.models import (
    PhoneNumberIdentifier, CallInvite, FileSource, TextSource,
    RecognizeInputType, DtmfTone
)

client = CallAutomationClient.from_connection_string("<connection_string>")

# Create outbound call
target = PhoneNumberIdentifier("+12065550100")
source = PhoneNumberIdentifier("+18005551234")

result = client.create_call(
    target_participant=CallInvite(target=target, source_caller_id_number=source),
    callback_url="https://myapp.example.com/events",
)
call_connection_id = result.call_connection_id

# Answer inbound call (from Event Grid webhook)
incoming_call_context = "<incoming_call_context_from_event>"
result = client.answer_call(
    incoming_call_context=incoming_call_context,
    callback_url="https://myapp.example.com/events",
)

# Play media / TTS
connection: CallConnectionClient = client.get_call_connection(call_connection_id)
connection.play_media_to_all(
    play_source=TextSource(text="Press 1 for sales, press 2 for support.", voice_name="en-US-JennyNeural"),
)

# Recognize DTMF
connection.start_recognizing_media(
    input_type=RecognizeInputType.DTMF,
    target_participant=target,
    play_prompt=TextSource(text="Enter your 4-digit PIN."),
    dtmf_max_tones_to_collect=4,
    interrupt_prompt=True,
)

# Recording
recording = client.start_recording(server_call_id="<server_call_id>")
client.stop_recording(recording_id=recording.recording_id)
```

### Direct Routing

```bash
# Register a Session Border Controller (SBC)
az communication direct-routing gateway create \
  --resource-group rg-comms \
  --communication-service-name myCommsService \
  --name sbc.contoso.com \
  --fqdn sbc.contoso.com \
  --sip-signaling-port 5061
```

---

## Video Calling

### Rooms (Server-Side)

```python
from azure.communication.rooms import RoomsClient
from datetime import datetime, timedelta, timezone

client = RoomsClient.from_connection_string("<connection_string>")

# Create a room with a time window
room = client.create_room(
    valid_from=datetime.now(timezone.utc),
    valid_until=datetime.now(timezone.utc) + timedelta(hours=2),
)
print(f"Room ID: {room.id}")

# Add participants with roles (Presenter, Attendee, Consumer)
client.add_or_update_participants(
    room_id=room.id,
    participants=[
        {"communication_identifier": {"communication_user": {"id": "<acs_user_id>"}},
         "role": "Presenter"},
        {"communication_identifier": {"communication_user": {"id": "<acs_user_id_2>"}},
         "role": "Attendee"},
    ]
)
```

### Client SDK — JavaScript (React)

```typescript
import { CallClient, VideoStreamRenderer } from "@azure/communication-calling";
import { AzureCommunicationTokenCredential } from "@azure/communication-common";

const callClient = new CallClient();
const tokenCredential = new AzureCommunicationTokenCredential("<user_access_token>");
const callAgent = await callClient.createCallAgent(tokenCredential, { displayName: "Jane" });
const deviceManager = await callClient.getDeviceManager();
await deviceManager.askDevicePermission({ video: true, audio: true });

// Join a Room call
const call = callAgent.join({ roomId: "<room_id>" });

// Subscribe to remote participants
call.on("remoteParticipantsUpdated", (e) => {
    e.added.forEach(async (participant) => {
        participant.videoStreams.forEach(async (stream) => {
            if (stream.isAvailable) {
                const renderer = new VideoStreamRenderer(stream);
                const view = await renderer.createView();
                document.getElementById("remote-video").appendChild(view.target);
            }
        });
    });
});
```

### UI Library (React)

```tsx
import { CallComposite, useAzureCommunicationCallAdapter } from "@azure/communication-react";
import { AzureCommunicationTokenCredential } from "@azure/communication-common";

const adapter = useAzureCommunicationCallAdapter({
    userId: { communicationUserId: "<acs_user_id>" },
    displayName: "Jane Doe",
    credential: new AzureCommunicationTokenCredential("<token>"),
    locator: { roomId: "<room_id>" },      // or { groupId: "<guid>" } for group calls
});

return <CallComposite adapter={adapter} />;
```

---

## Chat

### Create Thread & Send Messages (Python)

```python
from azure.communication.chat import ChatClient, ChatThreadClient, CommunicationTokenCredential

credential = CommunicationTokenCredential("<user_access_token>")
chat_client = ChatClient(endpoint="https://<resource>.communication.azure.com", credential=credential)

# Create a thread
thread = chat_client.create_chat_thread(
    topic="Project Phoenix",
    thread_participants=[
        {"user": {"communication_user": {"id": "<user_id_2>"}}, "display_name": "Bob"}
    ]
)
thread_id = thread.chat_thread.id

thread_client: ChatThreadClient = chat_client.get_chat_thread_client(thread_id)

# Send message
thread_client.send_message(content="Hello team!", sender_display_name="Jane")

# Send typing indicator
thread_client.send_typing_notification()

# Send read receipt
thread_client.send_read_receipt(message_id="<message_id>")

# List messages
for msg in thread_client.list_messages():
    print(f"{msg.sender_display_name}: {msg.content.message}")

# Add participant
thread_client.add_participants([
    {"user": {"communication_user": {"id": "<user_id_3>"}}, "display_name": "Carol"}
])
```

### Real-Time Notifications (JavaScript)

```typescript
import { ChatClient } from "@azure/communication-chat";
import { AzureCommunicationTokenCredential } from "@azure/communication-common";

const chatClient = new ChatClient(
    "https://<resource>.communication.azure.com",
    new AzureCommunicationTokenCredential("<token>")
);

await chatClient.startRealtimeNotifications();

chatClient.on("chatMessageReceived", (event) => {
    console.log(`New message from ${event.senderDisplayName}: ${event.message}`);
});

chatClient.on("typingIndicatorReceived", (event) => {
    console.log(`${event.senderDisplayName} is typing...`);
});
```

---

## Job Router

```python
from azure.communication.jobrouter import JobRouterAdministrationClient, JobRouterClient

admin_client = JobRouterAdministrationClient.from_connection_string("<connection_string>")
client = JobRouterClient.from_connection_string("<connection_string>")

# Create distribution policy (how to distribute jobs to workers)
admin_client.upsert_distribution_policy(
    distribution_policy_id="round-robin",
    offer_expires_after_seconds=60,
    mode={"kind": "round-robin"},
    name="Round Robin Policy",
)

# Create queue
admin_client.upsert_queue(
    queue_id="support-queue",
    distribution_policy_id="round-robin",
    name="Support Queue",
    labels={"department": "support"},
)

# Create classification policy
admin_client.upsert_classification_policy(
    classification_policy_id="priority-routing",
    name="Priority Routing",
    fallback_queue_id="support-queue",
    queue_selectors=[
        {"kind": "static", "label_selector": {"key": "department", "label_operator": "equal", "value": "support"}}
    ],
    prioritization_rule={"kind": "expression", "expression": "If(job.priority == 'high', 10, 1)"},
)

# Register a worker
client.upsert_worker(
    worker_id="worker-001",
    capacity=5,
    queues=["support-queue"],
    labels={"skills": ["billing", "general"]},
    channels=[{"channel_id": "voice", "capacity_cost_per_job": 1}],
    available_for_offers=True,
)

# Submit a job
client.upsert_job(
    job_id="job-001",
    channel_id="voice",
    queue_id="support-queue",
    priority=5,
    labels={"issue_type": "billing"},
)

# Accept a job offer (worker side)
client.accept_job_offer(worker_id="worker-001", offer_id="<offer_id>")

# Complete and close a job
client.complete_job(job_id="job-001", assignment_id="<assignment_id>")
client.close_job(job_id="job-001", assignment_id="<assignment_id>")
```

---

## Teams Interop

```python
from azure.communication.identity import CommunicationIdentityClient

identity_client = CommunicationIdentityClient.from_connection_string("<connection_string>")

# Create ACS user and token with Teams meeting scope
user, token = identity_client.create_user_and_token(scopes=["voip"])

# Join a Teams meeting URL using the Calling SDK (JavaScript)
# teamsMeetingLocator = { meetingLink: "https://teams.microsoft.com/l/meetup-join/..." }
# callAgent.join(teamsMeetingLocator)
```

### Teams Phone (Direct Routing via ACS)

```bash
az communication teams-extension create \
  --resource-group rg-comms \
  --communication-service-name myCommsService \
  --teams-tenant-id "<tenant_id>"
```

---

## User Access Tokens

```python
from azure.communication.identity import CommunicationIdentityClient, CommunicationTokenScope

identity_client = CommunicationIdentityClient.from_connection_string("<connection_string>")

# Create user
user = identity_client.create_user()

# Issue token (scopes: voip, chat)
token_response = identity_client.get_token(user, scopes=[CommunicationTokenScope.VOIP, CommunicationTokenScope.CHAT])
print(f"Token: {token_response.token}, Expires: {token_response.expires_on}")

# Revoke all tokens for a user
identity_client.revoke_tokens(user)

# Delete user
identity_client.delete_user(user)
```

---

## Event Grid Integration

| Event Type | Description |
|------------|-------------|
| `Microsoft.Communication.SMSReceived` | Inbound SMS message |
| `Microsoft.Communication.SMSDeliveryReportReceived` | Delivery receipt |
| `Microsoft.Communication.CallStarted` | Call started |
| `Microsoft.Communication.CallEnded` | Call ended |
| `Microsoft.Communication.CallParticipantAdded` | Participant joined |
| `Microsoft.Communication.CallParticipantRemoved` | Participant left |
| `Microsoft.Communication.RecordingFileStatusUpdated` | Recording ready |
| `Microsoft.Communication.EmailDeliveryReportReceived` | Email delivery status |
| `Microsoft.Communication.EmailEngagementTrackingReportReceived` | Email open/click |
| `Microsoft.Communication.RouterWorkerOfferIssued` | Job offered to worker |
| `Microsoft.Communication.RouterJobCompleted` | Job completed |

```bash
# Subscribe to ACS events via Event Grid
az eventgrid event-subscription create \
  --name acs-sms-subscription \
  --source-resource-id "/subscriptions/<sub>/resourceGroups/rg-comms/providers/Microsoft.Communication/communicationServices/myCommsService" \
  --endpoint-type webhook \
  --endpoint "https://myapp.example.com/sms-webhook" \
  --included-event-types "Microsoft.Communication.SMSReceived" \
                         "Microsoft.Communication.SMSDeliveryReportReceived"
```

---

## Monitoring & Diagnostics

```bash
# Enable diagnostic logs (to Log Analytics)
az monitor diagnostic-settings create \
  --name acs-diag \
  --resource "/subscriptions/<sub>/resourceGroups/rg-comms/providers/Microsoft.Communication/communicationServices/myCommsService" \
  --workspace "/subscriptions/<sub>/resourceGroups/rg-logs/providers/Microsoft.OperationalInsights/workspaces/myWorkspace" \
  --logs '[{"category":"SMSOperationalLogs","enabled":true},
           {"category":"ChatOperationalLogs","enabled":true},
           {"category":"VoIPOperationalLogs","enabled":true},
           {"category":"CallDiagnostics","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

### Call Diagnostics KQL Query

```kusto
ACSCallDiagnostics
| where TimeGenerated > ago(1h)
| where CallEndSubCode != 0    // non-zero = error
| project TimeGenerated, CallId, CallEndSubCode, ParticipantId, MediaType
| order by TimeGenerated desc
```

---

## Bicep Example

```bicep
param location string = 'global'
param dataLocation string = 'unitedstates'

resource acsResource 'Microsoft.Communication/communicationServices@2023-04-01' = {
  name: 'myCommsService'
  location: location
  properties: {
    dataLocation: dataLocation
  }
}

resource emailService 'Microsoft.Communication/emailServices@2023-04-01' = {
  name: 'myEmailService'
  location: location
  properties: {
    dataLocation: dataLocation
  }
}

resource emailDomain 'Microsoft.Communication/emailServices/domains@2023-04-01' = {
  parent: emailService
  name: 'AzureManagedDomain'
  location: location
  properties: {
    domainManagement: 'AzureManaged'
    userEngagementTracking: 'Enabled'
  }
}

output acsConnectionString string = listKeys(acsResource.id, '2023-04-01').primaryConnectionString
output emailEndpoint string = acsResource.properties.hostName
```

---

## Quick Reference — az communication CLI

```bash
# Resource management
az communication create / show / delete / list

# Phone numbers
az communication phone-number list / show / delete
az communication phone-number search-plan-create / purchase

# Keys
az communication list-key / regenerate-key

# Email domains
az communication email domain create / show / list / delete
az communication email domain initiate-verification / cancel-verification
az communication email sender-username create / list / delete

# SMS opt-out (toll-free)
az communication sms optout add / remove / show

# Direct routing
az communication direct-routing gateway create / delete / list
```
