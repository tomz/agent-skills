---
name: sf-data
description: SOQL, SOSL, Data Loader, sf data CLI, Bulk API 2.0, Record Types, sharing rules, field-level security, and data management patterns
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

# Salesforce Data Skill

## Overview

Salesforce data access is governed by a layered security model (org-wide defaults →
sharing rules → role hierarchy → manual sharing → FLS → record types). Understanding
this stack is essential for correct data access patterns. SOQL is the primary query
language; SOSL is for cross-object text search.

---

## SOQL — Salesforce Object Query Language

### Basic syntax
```sql
SELECT Id, Name, Email, Account.Name
FROM Contact
WHERE AccountId != null
  AND CreatedDate = LAST_N_DAYS:30
  AND (Email LIKE '%@acme.com' OR Email LIKE '%@acme.org')
ORDER BY LastName ASC, FirstName ASC
LIMIT 200
OFFSET 100
```

### Date literals (no quotes needed)
```sql
WHERE CreatedDate = TODAY
WHERE CreatedDate = THIS_WEEK
WHERE CreatedDate = LAST_MONTH
WHERE CreatedDate = LAST_N_DAYS:7
WHERE CreatedDate >= 2024-01-01T00:00:00Z
WHERE CloseDate = THIS_FISCAL_QUARTER
```

### Relationship queries

**Child-to-parent (dot notation — up to 5 levels):**
```sql
SELECT Id, Name, Account.Name, Account.Owner.Name, Account.Owner.Profile.Name
FROM Contact
WHERE Account.Industry = 'Technology'
```

**Parent-to-child (subquery — one level deep):**
```sql
SELECT Id, Name,
    (SELECT Id, Subject, Status FROM Cases ORDER BY CreatedDate DESC LIMIT 5),
    (SELECT Id, Name, Email FROM Contacts WHERE IsActive__c = true)
FROM Account
WHERE Type = 'Customer'
```

### Aggregate functions
```sql
-- COUNT, SUM, AVG, MIN, MAX, COUNT_DISTINCT
SELECT AccountId, COUNT(Id) total, SUM(Amount) totalAmount, AVG(Amount) avgAmount
FROM Opportunity
WHERE StageName = 'Closed Won'
  AND CloseDate = THIS_YEAR
GROUP BY AccountId
HAVING COUNT(Id) > 5
ORDER BY COUNT(Id) DESC
```

### SOQL for loops (avoids heap limits for large datasets)
```apex
// DON'T: loads all records into heap at once
List<Account> allAccounts = [SELECT Id FROM Account]; // up to 50k rows → heap issues

// DO: process in chunks of 200 (one chunk per iteration)
for (List<Account> chunk : [SELECT Id, Name FROM Account WHERE IsActive__c = true]) {
    // chunk is up to 200 records; governor limits apply PER ITERATION
    for (Account acc : chunk) {
        // process
    }
}
```

### Dynamic SOQL
```apex
// Use Database.query() for dynamic queries
String objectType = 'Contact';
String filterField = 'Email';
String filterValue = '%@acme.com';

String soql = 'SELECT Id, Name, Email FROM ' + objectType
    + ' WHERE ' + filterField + ' LIKE :filterValue'
    + ' LIMIT 200';

List<SObject> results = Database.query(soql);

// SECURITY: Never concatenate user input directly — use bind variables
// BAD:  'WHERE Name = \'' + userInput + '\''  → SOQL injection
// GOOD: 'WHERE Name = :userInput'             → bind variable (safe)
```

### Semi-joins and anti-joins
```apex
// Semi-join: Contacts who have open cases
SELECT Id FROM Contact
WHERE Id IN (SELECT ContactId FROM Case WHERE Status != 'Closed')

// Anti-join: Contacts with NO open cases
SELECT Id FROM Contact
WHERE Id NOT IN (SELECT ContactId FROM Case WHERE Status != 'Closed')

// Useful for finding orphaned records:
SELECT Id FROM Contact WHERE AccountId NOT IN (SELECT Id FROM Account)
```

---

## SOSL — Salesforce Object Search Language

Use SOSL for cross-object text search (like a search engine, not a filter).

```apex
List<List<SObject>> searchResults = [
    FIND 'acme*'
    IN ALL FIELDS           -- ALL FIELDS, NAME FIELDS, EMAIL FIELDS, PHONE FIELDS
    RETURNING
        Account(Id, Name, Phone ORDER BY Name LIMIT 10),
        Contact(Id, Name, Email WHERE AccountId != null),
        Lead(Id, Name, Company)
    WITH DATA CATEGORY Geography__c AT usa__c
    LIMIT 200
];

List<Account> accounts = (List<Account>) searchResults[0];
List<Contact> contacts = (List<Contact>) searchResults[1];
List<Lead> leads       = (List<Lead>)    searchResults[2];
```

**SOSL vs SOQL:**
- SOSL: searches across multiple objects, text-based, returns up to 2,000 records total
- SOQL: precise filter on one object (+ child via subquery), returns up to 50,000 rows
- Use SOSL for global search; use SOQL when you know the object and field

---

## sf CLI Data Commands

```bash
# Query (human-readable)
sf data query \
  --query "SELECT Id, Name FROM Account LIMIT 10" \
  --target-org myorg

# Query to CSV file
sf data query \
  --query "SELECT Id, Name, Email FROM Contact WHERE IsActive__c = true" \
  --result-format csv \
  --target-org myorg > contacts.csv

# Import (upsert from CSV)
sf data upsert bulk \
  --sobject Contact \
  --file contacts.csv \
  --external-id Email \
  --target-org myorg \
  --wait 10

# Import (insert)
sf data import bulk \
  --sobject Account \
  --file accounts.csv \
  --target-org myorg \
  --wait 10

# Delete records from CSV of IDs
sf data delete bulk \
  --sobject Contact \
  --file contact_ids.csv \
  --target-org myorg \
  --wait 10

# Get bulk job status
sf data bulk results --job-id 750... --target-org myorg
```

---

## Bulk API 2.0 (Direct REST)

For large-scale operations (millions of records). Works in chunks up to 150MB per job.

```bash
# Step 1: Create job
curl -s -X POST https://myorg.my.salesforce.com/services/data/v61.0/jobs/ingest \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"object":"Contact","operation":"upsert","externalIdFieldName":"Email__c","contentType":"CSV"}' \
  | python3 -m json.tool

# Step 2: Upload CSV data
curl -X PUT "https://myorg.my.salesforce.com/services/data/v61.0/jobs/ingest/$JOB_ID/batches" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: text/csv" \
  --data-binary @contacts.csv

# Step 3: Close job (triggers processing)
curl -X PATCH "https://myorg.my.salesforce.com/services/data/v61.0/jobs/ingest/$JOB_ID" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"state":"UploadComplete"}'

# Step 4: Poll for completion
curl "https://myorg.my.salesforce.com/services/data/v61.0/jobs/ingest/$JOB_ID" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
# state: Open → UploadComplete → InProgress → JobComplete

# Step 5: Download results
curl "https://myorg.my.salesforce.com/services/data/v61.0/jobs/ingest/$JOB_ID/successfulResults" \
  -H "Authorization: Bearer $ACCESS_TOKEN" > success.csv
curl "https://myorg.my.salesforce.com/services/data/v61.0/jobs/ingest/$JOB_ID/failedResults" \
  -H "Authorization: Bearer $ACCESS_TOKEN" > failed.csv
```

---

## Record Types

Record types allow different picklist values and page layouts per profile.

```apex
// Get Record Type Id by developer name (cache this — don't query in loops)
Id customerRT = Schema.SObjectType.Account.getRecordTypeInfosByDeveloperName()
    .get('Customer').getRecordTypeId();

// Create record with specific type
Account acc = new Account(
    Name = 'Acme Corp',
    RecordTypeId = customerRT
);
insert acc;

// Query by record type
List<Account> customers = [
    SELECT Id, Name FROM Account
    WHERE RecordType.DeveloperName = 'Customer'
];

// Get all record types for an object
Map<String, Schema.RecordTypeInfo> rtMap =
    Schema.SObjectType.Account.getRecordTypeInfosByDeveloperName();
for (String devName : rtMap.keySet()) {
    System.debug(devName + ': ' + rtMap.get(devName).getRecordTypeId());
}
```

---

## Sharing Rules and OWD

**Org-Wide Defaults (OWD)** → **Role Hierarchy** → **Sharing Rules** → **Manual Sharing**

```apex
// Query with sharing (respects user's sharing — use in LWC/controllers)
public with sharing class AccountController {
    public static List<Account> getAccounts() {
        return [SELECT Id, Name FROM Account]; // Filtered by user's access
    }
}

// Query without sharing (system-level access — use for triggers, batch)
public without sharing class AccountBatch implements Database.Batchable<SObject> {
    // Sees all records regardless of sharing
}

// Inherited sharing (uses calling context's sharing — safest default for reusable classes)
public inherited sharing class AccountService {
    // If called from a with sharing context → with sharing
    // If called from a without sharing context → without sharing
}
```

### Manual sharing in Apex
```apex
// Share a record with a user
AccountShare share = new AccountShare(
    AccountId = accountId,
    UserOrGroupId = userId,
    AccountAccessLevel = 'Edit',   // Read, Edit
    OpportunityAccessLevel = 'Read',
    CaseAccessLevel = 'None',
    RowCause = Schema.AccountShare.RowCause.Manual
);
insert share;
```

---

## Field-Level Security (FLS)

In controllers, always check FLS before reading/writing fields (especially for
AppExchange apps). Use `WITH SECURITY_ENFORCED` or `Security.stripInaccessible()`.

```apex
// Option 1: WITH SECURITY_ENFORCED (throws QueryException if field not accessible)
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue
    FROM Account
    WITH SECURITY_ENFORCED
];

// Option 2: stripInaccessible (strips inaccessible fields silently)
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, Name, SSN__c FROM Contact]
);
List<Contact> safeContacts = (List<Contact>) decision.getRecords();

// Check FLS manually
if (!Schema.SObjectType.Account.fields.AnnualRevenue.isAccessible()) {
    throw new AuraHandledException('Insufficient field access');
}
```

---

## Data Export Strategies

```bash
# Weekly export (Setup → Data Export)
# Or use sf CLI for full org export:
sf data export bulk \
  --query "SELECT Id, Name, Email FROM Contact" \
  --output-file contacts.csv \
  --target-org myorg \
  --wait 10

# Replicate all data for a sandbox refresh
# Use Data Loader with mapped CSV files + load order script
# Order matters: parents before children (Account before Contact)
```

### Load order for related objects
```
1. Account
2. Contact (AccountId)
3. Opportunity (AccountId)
4. OpportunityContactRole (OpportunityId, ContactId)
5. Case (AccountId, ContactId)
```

---

## Common Gotchas

- **SOQL injection**: Never concatenate user-controlled strings into dynamic SOQL.
  Use bind variables (`:variable`) — they are parameterized and safe.
- **Relationship field names**: Custom relationship fields use `__r` suffix
  (e.g., `Account__r.Name`). Standard relationships use the object name
  (e.g., `Account.Name` from Contact, not `AccountId.Name`).
- **NULL in SOQL**: Use `= null` or `!= null`, not `IS NULL` (that's SQL, not SOQL).
- **Governor limits on queries**: 100 SOQL per transaction. A SOQL for loop counts as
  ONE query, not one per iteration.
- **Bulk API CSV format**: First row must be column headers matching API field names.
  Dates must be ISO 8601: `2024-01-15T00:00:00.000Z`.
- **Record Type by Name vs DeveloperName**: `getRecordTypeInfosByName()` uses the label
  (changes with translations). Use `getRecordTypeInfosByDeveloperName()` for stability.
- **with sharing in triggers**: Triggers always run `without sharing` (system context).
  If you need user-based sharing in trigger logic, call a `with sharing` method.
