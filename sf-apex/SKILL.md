---
name: sf-apex
description: Apex language patterns — triggers, classes, batch/queueable/schedulable, governor limits, bulkification, trigger frameworks, test classes, and debug logs
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

# Salesforce Apex Skill

## Overview

Apex is Salesforce's strongly-typed, Java-like language that runs on Salesforce servers.
It is compiled and executed in a multi-tenant environment with strict governor limits
enforced per transaction. Every line of Apex must be written with bulkification in mind.

---

## Governor Limits (Memorize These)

| Resource | Synchronous | Asynchronous |
|----------|-------------|--------------|
| SOQL queries | 100 | 200 |
| SOQL rows returned | 50,000 | 50,000 |
| DML statements | 150 | 150 |
| DML rows | 10,000 | 10,000 |
| CPU time | 10,000 ms | 60,000 ms |
| Heap size | 6 MB | 12 MB |
| Callouts | 100 | 100 |
| Future methods | 50 | — |
| Queueable jobs enqueued | 50 | 1 (can chain) |
| Email invocations | 10 | 10 |

Check limits at runtime:
```apex
System.debug(Limits.getQueries() + ' / ' + Limits.getLimitQueries());
System.debug(Limits.getDmlStatements() + ' / ' + Limits.getLimitDmlStatements());
System.debug(Limits.getCpuTime() + ' / ' + Limits.getLimitCpuTime());
```

---

## Bulkification — The Golden Rule

**Never put SOQL or DML inside a loop.** Always collect IDs/records, query once, process in bulk.

### BAD (anti-pattern)
```apex
for (Account acc : trigger.new) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id]; // SOQL in loop!
    for (Contact c : contacts) {
        c.Description = 'Updated';
        update c; // DML in loop!
    }
}
```

### GOOD (bulkified)
```apex
Set<Id> accountIds = new Map<Id, Account>(trigger.new).keySet();

Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
    if (!contactsByAccount.containsKey(c.AccountId)) {
        contactsByAccount.put(c.AccountId, new List<Contact>());
    }
    contactsByAccount.get(c.AccountId).add(c);
}

List<Contact> toUpdate = new List<Contact>();
for (Account acc : trigger.new) {
    if (contactsByAccount.containsKey(acc.Id)) {
        for (Contact c : contactsByAccount.get(acc.Id)) {
            c.Description = 'Updated';
            toUpdate.add(c);
        }
    }
}
update toUpdate;
```

---

## Trigger Framework (TriggerHandler Pattern)

Avoid logic directly in trigger files. Use a handler class.

### AccountTrigger.trigger
```apex
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    new AccountTriggerHandler().run();
}
```

### TriggerHandler.cls (base class)
```apex
public virtual class TriggerHandler {
    @TestVisible private static Boolean bypass = false;

    public void run() {
        if (bypass) return;
        if (Trigger.isBefore) {
            if (Trigger.isInsert) beforeInsert();
            if (Trigger.isUpdate) beforeUpdate();
            if (Trigger.isDelete) beforeDelete();
        } else {
            if (Trigger.isInsert) afterInsert();
            if (Trigger.isUpdate) afterUpdate();
            if (Trigger.isDelete) afterDelete();
            if (Trigger.isUndelete) afterUndelete();
        }
    }

    protected virtual void beforeInsert() {}
    protected virtual void beforeUpdate() {}
    protected virtual void beforeDelete() {}
    protected virtual void afterInsert() {}
    protected virtual void afterUpdate() {}
    protected virtual void afterDelete() {}
    protected virtual void afterUndelete() {}

    public static void bypass(Boolean b) { bypass = b; }
}
```

### AccountTriggerHandler.cls
```apex
public class AccountTriggerHandler extends TriggerHandler {
    private List<Account> newList = (List<Account>) Trigger.new;
    private Map<Id, Account> oldMap = (Map<Id, Account>) Trigger.oldMap;

    protected override void beforeInsert() {
        AccountService.setDefaultRating(newList);
    }

    protected override void afterInsert() {
        AccountService.createRelatedRecords(newList);
    }

    protected override void beforeUpdate() {
        AccountService.validateChanges(newList, oldMap);
    }
}
```

---

## Batch Apex

Use for processing > 10,000 records or when you need fresh governor limits per chunk.

```apex
global class AccountBatch implements Database.Batchable<SObject>, Database.Stateful {
    global Integer processedCount = 0;

    global Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator(
            'SELECT Id, Name, Rating FROM Account WHERE Rating = null'
        );
    }

    global void execute(Database.BatchableContext bc, List<Account> scope) {
        List<Account> toUpdate = new List<Account>();
        for (Account acc : scope) {
            acc.Rating = 'Warm';
            toUpdate.add(acc);
            processedCount++;
        }
        update toUpdate;
    }

    global void finish(Database.BatchableContext bc) {
        System.debug('Processed: ' + processedCount);
        // Send email, trigger follow-up job, etc.
    }
}

// Execute:
Id jobId = Database.executeBatch(new AccountBatch(), 200); // scope size 200
```

**Batch gotchas:**
- Default scope size is 200; max is 2,000. Start small (200) for callout-heavy jobs.
- `Database.Stateful` preserves instance variables between chunks (uses heap — use sparingly).
- Cannot make callouts in `execute()` unless also implementing `Database.AllowsCallouts`.
- Batch jobs run asynchronously; test with `Test.startTest()` / `Test.stopTest()`.

---

## Queueable Apex

Lighter-weight async; supports chaining and can accept complex objects.

```apex
public class EmailQueueable implements Queueable, Database.AllowsCallouts {
    private List<Id> contactIds;

    public EmailQueueable(List<Id> contactIds) {
        this.contactIds = contactIds;
    }

    public void execute(QueueableContext ctx) {
        List<Contact> contacts = [SELECT Id, Email FROM Contact WHERE Id IN :contactIds];
        // Make callouts, send emails, chain another job
        System.enqueueJob(new FollowUpQueueable(contactIds)); // Chain
    }
}

// Enqueue:
System.enqueueJob(new EmailQueueable(myContactIds));
```

---

## Schedulable Apex

```apex
global class WeeklyReportScheduler implements Schedulable {
    global void execute(SchedulableContext ctx) {
        Database.executeBatch(new ReportBatch(), 200);
    }
}

// Schedule via Apex (runs every Monday at 8am):
String cron = '0 0 8 ? * MON';
System.schedule('Weekly Report', cron, new WeeklyReportScheduler());

// Or schedule via Setup → Apex Classes → Schedule Apex
```

---

## Future Methods

Simplest async option. Can make callouts. Cannot be chained or passed SObjects.

```apex
public class IntegrationService {
    @future(callout=true)
    public static void syncToExternalSystem(Set<Id> accountIds) {
        List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];
        // Make HTTP callouts here
    }
}
```

**Future gotchas:** Max 50 per transaction. Cannot pass SObjects (only primitives/collections
of primitives). Cannot call another `@future` from a `@future`.

---

## Test Classes

Minimum 75% overall code coverage required for deployment to production. Aim for 90%+.

```apex
@isTest
private class AccountServiceTest {

    @TestSetup
    static void makeData() {
        // @TestSetup runs once per test class; each test method gets a rollback
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(Name = 'Test Account ' + i));
        }
        insert accounts;
    }

    @isTest
    static void testSetDefaultRating_Positive() {
        List<Account> accounts = [SELECT Id, Name, Rating FROM Account LIMIT 200];

        Test.startTest();
        AccountService.setDefaultRating(accounts);
        Test.stopTest(); // Flushes async jobs (@future, batch, queueable)

        // Reload from DB after DML
        accounts = [SELECT Id, Rating FROM Account WHERE Id IN :accounts];
        for (Account acc : accounts) {
            System.assertEquals('Warm', acc.Rating, 'Rating should be Warm');
        }
    }

    @isTest
    static void testSetDefaultRating_NullInput() {
        Test.startTest();
        AccountService.setDefaultRating(null); // Should not throw
        Test.stopTest();
        // No assertion needed — verifying no exception
    }

    @isTest
    static void testBulkLoad_200Records() {
        // Explicitly test bulk behavior
        List<Account> accounts = [SELECT Id FROM Account];
        System.assertEquals(200, accounts.size());
        // Run trigger logic
        Test.startTest();
        update accounts;
        Test.stopTest();
        // Assert no governor limit exceptions thrown
    }
}
```

**Test best practices:**
- Use `@TestSetup` for shared data (faster than `@testSetup` per-method inserts).
- Always wrap async code in `Test.startTest()` / `Test.stopTest()`.
- Use `System.assertEquals(expected, actual, message)` — put expected first.
- Test bulkification explicitly with 200 records (standard trigger batch size).
- Use `TriggerHandler.bypass(true)` in tests that don't need trigger logic.
- Never use `seeAllData=true` except for legacy Pricebook tests.
- Mock callouts with `HttpCalloutMock` interface.

### Mocking HTTP callouts
```apex
@isTest
global class MockHttpResponse implements HttpCalloutMock {
    global HTTPResponse respond(HTTPRequest req) {
        HttpResponse res = new HttpResponse();
        res.setHeader('Content-Type', 'application/json');
        res.setBody('{"status":"ok"}');
        res.setStatusCode(200);
        return res;
    }
}

// In test:
Test.setMock(HttpCalloutMock.class, new MockHttpResponse());
Test.startTest();
MyCalloutClass.doCallout();
Test.stopTest();
```

---

## Error Handling Patterns

```apex
// Database.insert with partial save (allOrNone=false)
List<Database.SaveResult> results = Database.insert(records, false);
List<Account> failed = new List<Account>();
for (Integer i = 0; i < results.size(); i++) {
    if (!results[i].isSuccess()) {
        Database.Error err = results[i].getErrors()[0];
        System.debug('Failed: ' + records[i].Name + ' — ' + err.getMessage());
        failed.add(records[i]);
    }
}

// Custom exception
public class AccountServiceException extends Exception {}
throw new AccountServiceException('Account Rating is required for enterprise accounts');
```

---

## Common Gotchas

- **Trigger recursion**: Triggers fire again when you update records inside a trigger.
  Use a static boolean flag (`private static Boolean hasRun = false;`) to prevent infinite loops.
- **Mixed DML**: Cannot perform DML on setup objects (User, Profile) and non-setup objects
  in the same transaction. Use `@future` to separate them.
- **SOQL in loops**: The #1 cause of governor limit exceptions. Always query outside loops.
- **Null pointer exceptions**: Always null-check before accessing fields on related objects.
  `account.Owner.Email` will NPE if Owner wasn't queried.
- **Order of execution**: Validation rules fire after `before` triggers. If you set a field
  in a `before insert` trigger, the validation rule sees the updated value.
- **Test isolation**: Test methods share nothing unless using `@TestSetup`. Each test method
  runs in its own transaction (rolled back after).
