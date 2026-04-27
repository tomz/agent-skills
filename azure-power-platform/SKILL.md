---
name: azure-power-platform
description: Microsoft Power Platform — Power Apps, Power Automate, Power Pages, Dataverse, environments, solutions, ALM, Copilot Studio
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Microsoft Power Platform — Comprehensive Reference

## Table of Contents
1. [Power Apps](#power-apps)
2. [Power Automate](#power-automate)
3. [Dataverse](#dataverse)
4. [Power Pages](#power-pages)
5. [Copilot Studio](#copilot-studio)
6. [Environments](#environments)
7. [Solutions and ALM](#solutions-and-alm)
8. [pac CLI Reference](#pac-cli-reference)
9. [Admin Center, DLP, and Governance](#admin-center-dlp-and-governance)
10. [Licensing](#licensing)
11. [Azure Integration](#azure-integration)
12. [Center of Excellence Toolkit](#center-of-excellence-toolkit)

---

## Power Apps

### App Types

| Type | Use Case | Data Source |
|------|----------|-------------|
| **Canvas App** | Pixel-perfect UI, mobile-first, any connector | Any (SharePoint, Dataverse, SQL, Excel…) |
| **Model-Driven App** | Data-first, auto-generated UI from Dataverse schema | Dataverse only |
| **Card** | Micro-UI embedded in Teams, email, dashboards | Any connector |
| **Custom Page** | Modern pages embedded in model-driven apps | Dataverse |

### Canvas App — Power Fx Formulas

Power Fx is the open-source, Excel-like formula language used across the platform.

#### Navigation and Screens
```powerfx
// Navigate to a screen with a transition
Navigate(ScreenName, ScreenTransition.Fade)

// Navigate and pass context
Navigate(DetailScreen, ScreenTransition.Cover, {RecordID: Gallery1.Selected.ID})

// Back navigation
Back()

// Navigate inside a component — use OnSelect of a button
Button1.OnSelect = Navigate(HomeScreen, ScreenTransition.None)
```

#### Variables
```powerfx
// Global variable (persists across screens)
Set(gblUserName, User().FullName)

// Context variable (scoped to current screen)
UpdateContext({locIsLoading: true, locSelectedItem: Gallery1.Selected})

// Collection (in-memory table)
ClearCollect(colContacts,
    Filter(Contacts, Status = "Active")
)

// Add a record to a collection
Collect(colCart, {Product: "Widget", Qty: TextInput1.Text * 1, Price: 9.99})

// Remove from collection
Remove(colCart, ThisItem)
RemoveIf(colCart, Qty = 0)
```

#### Galleries and Forms
```powerfx
// Gallery Items with filter and sort
Gallery1.Items = SortByColumns(
    Filter(
        'Work Orders',
        StartsWith(Title, SearchBox.Text),
        Status <> "Closed"
    ),
    "Created On", Descending
)

// Gallery selected item detail
Label1.Text = Gallery1.Selected.Title
Label2.Text = Text(Gallery1.Selected.'Due Date', "dd/mm/yyyy")

// Form — submit a new record
SubmitForm(Form1)
NewForm(Form1)
EditForm(Form1)
ResetForm(Form1)

// Patch — write without a form
Patch(Accounts,
    Defaults(Accounts),                       // new record
    {Name: TextInput_Name.Text,
     'Account Number': TextInput_Num.Text,
     Revenue: Value(TextInput_Rev.Text)}
)

// Patch — update existing record
Patch(Tasks, Gallery1.Selected,
    {Status: "Complete", 'Completed Date': Now()}
)

// Remove a record
Remove(Projects, Gallery1.Selected);
Notify("Project deleted", NotificationType.Success)
```

#### Delegation
Delegation allows the data source to process filter/sort server-side (critical for large datasets).

```powerfx
// DELEGABLE — Dataverse supports these operators
Filter(Accounts, Revenue > 1000000)          // numeric comparison ✓
Filter(Contacts, City = "London")            // equality ✓
Search(Products, SearchInput.Text, "Name")   // Search() on text columns ✓
SortByColumns(Accounts, "Name")              // sort ✓

// NON-DELEGABLE — processed client-side (max 500/2000 rows)
Filter(Orders, Len(Notes) > 100)             // Len() not delegable ✗
Filter(Contacts, Upper(City) = "LONDON")     // Upper() not delegable ✗

// Workaround: increase data row limit (max 2000) in app settings
// Or pre-filter with a delegable predicate, then apply non-delegable in-memory
ClearCollect(colFiltered,
    Filter(Contacts, City = "London")        // delegable first pass
);
// Then filter colFiltered locally
```

#### Connectors and Data Sources
```powerfx
// SharePoint — read a list
ClearCollect(colSPItems, 'SharePoint List Name')

// Office 365 Users — get current user's profile
Set(gblProfile, Office365Users.MyProfileV2())
Label1.Text = gblProfile.displayName

// Send an email via Office 365 Outlook connector
Office365Outlook.SendEmailV2(
    "recipient@company.com",
    "Subject Here",
    "Body text here",
    {Importance: "High"}
)

// HTTP connector — call a custom API
Set(gblResponse,
    ParseJSON(
        Office365Outlook.HttpRequestV2(
            "https://api.example.com/data",
            "GET",
            {},
            {},
            ""
        ).body
    )
)
```

#### Component Libraries
```powerfx
// Components expose custom properties
// In the component definition, add a custom Input property:
//   Property: CustomerName (Text, Input)
//
// In the host app, reference the component:
ComponentInstance.CustomerName = "Acme Corp"

// Components can fire custom output events
// Define an Output property (e.g., OnSave) and trigger in component:
Self.OnSave()
```

---

## Power Automate

### Flow Types

| Type | Trigger | Use Case |
|------|---------|----------|
| **Automated** | Event-based (new row, email, webhook) | React to events automatically |
| **Instant** | Manual (button, Power Apps, Teams) | On-demand actions |
| **Scheduled** | Recurrence (daily, hourly, cron) | Batch processing, reports |
| **Desktop Flow** | Called from cloud flow or manual | RPA — automate Windows/web UI |
| **Business Process Flow** | Dataverse record lifecycle | Guided stage-based processes |

### Common Triggers and Actions

#### Dataverse Triggers
```
Trigger: "When a row is added, modified or deleted"
  - Table: Accounts
  - Change type: Added or Modified
  - Scope: Organization

Trigger: "When an action is performed" (custom Dataverse action)
```

#### Expressions (Power Automate uses Azure Logic Apps expression language)
```javascript
// String functions
concat('Hello, ', triggerBody()?['name'])
toUpper(outputs('Get_item')?['body/Title'])
substring(triggerBody()?['email'], 0, indexOf(triggerBody()?['email'], '@'))
replace(variables('myString'), ' ', '_')
split(body('Get_response_details')?['r3b77'], ';')

// Date/time
utcNow()
addDays(utcNow(), 7)
addDays(utcNow(), -30)
formatDateTime(utcNow(), 'yyyy-MM-dd')
formatDateTime(triggerBody()?['DueDate'], 'dd/MM/yyyy HH:mm')
convertTimeZone(utcNow(), 'UTC', 'GMT Standard Time')

// Math
add(variables('counter'), 1)
mul(triggerBody()?['Quantity'], triggerBody()?['UnitPrice'])
div(outputs('total'), 100)
mod(variables('index'), 2)

// Logic
if(equals(triggerBody()?['Status'], 'Approved'), 'Send approval email', 'Skip')
and(greater(variables('score'), 80), equals(variables('passed'), true))
empty(triggerBody()?['Description'])
coalesce(triggerBody()?['Phone'], triggerBody()?['Mobile'], 'No phone')

// Array
length(body('List_rows')?['value'])
first(body('List_rows')?['value'])
last(body('List_rows')?['value'])
join(variables('myArray'), ', ')
union(variables('arr1'), variables('arr2'))
contains(variables('roles'), 'Admin')

// JSON / object
json('{"key": "value"}')
string(body('Parse_JSON'))
triggerBody()?['_ownerid_value']           // Dataverse lookup GUID
body('Get_a_row_by_ID')?['statuscode@OData.Community.Display.V1.FormattedValue']
```

#### Approval Flows
```yaml
Action: "Start and wait for an approval"
  Approval type: Approve/Reject - Everyone must approve
  Title: "Expense claim: @{triggerBody()?['Amount']}"
  Assigned to: manager@company.com; finance@company.com
  Details: "Submitted by: @{triggerBody()?['SubmittedBy']}"
  Item link: https://apps.powerapps.com/play/...
  Item link description: View expense claim

Condition: outputs('Start_and_wait_for_an_approval')?['body/outcome'] is equal to 'Approve'
  Yes branch: Send approval email + update Dataverse row
  No branch: Send rejection email + update status to Rejected
```

#### Adaptive Cards in Teams
```json
{
  "type": "AdaptiveCard",
  "version": "1.4",
  "body": [
    {"type": "TextBlock", "text": "New Leave Request", "weight": "Bolder", "size": "Medium"},
    {"type": "FactSet", "facts": [
      {"title": "Employee:", "value": "@{triggerBody()?['EmployeeName']}"},
      {"title": "Dates:", "value": "@{triggerBody()?['StartDate']} to @{triggerBody()?['EndDate']}"},
      {"title": "Days:", "value": "@{triggerBody()?['Days']}"}
    ]}
  ],
  "actions": [
    {"type": "Action.Submit", "title": "Approve", "data": {"action": "approve"}},
    {"type": "Action.Submit", "title": "Reject",  "data": {"action": "reject"}}
  ]
}
```

#### Desktop Flows (RPA)
Desktop flows use Power Automate Desktop (PAD) to automate UI interactions.
```
// Common PAD actions (written in PAD scripting notation):
Launch application: notepad.exe
Attach to running application: "Chrome" with title "SAP"
Click UI element: button with automationid="btnSubmit"
Populate text field: input[name="username"] with value: CurrentUser
Extract data from web page table → DataTable variable
Loop For Each Row In DataTable:
    Set variable TotalAmount to TotalAmount + CurrentRow['Amount']
Write to Excel worksheet: TotalAmount at row 1, column 3
Close application: gracefully
```

---

## Dataverse

### Tables, Columns, and Relationships

#### Table Types
| Type | Description |
|------|-------------|
| **Standard** | Full feature set — forms, views, charts, BPF |
| **Activity** | Special type for emails, tasks, appointments |
| **Elastic** | Cosmos DB-backed, for high-volume append-heavy data |
| **Virtual** | Data lives in external source, surfaced in Dataverse |

#### Column Types
```
Text (single line, multi-line, rich text, email, URL, phone, ticker)
Number (whole, decimal, float, currency)
Date and Time (date only, date & time)
Choice / Choices (local or global option set)
Lookup (many-to-one relationship)
File / Image
Yes/No (boolean)
Autonumber
Formula (Power Fx — calculated at read time, not stored)
Rollup (aggregated from related rows)
```

#### Relationships
```
Many-to-One (Lookup): Contact → Account
One-to-Many (1:N): Account → Contacts
Many-to-Many (N:N): Contact ↔ Marketing List
  - Intersect table auto-created
  - Or use a custom intersect table for additional columns

Relationship behaviors (cascade):
  Parental: Delete parent → delete children
  Referential: Delete parent → restrict if children exist
  Referential (restrict delete): most common for important data
  Custom: set each cascade type individually
```

#### Business Rules
Applied server-side (and optionally client-side in forms):
```
Condition: Status equals "Approved" AND Amount > 10000
Action: Set field "Requires Finance Review" = true
        Show error message: "Finance approval required for amounts over $10,000"
        Set field "Approver" as required
        Lock field "Amount"
```

#### Calculated and Rollup Fields
```
// Calculated (Formula column — Power Fx):
Text(ThisRecord.'Start Date', "MMM yyyy") & " — " & ThisRecord.Title

// Rollup:
Source: Account → Contacts (1:N relationship)
Aggregate: COUNT of Contact rows WHERE Status = Active
Result stored in: Account.'Active Contact Count'
Recalculated: every 12 hours (or on-demand via workflow)
```

#### Security Roles and Business Units
```
Privilege levels (CRUD + Append + Share + Assign):
  None → User → Business Unit → Parent: Child Business Units → Organization

Security role assignment:
  User → can have multiple roles
  Team → role inherited by team members (recommended pattern)

Business units:
  Root BU (org-level) → child BUs → grandchild BUs
  Records owned by users/teams scoped to their BU

Row-level security: use Column Security Profiles for sensitive columns
```

#### Environment Variables
```
// Types: text, number, boolean, JSON, data source, secret (Key Vault linked)
// Used in flows and apps instead of hard-coded values

// In Power Automate — reference via:
outputs('Get_environment_variable')['value']

// In canvas app:
LookUp('Environment Variable Values', 'Environment Variable Definition'.'Schema Name' = "prefix_VariableName", Value)

// pac CLI — set environment variable value:
pac env set --environment <env-id> --name "prefix_VariableName" --value "production-url"
```

---

## Power Pages

### Site Architecture
```
Power Pages site
├── Web Roles          (Anonymous, Authenticated, Custom roles)
├── Table Permissions  (per table, per web role — CRUD control)
├── Page Templates     (Liquid + Bootstrap 5)
├── Web Files          (CSS, JS, images)
├── Site Settings      (key/value config pairs)
└── Web APIs           (REST endpoints for Dataverse from browser JS)
```

### Table Permissions
```
Access type:
  Global          — all rows of that table
  Contact         — rows related to the logged-in contact
  Account         — rows related to the contact's parent account
  Self            — only the contact's own row (Contact table)
  Parent          — inherits from parent table permission

Privileges: Read, Write, Create, Delete, Append, Append To
Scope chain: WebRole → TablePermission → (optional) Parent TablePermission
```

### Liquid Templates
```liquid
{% comment %} Display logged-in user info {% endcomment %}
{% if user %}
  <p>Welcome, {{ user.fullname }}</p>
  {% assign myContacts = entities['contact'] | where: 'ownerid', user.id %}
  {% for c in myContacts.items %}
    <li>{{ c.fullname }} — {{ c.emailaddress1 }}</li>
  {% endfor %}
{% else %}
  <p><a href="/signin">Please sign in</a></p>
{% endif %}

{% comment %} Fetch with fetchXML {% endcomment %}
{% fetchxml accountQuery %}
<fetch top="10">
  <entity name="account">
    <attribute name="name" />
    <attribute name="revenue" />
    <filter>
      <condition attribute="statecode" operator="eq" value="0" />
    </filter>
    <order attribute="revenue" descending="true" />
  </entity>
</fetch>
{% endfetchxml %}

{% for account in accountQuery.results.entities %}
  <tr><td>{{ account.name }}</td><td>{{ account.revenue }}</td></tr>
{% endfor %}
```

### Web API (client-side JavaScript)
```javascript
// Read records (requires Table Permission with Read privilege)
fetch('/_api/accounts?$select=name,revenue&$top=10&$orderby=revenue desc', {
  headers: { '__RequestVerificationToken': tokenValue }
})
.then(r => r.json()).then(data => console.log(data.value));

// Create a record
fetch('/_api/contacts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    '__RequestVerificationToken': tokenValue
  },
  body: JSON.stringify({ firstname: 'Jane', lastname: 'Doe', emailaddress1: 'jane@example.com' })
});

// Update a record
fetch(`/_api/contacts(${contactId})`, {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/json', '__RequestVerificationToken': tokenValue },
  body: JSON.stringify({ jobtitle: 'Manager' })
});
```

---

## Copilot Studio

### Core Concepts
```
Topics        — conversation branches (System topics + Custom topics)
Entities      — slot-filling: extract structured data from user input
Actions       — connector actions, Power Automate flows, HTTP requests, AI plugins
Generative AI — generative answers from knowledge sources (websites, files, Dataverse)
Variables     — topic-scoped (Topic.), global (Global.), system (System.)
```

### Topic Authoring
```yaml
Trigger phrases:
  - "I want to reset my password"
  - "password reset"
  - "can't log in"

Nodes:
  Message: "I can help you reset your password. What's your employee ID?"
  Question: Store response in Topic.EmployeeID (Entity: Number)
  Condition:
    if Topic.EmployeeID > 0:
      Call action: Reset Password Flow (passes Topic.EmployeeID)
      Message: "Done! Check your email for a temporary password."
    else:
      Message: "I didn't catch that. Please enter a valid employee ID."
      Redirect to: current topic (retry)
  End conversation
```

### Generative AI Configuration
```
Knowledge sources:
  - Public website: https://support.company.com
  - SharePoint site: https://company.sharepoint.com/sites/HR
  - Uploaded documents: PDF, Word, PowerPoint
  - Dataverse table (via Dataverse search)

Generative answers node:
  Input: System.Activity.Text   (user's message)
  Search type: Semantic
  Fallback: "I couldn't find an answer. Want me to connect you to support?"

Orchestration mode (GPT-4 driven):
  Topics triggered by intent, not just keyword matching
  Actions called dynamically based on user intent
```

### System Variables
```
System.Activity.Text          — user's raw message
System.User.DisplayName       — authenticated user's name
System.User.Id                — AAD object ID
System.Conversation.Id        — conversation ID
System.Bot.Name               — bot display name
System.LastMessage.Text       — previous user message
```

---

## Environments

### Environment Types

| Type | Purpose | Dataverse | Limits |
|------|---------|-----------|--------|
| **Default** | Org-wide default, all licensed users | Included | 1 per tenant |
| **Developer** | Personal sandbox for makers | Included | 1 per user |
| **Sandbox** | Testing and UAT | Optional | Based on license |
| **Production** | Live apps | Optional | Based on license |
| **Trial** | 30-day evaluation | Included | 1 per user |
| **Managed** | Governed with guardrails | Optional | Add-on feature |

### Managed Environments Features
```
- Weekly digest emails to admins
- Solution checker enforcement (block on critical rules)
- Sharing limits (e.g., max 20 users per canvas app)
- IP firewall (allowlist inbound IPs)
- Customer-managed keys (CMK)
- Lockbox (approve/deny Microsoft support access)
- DLP policy inheritance from parent group
```

---

## Solutions and ALM

### Solution Types
```
Unmanaged solution:
  - Development environment
  - All components editable
  - Can export as managed or unmanaged
  - Contains the "source of truth" — store in source control

Managed solution:
  - Installed in test/production
  - Components locked (cannot edit in target environment)
  - Can be upgraded, patched, or removed as a unit
  - Layering: multiple managed solutions can coexist
```

### Solution Layering
```
Layer order (top wins):
  1. Unmanaged customizations (in target — avoid in prod)
  2. Managed solution — most recently installed publisher wins
  3. System (Microsoft base layer)

Best practice:
  Dev environment → unmanaged work
  Export as managed → import to Test → import to Prod
  Never make unmanaged changes in production
```

### Pipelines (in-product ALM)
```
Setup:
  1. Create a Pipeline record in the host environment
  2. Add stages: Dev → Test → Production (linked environments)
  3. Assign a service principal as the deployment user

Deploy:
  In dev environment → Solutions → Pipelines → "Deploy here"
  Creates a managed copy, runs solution checker, deploys to next stage
  Supports pre/post deployment steps (Power Automate flows)
```

### Export/Import via pac CLI
```bash
# Export unmanaged solution from dev
pac solution export \
  --name MySolution \
  --path ./solutions/MySolution.zip \
  --managed false \
  --environment <dev-env-id>

# Unpack for source control
pac solution unpack \
  --zipfile ./solutions/MySolution.zip \
  --folder ./solutions/MySolution \
  --processCanvasApps

# Pack before deployment
pac solution pack \
  --zipfile ./solutions/MySolution_managed.zip \
  --folder ./solutions/MySolution \
  --managed true \
  --processCanvasApps

# Import managed to target
pac solution import \
  --path ./solutions/MySolution_managed.zip \
  --environment <prod-env-id> \
  --activate-plugins \
  --force-overwrite
```

---

## pac CLI Reference

### Installation
```bash
# Install via .NET tool
dotnet tool install --global Microsoft.PowerApps.CLI.Tool

# Or download installer from:
# https://aka.ms/PowerAppsCLI

pac --version
```

### Authentication
```bash
# Interactive login (browser popup)
pac auth create --name MyOrg --url https://myorg.crm.dynamics.com

# Service principal (for CI/CD)
pac auth create \
  --name ProdSP \
  --url https://prod.crm.dynamics.com \
  --applicationId <app-id> \
  --clientSecret <secret> \
  --tenant <tenant-id>

# List profiles
pac auth list

# Switch active profile
pac auth select --index 2

# Delete a profile
pac auth delete --index 1
```

### Environment Commands
```bash
# List all environments
pac env list

# List environments with details
pac env list --filter production

# Show current environment
pac env who

# Create a new developer environment
pac env create \
  --name "My Dev Env" \
  --type Developer \
  --region unitedstates

# Copy environment
pac env copy \
  --source-env <source-id> \
  --name "Test Copy" \
  --type Sandbox
```

### Solution Commands
```bash
# List solutions in current environment
pac solution list

# Check solution (runs solution checker)
pac solution check \
  --path ./MySolution.zip \
  --geo unitedstates

# Clone solution (creates local project)
pac solution clone \
  --name MySolution \
  --outputDirectory ./solutions

# Upgrade managed solution
pac solution upgrade \
  --solution-name MySolution

# Delete solution
pac solution delete --solution-name MySolution
```

### Canvas App Commands
```bash
# Download canvas app as .msapp
pac canvas pack \
  --sources ./src/MyApp \
  --msapp ./dist/MyApp.msapp

pac canvas unpack \
  --msapp ./dist/MyApp.msapp \
  --sources ./src/MyApp
```

### Power Automate / Flow Commands
```bash
# List cloud flows
pac flow list

# Export a flow
pac flow export \
  --id <flow-guid> \
  --environment <env-id> \
  --packageType zip \
  --outputDirectory ./flows

# Run a desktop flow
pac flow run \
  --flow-id <desktop-flow-guid> \
  --environment <env-id>
```

### Dataverse Commands
```bash
# Publish all customizations
pac org publish

# Export data (configuration migration)
pac data export \
  --schemaFile schema.xml \
  --dataFile export.zip \
  --environment <env-id>

pac data import \
  --data export.zip \
  --environment <target-env-id>

# Generate Power Fx model (early access)
pac powerfx run --file script.fx
```

### Copilot Studio Commands
```bash
# List bots
pac copilot list

# Export bot
pac copilot export \
  --bot-id <bot-guid> \
  --environment <env-id> \
  --outputDirectory ./bots

# Import bot
pac copilot import \
  --bot-zip ./bots/MyBot.zip \
  --environment <target-env-id>
```

---

## Admin Center, DLP, and Governance

### Data Loss Prevention (DLP) Policies
```
DLP policies classify connectors into groups:
  Business (can share data with each other)
  Non-Business (can share data with each other)
  Blocked (cannot be used at all)

Rules:
  - Business ↔ Non-Business connectors cannot be combined in one flow/app
  - Blocked connectors cannot be used at all

Scope:
  Tenant-level policy    — applies to all environments
  Environment-level      — applies to specific environments
  Exclude environments   — exclude certain envs from a policy

Common patterns:
  Policy 1 (tenant): Block all social media + file sharing connectors
  Policy 2 (prod): Business = Dataverse, SharePoint, Outlook; Non-Business = everything else
  Policy 3 (sandbox): Relaxed — more connectors in Business group for testing

// PowerShell — create DLP policy
Add-PowerAppsPolicyConnectorConfigurations `
  -PolicyName "Production DLP" `
  -ConnectorId "/providers/Microsoft.PowerApps/apis/shared_sharepointonline" `
  -GroupName "BusinessData"
```

### Tenant Isolation
```
Settings → Power Platform → Tenant isolation:
  Inbound:  block flows in OTHER tenants from connecting to YOUR Dataverse
  Outbound: block flows in YOUR tenant from connecting to OTHER tenants' resources

Allowlist: specific tenant IDs exempt from isolation rules
Recommended: enable both for regulated industries
```

### Admin Center Key Areas
```
Environments    — create, manage, backup, restore, copy environments
Capacity        — Dataverse storage (database + file + log), add-on capacity
Analytics       — usage reports, service health, adoption metrics
Policies        — DLP, tenant isolation, connector classification
Billing         — pay-as-you-go Azure subscriptions, license assignment
Settings        — trial policy, environment creation policy, Teams integration
```

---

## Licensing

### Per-User Plans
| License | Included |
|---------|----------|
| **Power Apps Premium** | Unlimited apps + Dataverse; $20/user/month |
| **Power Automate Premium** | Cloud flows + attended RPA; $15/user/month |
| **Power Automate Process** | Unattended RPA (bot); $150/bot/month |
| **Copilot Studio** | 25,000 messages/month; $200/tenant/month |
| **Power Pages** | 100 authenticated users/month or 500K anonymous sessions |

### Per-App Plan
```
$5/user/app/month
Allows a user to run ONE specific app + associated flows
No Dataverse access for other apps
Good for: deploying a single app to many non-maker users
```

### Pay-As-You-Go
```
Linked to an Azure subscription
No per-user licensing needed — billed per unique app user/month
Rates: $10/active user/app/month for canvas/model-driven
Dataverse storage billed at Azure rates
Ideal for: variable usage, external users, apps with unpredictable adoption
```

### Microsoft 365 / Dynamics 365 Seeded Rights
```
M365 E3/E5 users:
  - Canvas apps using standard connectors (SharePoint, Teams, Outlook, etc.)
  - Limited cloud flows (standard connectors only)
  - No Dataverse (except limited Tables in Teams)

Dynamics 365 (e.g., Sales Enterprise):
  - Power Apps use rights for the Dynamics app
  - Power Automate flows for that Dynamics app
  - Cannot build unrelated apps on the same license
```

---

## Azure Integration

### Custom Connectors
```yaml
# OpenAPI 2.0 / Swagger definition
swagger: "2.0"
info:
  title: My Company API
  version: "1.0"
host: api.company.com
basePath: /v1
schemes: [https]
securityDefinitions:
  apiKey:
    type: apiKey
    in: header
    name: x-api-key
paths:
  /customers/{id}:
    get:
      operationId: GetCustomer
      summary: Get customer by ID
      parameters:
        - name: id
          in: path
          required: true
          type: string
      responses:
        200:
          description: Customer object
          schema:
            $ref: '#/definitions/Customer'

# Import via pac CLI:
pac connector create --api-definition-file ./swagger.json --environment <env-id>
```

### Azure Functions Integration
```csharp
// Azure Function exposed as custom connector endpoint
[FunctionName("CalculateRisk")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    ILogger log)
{
    var body = await new StreamReader(req.Body).ReadToEndAsync();
    var input = JsonSerializer.Deserialize<RiskInput>(body);
    var score = RiskEngine.Calculate(input);
    return new OkObjectResult(new { RiskScore = score, Category = score > 75 ? "High" : "Low" });
}

// In Power Automate: call via custom connector or HTTP action
// URL: https://<funcapp>.azurewebsites.net/api/CalculateRisk?code=<key>
```

### Logic Apps vs Power Automate
```
Logic Apps:
  - Azure resource (IaC via ARM/Bicep, source-controlled JSON)
  - Consumption (serverless) or Standard (dedicated)
  - Better for complex enterprise integration, long-running workflows
  - Direct access to all Azure connectors + on-prem gateway

Power Automate:
  - Power Platform service (managed by Microsoft)
  - Maker-friendly UI — no Azure knowledge needed
  - Better for business users, M365 integration, approvals
  - Shares connector ecosystem with Logic Apps

Bridge: Power Automate can call Logic Apps via HTTP action (and vice versa)
```

### On-Premises Data Gateway
```
Connects cloud flows/apps to:
  SQL Server, Oracle, SharePoint on-prem, file shares, SAP

Architecture:
  Gateway client (on-prem server) ← polling → Azure Relay ← → Power Platform

Install:
  Download from https://aka.ms/opdg
  Register with Power Platform via admin account
  Cluster mode: install on multiple machines for HA

Reference in connector: select "Connect via on-premises data gateway"
```

---

## Center of Excellence Toolkit

### Installation
```bash
# Download CoE Starter Kit from:
# https://aka.ms/CoEStarterKitDownload

# Import core components (managed solution) into your CoE environment
pac solution import --path CenterOfExcellenceCoreComponents.zip

# After import, run the "Admin | Welcome Email" flow setup
# Configure environment variables:
#   Admin eMail            admin@company.com
#   Power Platform Maker   maker@company.com
#   Also Notify Co-Admin   coadmin@company.com
```

### Key CoE Components
```
Core:
  Admin | Sync flows     — inventories all apps, flows, connectors, makers
  Power Platform Admin View — model-driven app with full tenant inventory
  Maker Overview App     — canvas app for makers to see their own resources

Governance:
  DLP Editor App         — UI to manage DLP policies without PowerShell
  Admin | Archive flows  — notify and archive unused apps/flows
  Compliance detail request — ask makers to document their apps

Nurture:
  Maker Assessment       — onboarding quiz and training path
  Template Library       — curated starter apps and flows for makers
  Innovation Backlog     — idea submission + voting canvas app

Audit & Report:
  Power BI dashboard     — usage trends, maker adoption, risk assessment
  Telemetry              — Azure Application Insights integration
```

### Recommended Governance Patterns
```
1. Environment strategy:
   - Default env: block app creation (limit to admins/approved makers)
   - Dedicated maker sandbox envs (1 per team or power user)
   - Shared dev → test → prod pipeline per business unit

2. DLP layering:
   - Tenant-wide baseline: block high-risk connectors
   - Environment-specific: loosen for approved use cases

3. App approval workflow:
   - Makers submit via CoE Governance app
   - Admin reviews, provisions production environment
   - App promoted via pipeline with managed solutions

4. Naming conventions:
   Apps:     [Team]_[AppName]_[Version]  e.g. HR_LeaveRequest_v2
   Flows:    [Trigger]_[Action]          e.g. NewAccount_SendWelcomeEmail
   Envs:     [OrgUnit]-[Purpose]         e.g. Finance-Production
   Solutions: [Publisher]_[Product]      e.g. Contoso_HRSuite

5. Monitoring:
   - Weekly admin digest (Managed Environments or CoE flows)
   - Alert on: new connectors, high-risk sharing, storage thresholds
   - Monthly review of CoE Power BI dashboard
```

---

## Quick Reference Cheat Sheet

### Power Fx Key Functions
```powerfx
If(condition, trueResult, falseResult)
Switch(value, match1, result1, match2, result2, defaultResult)
IsBlank(value) / IsEmpty(table)
Coalesce(v1, v2, v3)                    // first non-blank
Distinct(table, column)                 // unique values
AddColumns(table, "NewCol", formula)    // virtual columns
ShowColumns(table, "Col1", "Col2")
DropColumns(table, "Col1")
RenameColumns(table, "Old", "New")
Ungroup(table, "nestedCol")
ForAll(table, formula)                  // iterate, returns table
With({x: 1, y: 2}, x + y)             // scoped variables
```

### OData Filter Syntax (Dataverse REST / Flow)
```
$filter=statecode eq 0 and revenue gt 1000000
$filter=contains(name, 'Contoso')
$filter=startswith(emailaddress1, 'admin')
$filter=createdon ge 2024-01-01T00:00:00Z
$filter=_ownerid_value eq <user-guid>
$filter=statuscode in (1,2,3)
$orderby=createdon desc
$top=50
$select=name,revenue,emailaddress1
$expand=contact_customer_accounts($select=fullname,emailaddress1)
```

### pac CLI Quick Reference
```bash
pac auth create --url <env-url>          # login
pac env list                             # list environments
pac solution list                        # list solutions
pac solution export --name X --path X.zip
pac solution import --path X.zip
pac solution unpack --zipfile X.zip --folder ./src
pac solution pack   --zipfile X.zip --folder ./src --managed true
pac org publish                          # publish customizations
pac env who                              # show current environment
pac telemetry disable                    # opt out of telemetry
```
