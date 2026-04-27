---
name: azure-powerbi
description: Azure Power BI — datasets, reports, dashboards, embedded analytics, Power BI Service, Gateway, dataflows, DAX, Power Query M
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Power BI — Comprehensive Reference

## 1. Product Landscape

### Power BI Service (app.powerbi.com)
- Cloud-hosted SaaS platform on Microsoft Fabric
- Publish, share, and collaborate on reports and dashboards
- Schedules refresh, alerts, subscriptions, workspaces
- REST API surface for automation

### Power BI Desktop
- Free Windows app for report authoring
- Full DAX editor, Power Query M editor, model view
- Publish `.pbix` files to the Service
- Does NOT require a Pro licence to build — only to publish/share

### Power BI Embedded (Azure resource)
- Embed reports in your own apps without users needing PBI licences
- Capacity-based (A-SKUs in Azure, or P/F SKUs via Fabric)
- Uses embed tokens (service principal) instead of user identity

### Power BI Report Server (on-premises)
- Self-hosted, behind firewall
- Subset of cloud features — no dataflows, limited AI visuals
- Separate `.pbix` build target (Desktop for Report Server)

### Microsoft Fabric
- Unified analytics platform that includes Power BI
- Fabric items: Lakehouses, Warehouses, Notebooks, Pipelines, Eventstreams
- Power BI is the reporting layer of Fabric

---

## 2. Workspaces, Apps, and Deployment Pipelines

### Workspaces
- Collaboration containers; every item lives in exactly one workspace
- Types: My Workspace (personal), shared workspace
- Roles: Admin, Member, Contributor, Viewer
- Premium/Fabric capacity can be assigned per workspace

```powershell
# List workspaces via PowerShell (MicrosoftPowerBIMgmt)
Connect-PowerBIServiceAccount
Get-PowerBIWorkspace -All | Select-Object Id, Name, IsOnDedicatedCapacity
```

```bash
# List workspaces via REST API
TOKEN=$(az account get-access-token --resource https://analysis.windows.net/powerbi/api --query accessToken -o tsv)
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.powerbi.com/v1.0/myorg/groups" | python3 -m json.tool
```

### Apps
- Published view of a workspace for end-users
- App consumers don't need workspace access
- Audience-based permissions (different sections per audience)

```powershell
# Publish app (updates existing if present)
New-PowerBIApp -WorkspaceId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
               -Name "Sales Analytics" -Description "Monthly sales app"
```

### Deployment Pipelines (Premium/Fabric)
- Dev → Test → Prod promotion without manual republish
- Deployment rules override data source connection strings per stage

```powershell
# Trigger a pipeline deployment stage via REST
$body = @{ "sourceStageOrder" = 0 } | ConvertTo-Json
Invoke-PowerBIRestMethod -Url "pipelines/{pipelineId}/deploy" `
                         -Method Post -Body $body
```

---

## 3. Datasets, Dataflows, and Datamarts

### Datasets (Semantic Models)
- The data model behind reports: tables, relationships, DAX measures
- Import mode: data copied into in-memory engine (VertiPaq)
- DirectQuery: queries pass through to source at render time
- Composite: mix Import + DirectQuery tables in one model

```bash
# Trigger dataset refresh via REST API
DATASET_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
GROUP_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/datasets/$DATASET_ID/refreshes"
```

```powershell
# Get refresh history
Get-PowerBIDatasetRefreshHistory -DatasetId $DATASET_ID -WorkspaceId $GROUP_ID -Top 5
```

### Dataflows (Gen1 / Gen2)
- Self-service ETL using Power Query Online
- Output: Azure Data Lake Storage Gen2 (CDM folders)
- Gen2 (Fabric): outputs to Lakehouse tables, no separate storage needed
- Reusable across datasets — define once, reference many times

### Datamarts (Preview/GA varies by region)
- Auto-generated Azure SQL database behind a dataflow
- Built-in dataset + dataflow + relational store in one package
- Good for analysts who want SQL access without IT involvement

---

## 4. DAX Patterns

### CALCULATE — the most important function
```dax
-- Basic filter override
Sales YTD =
CALCULATE(
    [Total Sales],
    DATESYTD('Date'[Date])
)

-- Remove a filter with ALL
Market Share =
DIVIDE(
    [Total Sales],
    CALCULATE([Total Sales], ALL('Product'))
)

-- Multiple filter arguments (AND logic)
Sales East Q1 =
CALCULATE(
    [Total Sales],
    'Region'[Area] = "East",
    'Date'[Quarter] = "Q1"
)
```

### FILTER for row-by-row evaluation
```dax
-- Avoid FILTER on large tables when possible; prefer CALCULATE with direct column filters
High Value Customers =
CALCULATE(
    [Total Sales],
    FILTER('Customer', 'Customer'[LifetimeValue] > 10000)
)
```

### Time Intelligence
```dax
Sales LY      = CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date]))
Sales MoM %   = DIVIDE([Total Sales] - [Sales LY], [Sales LY])
Sales QTD     = CALCULATE([Total Sales], DATESQTD('Date'[Date]))
Sales Rolling12 = CALCULATE([Total Sales], DATESINPERIOD('Date'[Date], LASTDATE('Date'[Date]), -12, MONTH))
```

### Variables for readability and performance
```dax
Profit Margin % =
VAR TotalRevenue = [Total Sales]
VAR TotalCost    = [Total Cost]
VAR Profit       = TotalRevenue - TotalCost
RETURN
    DIVIDE(Profit, TotalRevenue)
```

### Iterator functions (X-functions)
```dax
-- SUMX evaluates expression row-by-row then sums
Revenue = SUMX('Sales', 'Sales'[Qty] * RELATED('Product'[UnitPrice]))

-- RANKX for rankings
Product Rank =
RANKX(
    ALL('Product'[ProductName]),
    [Total Sales],
    ,
    DESC,
    Dense
)
```

### Context Transition
```dax
-- Row context → filter context using CALCULATE inside iterators
Avg Sales per Customer =
AVERAGEX(
    VALUES('Customer'[CustomerID]),
    CALCULATE([Total Sales])   -- CALCULATE transitions row → filter context
)
```

---

## 5. Power Query M Patterns

### Basic table transformations
```m
// Load and filter a CSV
let
    Source     = Csv.Document(File.Contents("C:\data\sales.csv"), [Delimiter=",", Encoding=1252]),
    Promoted   = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    Typed      = Table.TransformColumnTypes(Promoted, {{"Date", type date}, {"Amount", type number}}),
    Filtered   = Table.SelectRows(Typed, each [Amount] > 0),
    Renamed    = Table.RenameColumns(Filtered, {{"Amount", "SalesAmount"}})
in
    Renamed
```

### Custom function (reusable ETL)
```m
// Define a function to fetch paginated API data
let
    FetchPage = (pageNum as number) as table =>
        let
            Url      = "https://api.example.com/orders?page=" & Number.ToText(pageNum),
            Response = Json.Document(Web.Contents(Url)),
            Data     = Table.FromList(Response[data], Splitter.SplitByNothing()),
            Expanded = Table.ExpandRecordColumn(Data, "Column1", {"id","amount","date"})
        in
            Expanded,

    Pages    = List.Transform({1..10}, each FetchPage(_)),
    Combined = Table.Combine(Pages)
in
    Combined
```

### Query folding — keep transformations foldable
```m
// Good: filter early so SQL source can fold the WHERE clause
let
    Source   = Sql.Database("server", "db"),
    Orders   = Source{[Schema="dbo", Item="Orders"]}[Data],
    Filtered = Table.SelectRows(Orders, each [OrderDate] >= #date(2024,1,1)),  // folds to SQL
    // BAD: adding a custom column before filtering breaks folding
    // CustomCol = Table.AddColumn(Orders, "Flag", each if [Amt]>100 then "High" else "Low"),
    Selected = Table.SelectColumns(Filtered, {"OrderID","OrderDate","Amount"})
in
    Selected
```

### Parameters for environment switching
```m
// Parameter: ServerName (text), e.g. "prod-sql.database.windows.net"
let
    Source = Sql.Database(ServerName, "SalesDB")
in
    Source
```

---

## 6. Row-Level Security (RLS) and Object-Level Security (OLS)

### Static RLS
```dax
-- In "Manage Roles" → role "SalesEast":
[Region] = "East"
```

### Dynamic RLS (username-based)
```dax
-- Region table has an [Email] column matching AAD UPN
[Email] = USERPRINCIPALNAME()
```

### DAX for manager hierarchy RLS
```dax
-- Employee[ManagerEmail] chain — user sees self + all reports
[EmployeeEmail] IN
    CALCULATETABLE(
        VALUES('Employee'[EmployeeEmail]),
        FILTER(
            'Employee',
            PATHCONTAINS(
                PATH('Employee'[EmployeeID], 'Employee'[ManagerID]),
                LOOKUPVALUE('Employee'[EmployeeID], 'Employee'[EmployeeEmail], USERPRINCIPALNAME())
            )
        )
    )
```

### Assigning RLS roles
```bash
# Assign security role to AAD group via REST
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"members":[{"memberType":"Group","identifier":"aad-group-object-id"}]}' \
  "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/datasets/$DATASET_ID/users"
```

### Object-Level Security (OLS) — Premium only
- Hides entire tables or columns from specific roles
- Set via Tabular Editor or XMLA endpoint
- OLS role + RLS role can be combined on the same dataset

---

## 7. Gateway Setup and Management

### On-Premises Data Gateway
- Windows service, connects cloud refreshes to on-prem sources
- Install on a server with always-on connectivity
- Cluster multiple gateways for HA

```powershell
# List gateway clusters
Get-OnPremisesDataGatewayCluster

# List data sources on a gateway
Get-OnPremisesDataGatewayClusterDatasource -GatewayClusterId $gwId
```

```bash
# REST: get all gateways accessible to the user
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.powerbi.com/v1.0/myorg/gateways" | python3 -m json.tool
```

### Personal Gateway
- Single-user only, runs in user session (not as Windows service)
- Not suitable for shared/scheduled enterprise refresh
- Use on-prem gateway for production workloads

### Bind dataset to gateway
```bash
GATEWAY_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
DATASOURCE_ID="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"
curl -s -X PATCH \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"gatewayObjectId\":\"$GATEWAY_ID\",\"datasourceObjectIds\":[\"$DATASOURCE_ID\"]}" \
  "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/datasets/$DATASET_ID/Default.BindToGateway"
```

---

## 8. Embedded Analytics

### Embed scenarios
| Scenario | Identity | Licence needed |
|---|---|---|
| Embed for your customers (app owns data) | Service Principal | A-SKU or P-SKU capacity |
| Embed for your org (user owns data) | AAD user | Pro or PPU per viewer |
| Secure embed (iFrame in SharePoint/Teams) | AAD user | Pro |

### Service Principal setup
```bash
# Register AAD app
az ad app create --display-name "PowerBI-Embed-App"
az ad sp create --id <appId>

# Grant Power BI API permissions (via portal: Report.Read.All, Dataset.Read.All, etc.)
# Then enable service principals in Power BI tenant admin settings
```

### Generate embed token (app-owns-data)
```bash
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"accessLevel\": \"View\",
    \"identities\": [{
      \"username\": \"user@contoso.com\",
      \"roles\": [\"SalesEast\"],
      \"datasets\": [\"$DATASET_ID\"]
    }]
  }" \
  "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/reports/$REPORT_ID/GenerateToken"
```

### JavaScript embed (powerbi-client)
```html
<div id="reportContainer" style="height:600px;"></div>
<script src="https://cdn.jsdelivr.net/npm/powerbi-client/dist/powerbi.min.js"></script>
<script>
const models = window['powerbi-client'].models;
const embedConfig = {
  type: 'report',
  tokenType: models.TokenType.Embed,
  accessToken: '<embedToken>',
  embedUrl: 'https://app.powerbi.com/reportEmbed?reportId=<reportId>&groupId=<groupId>',
  id: '<reportId>',
  permissions: models.Permissions.Read,
  settings: { filterPaneEnabled: false, navContentPaneEnabled: true }
};
const reportContainer = document.getElementById('reportContainer');
const report = powerbi.embed(reportContainer, embedConfig);
report.on('loaded', () => console.log('Report loaded'));
</script>
```

---

## 9. REST API and PowerShell

### MicrosoftPowerBIMgmt module
```powershell
Install-Module -Name MicrosoftPowerBIMgmt -Scope CurrentUser

# Interactive login
Connect-PowerBIServiceAccount

# Service principal login
$cred = New-Object PSCredential($appId, (ConvertTo-SecureString $appSecret -AsPlainText -Force))
Connect-PowerBIServiceAccount -ServicePrincipal -Credential $cred -TenantId $tenantId

# Common admin commands
Get-PowerBIWorkspace -All -Scope Organization        # all workspaces (admin)
Get-PowerBIReport   -WorkspaceId $wsId
Get-PowerBIDataset  -WorkspaceId $wsId
Get-PowerBITile     -DashboardId $dashId -WorkspaceId $wsId
Export-PowerBIReport -Id $reportId -OutFile "report.pbix" -WorkspaceId $wsId
```

### REST API — common operations
```bash
# Set variables
export TOKEN=$(az account get-access-token \
  --resource https://analysis.windows.net/powerbi/api \
  --query accessToken -o tsv)
export BASE="https://api.powerbi.com/v1.0/myorg"

# Clone report
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"Sales Report - Copy\",\"targetWorkspaceId\":\"$TARGET_WS\"}" \
  "$BASE/groups/$GROUP_ID/reports/$REPORT_ID/Clone"

# Update dataset credentials
curl -s -X PATCH \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "credentialDetails": {
      "credentialType": "Basic",
      "credentials": "{\"credentialData\":[{\"name\":\"username\",\"value\":\"sa\"},{\"name\":\"password\",\"value\":\"P@ssw0rd\"}]}",
      "encryptedConnection": "Encrypted",
      "encryptionAlgorithm": "None",
      "privacyLevel": "Organizational"
    }
  }' \
  "$BASE/gateways/$GW_ID/datasources/$DS_ID"

# Get activity events (audit log) — admin only
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/admin/activityevents?startDateTime='2026-04-23T00:00:00'&endDateTime='2026-04-23T23:59:59'" \
  | python3 -m json.tool
```

---

## 10. Incremental Refresh and Aggregations

### Incremental refresh setup (Desktop → Service)
```
1. Create parameters: RangeStart (DateTime), RangeEnd (DateTime)
2. Filter table in Power Query:
   each [OrderDate] >= RangeStart and [OrderDate] < RangeEnd
3. In Desktop → right-click table → Incremental refresh:
   - Archive: 3 years
   - Refresh:  1 day (rolling)
   - Enable detect data changes (optional)
4. Publish — Service handles partition management automatically
```

### User-defined aggregations (composite models)
```
1. Import aggregated table (e.g., daily/regional totals) alongside DirectQuery detail table
2. In Model view → select agg table → "Manage aggregations"
3. Map each measure column: SummarizeBy = Sum, GroupBy columns = Date, Region
4. Set agg table storage mode = Import; detail table = DirectQuery
5. Power BI query engine auto-routes — no DAX changes needed
```

---

## 11. Licensing

| Feature | Free | Pro ($10/u/mo) | PPU ($20/u/mo) | Premium (P-SKU) | Fabric (F-SKU) |
|---|---|---|---|---|---|
| Build reports | Yes | Yes | Yes | Yes | Yes |
| Publish to shared workspace | No | Yes | Yes | Yes | Yes |
| Share with Pro users | No | Yes | Yes | Yes | Yes |
| Share with free users | No | No | No | Yes (in Premium ws) | Yes |
| XMLA endpoint | No | No | Yes | Yes | Yes |
| Deployment pipelines | No | No | Yes | Yes | Yes |
| Datamarts | No | No | Yes | Yes | Yes |
| Paginated reports | No | No | Yes | Yes | Yes |
| AI features (AutoML, etc.) | No | No | Yes | Yes | Yes |
| Embedded A-SKU | N/A | N/A | N/A | Yes | Yes |

**Key rule:** Content in a Premium/Fabric workspace can be viewed by free-licence users.

---

## 12. Microsoft Fabric Integration

### Direct Lake mode
- Reads Parquet/Delta files directly from OneLake — no data import or DirectQuery pass-through
- Best performance: in-memory speed, no copy, no Gateway
- Requires Fabric capacity (F-SKU); not available on P-SKU legacy Premium

### Lakehouse as dataset source
```
1. Create Fabric Lakehouse → load Delta tables
2. In Fabric workspace → New Semantic Model → select Lakehouse tables
3. Semantic model uses Direct Lake automatically
4. Add DAX measures, relationships — publish as normal
```

### Shortcuts — virtual data references
```
# OneLake shortcut to ADLS Gen2 (no data movement):
Fabric Lakehouse → New shortcut → Azure Data Lake Storage Gen2
→ enter storage account URL + container → select folders
```

### Notebooks → Lakehouse → Power BI pipeline
```python
# PySpark in Fabric Notebook: write to Lakehouse Delta table
df = spark.read.format("csv").option("header","true").load("Files/raw/sales.csv")
df.write.format("delta").mode("overwrite").saveAsTable("silver_sales")
# Power BI semantic model auto-refreshes via Direct Lake — no manual refresh needed
```

---

## 13. Star Schema and Performance Best Practices

### Star schema design
```
Fact table:    SalesFact    [DateKey, ProductKey, CustomerKey, StoreKey, Qty, Amount]
Dim tables:    DimDate      [DateKey, Date, Year, Quarter, Month, Week, IsWorkday]
               DimProduct   [ProductKey, Name, Category, SubCategory, Brand, Cost]
               DimCustomer  [CustomerKey, Name, Email, Region, Segment]
               DimStore     [StoreKey, Name, City, State, Country]

Rules:
  - Integer surrogate keys on all relationships
  - Single-direction relationships (bi-directional only where truly needed)
  - Date table must be marked as "Date table" with unique, contiguous date column
  - All measures in a dedicated empty "Measures" table
```

### VertiPaq compression tips
- **Sort columns** for high-cardinality columns: `Sort [CustomerID] by [Region]` (groups values)
- **Remove unused columns** — every column costs memory
- **Use integers over strings** for keys
- **Avoid calculated columns** where measures suffice (measures don't persist in memory)

### Query performance
```dax
-- Use variables to avoid re-evaluation
-- Bad:
Measure = IF([Sales] > [Budget], [Sales] - [Budget], 0)
-- (Sales and Budget each evaluated twice)

-- Good:
Measure =
VAR s = [Sales]
VAR b = [Budget]
RETURN IF(s > b, s - b, 0)
```

### Performance Analyzer (Desktop)
```
View → Performance Analyzer → Start recording → interact with visual
→ Copy query → paste into DAX Studio for deep analysis
```

```
DAX Studio (free tool):
  - Server Timings: shows SE (Storage Engine) vs FE (Formula Engine) time
  - Goal: maximize SE, minimize FE — SE is massively parallel, FE is single-threaded
```

---

## 14. Security: Sensitivity Labels, DLP, and Tenant Settings

### Sensitivity labels (Microsoft Purview integration)
```
Admin portal → Information protection → Enable sensitivity labels
Labels flow from M365 to Power BI items and exported files
```

```powershell
# Set sensitivity label on a dataset via REST
$body = @{
    "sensitivityLabelId" = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    "assignmentMethod"   = "Privileged"
} | ConvertTo-Json

Invoke-PowerBIRestMethod `
  -Url "datasets/$DATASET_ID/sensitivity" `
  -Method Patch -Body $body
```

### Data Loss Prevention (DLP) policies
- Configured in Microsoft Purview compliance portal
- Policies can block export of datasets with "Highly Confidential" label
- Alert admins when sensitive data is shared externally

### Key Tenant Settings (Admin Portal)
```
Tenant settings to review and lock down:
  - "Export data" — control CSV/Excel export per group
  - "Publish to web" — disable for most orgs (creates public URL, no auth)
  - "Allow service principals to use Power BI APIs" — enable for automation
  - "Guest user access" — restrict external sharing
  - "Certified content" — define who can certify datasets/reports
  - "Allow XMLA endpoints" — required for Tabular Editor, SSMS connections
```

### XMLA endpoint — direct model access
```
Connection string: powerbi://api.powerbi.com/v1.0/myorg/<WorkspaceName>
Tools: SSMS, Tabular Editor, DAX Studio, Analysis Services client libraries
Use for: OLS setup, advanced scripting, TMSL deployments
```

---

## 15. Common CLI Patterns and Automation

### Bulk workspace report export
```powershell
Connect-PowerBIServiceAccount
$ws = Get-PowerBIWorkspace -Name "Finance" | Select-Object -First 1
Get-PowerBIReport -WorkspaceId $ws.Id | ForEach-Object {
    $outFile = ".\exports\$($_.Name -replace '[\\/:*?"<>|]','_').pbix"
    Export-PowerBIReport -Id $_.Id -WorkspaceId $ws.Id -OutFile $outFile
    Write-Host "Exported: $($_.Name)"
}
```

### Trigger all dataset refreshes in a workspace
```bash
GROUP_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# Get all dataset IDs
DATASETS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/datasets" \
  | python3 -c "import sys,json; [print(d['id']) for d in json.load(sys.stdin)['value']]")

for DS in $DATASETS; do
  echo "Refreshing $DS"
  curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/datasets/$DS/refreshes"
done
```

### Deploy report via REST (import .pbix)
```bash
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@SalesReport.pbix" \
  "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/imports?datasetDisplayName=SalesReport&nameConflict=Overwrite"
```

### Monitor refresh status
```bash
# Poll until refresh completes
DATASET_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
while true; do
  STATUS=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/datasets/$DATASET_ID/refreshes?\$top=1" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['value'][0]['status'])")
  echo "$(date): $STATUS"
  [[ "$STATUS" != "Unknown" ]] && break
  sleep 30
done
```

### Capacity management
```bash
# List capacities (admin)
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.powerbi.com/v1.0/myorg/capacities" | python3 -m json.tool

# Assign workspace to capacity
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"capacityId\":\"$CAPACITY_ID\"}" \
  "https://api.powerbi.com/v1.0/myorg/groups/$GROUP_ID/AssignToCapacity"
```

---

## 16. Troubleshooting Quick Reference

| Symptom | Likely cause | Fix |
|---|---|---|
| Refresh fails "credentials" | Gateway datasource creds expired | Update datasource credentials in gateway |
| Refresh fails "timeout" | Query too slow, gateway timeout | Optimize query, increase gateway timeout, add incremental refresh |
| Report slow to load | Too many DirectQuery visuals | Use Import mode or aggregations; reduce visual count |
| RLS not working | User not assigned to role | Manage roles → assign AAD user/group to role |
| Embed token 401 | SP not in workspace or wrong scope | Add SP to workspace; verify API permissions granted |
| XMLA connect fails | Tenant setting disabled | Admin portal → enable XMLA endpoints for workspace |
| Direct Lake falls back to DirectQuery | Delta table schema changed | Refresh semantic model or re-map columns |
| Dataflow refresh circular ref | Dataflow depends on itself | Restructure dataflow dependency chain |
