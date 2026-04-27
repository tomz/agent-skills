---
name: sf-lwc
description: Lightning Web Components — component structure, reactive properties, lifecycle hooks, wire service, imperative Apex, events, navigation, LDS, and common patterns
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

# Lightning Web Components (LWC) Skill

## Overview

LWC is Salesforce's modern UI framework built on web standards (custom elements, shadow DOM,
ES modules). Components live in `force-app/main/default/lwc/<componentName>/` and consist
of up to 5 files. LWC replaced Aura as the recommended component model.

---

## Component Structure

Every component is a directory with the same name (camelCase → kebab-case in HTML):

```
lwc/
└── myAccountCard/
    ├── myAccountCard.html        # Template (required)
    ├── myAccountCard.js          # Controller (required)
    ├── myAccountCard.css         # Styles (optional, shadow-scoped)
    ├── myAccountCard.js-meta.xml # Config/targets (required for deployment)
    └── myAccountCard.svg         # Custom icon (optional)
```

### myAccountCard.js-meta.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>61.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__AppPage</target>
        <target>lightning__RecordPage</target>
        <target>lightning__HomePage</target>
        <target>lightning__Tab</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__RecordPage">
            <property name="recordId" type="String" />
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

---

## Template Syntax

```html
<!-- myAccountCard.html -->
<template>
    <!-- Conditional rendering -->
    <template if:true={isLoaded}>
        <p>Name: {account.Name}</p>
        <p>Rating: {account.Rating}</p>
    </template>

    <template if:false={isLoaded}>
        <lightning-spinner alternative-text="Loading..."></lightning-spinner>
    </template>

    <!-- Iteration -->
    <template for:each={contacts} for:item="contact" for:index="idx">
        <p key={contact.Id}>{idx}: {contact.Name}</p>
    </template>

    <!-- Iterator (first/last access) -->
    <template iterator:it={contacts}>
        <li key={it.value.Id}
            class={it.first ? 'first' : ''}>
            {it.value.Name}
        </li>
    </template>

    <!-- Slot (for parent to inject content) -->
    <slot name="header"></slot>
    <slot></slot>
</template>
```

**LWC uses `if:true` / `if:false` directives (not `v-if`). In API v59+, use `lwc:if`,
`lwc:elseif`, `lwc:else` for better performance:**
```html
<template lwc:if={isLoaded}>...</template>
<template lwc:elseif={isError}>...</template>
<template lwc:else>...</template>
```

---

## Reactive Properties

```javascript
// myAccountCard.js
import { LightningElement, api, track, wire } from 'lwc';

export default class MyAccountCard extends LightningElement {
    // @api — public property, settable by parent component
    @api recordId;          // automatically set on record pages
    @api title = 'Account Details';

    // @track — deep reactivity for objects/arrays (objects/arrays are reactive by default in modern LWC)
    // In API v39+, primitive @track is unnecessary; objects need @track for deep property changes
    @track filters = { status: 'Active', type: 'Customer' };

    // Private reactive (plain property — primitives are reactive)
    isLoaded = false;
    errorMessage;

    // Computed getter (recalculated when dependencies change)
    get formattedTitle() {
        return `${this.title} — ${this.recordId}`;
    }

    get hasError() {
        return !!this.errorMessage;
    }
}
```

---

## Lifecycle Hooks

```javascript
export default class MyComponent extends LightningElement {
    // Called after component is inserted into DOM
    // Safe to access DOM elements (this.template.querySelector)
    connectedCallback() {
        this.loadData();
        document.addEventListener('keydown', this.handleKeydown.bind(this));
    }

    // Called when component is removed from DOM — clean up subscriptions, listeners
    disconnectedCallback() {
        document.removeEventListener('keydown', this.handleKeydown.bind(this));
    }

    // Called after every render (initial + subsequent)
    renderedCallback() {
        // Fires often — use a flag to run logic only once
        if (this.initialized) return;
        this.initialized = true;
        // Access DOM, initialize third-party libs
        const canvas = this.template.querySelector('canvas');
        this.initChart(canvas);
    }

    // Called when @api property set by parent (before render)
    // Used in Aura-interop or when you need to react to property change
    // LWC doesn't have a dedicated "property changed" hook — use getters/setters
    @api
    get recordId() { return this._recordId; }
    set recordId(value) {
        this._recordId = value;
        this.loadRecord();  // React to property change
    }
    _recordId;
}
```

---

## Wire Service

Wire adapters automatically fetch data and re-fetch when parameters change.

### Wire with LDS (Lightning Data Service)
```javascript
import { LightningElement, wire, api } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';
import NAME_FIELD from '@salesforce/schema/Account.Name';
import RATING_FIELD from '@salesforce/schema/Account.Rating';
import INDUSTRY_FIELD from '@salesforce/schema/Account.Industry';

export default class AccountWire extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields: [NAME_FIELD, RATING_FIELD, INDUSTRY_FIELD] })
    account;  // { data, error } object

    get name() {
        return getFieldValue(this.account.data, NAME_FIELD);
    }

    get isLoading() {
        return !this.account.data && !this.account.error;
    }
}
```

### Wire with Apex
```javascript
import { LightningElement, wire, api } from 'lwc';
import getAccountContacts from '@salesforce/apex/ContactController.getAccountContacts';

export default class ContactList extends LightningElement {
    @api recordId;

    // Wire to Apex — Apex method must be @AuraEnabled(cacheable=true)
    @wire(getAccountContacts, { accountId: '$recordId' })
    wiredContacts({ data, error }) {
        if (data) {
            this.contacts = data;
            this.error = undefined;
        } else if (error) {
            this.error = error.body.message;
            this.contacts = undefined;
        }
    }

    contacts;
    error;
}
```

**Apex method for cacheable wire:**
```apex
@AuraEnabled(cacheable=true)
public static List<Contact> getAccountContacts(Id accountId) {
    return [SELECT Id, Name, Email, Phone FROM Contact WHERE AccountId = :accountId];
}
```

---

## Imperative Apex Calls

Use when you need user-triggered calls, non-cacheable operations, or DML.

```javascript
import { LightningElement, api } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import saveAccount from '@salesforce/apex/AccountController.saveAccount';

export default class AccountEditor extends LightningElement {
    @api recordId;
    isLoading = false;

    handleSave() {
        const name = this.template.querySelector('[data-id="name"]').value;
        this.isLoading = true;

        saveAccount({ accountId: this.recordId, name })
            .then(result => {
                this.dispatchEvent(new ShowToastEvent({
                    title: 'Success',
                    message: 'Account saved',
                    variant: 'success'
                }));
            })
            .catch(error => {
                this.dispatchEvent(new ShowToastEvent({
                    title: 'Error',
                    message: error.body.message,
                    variant: 'error'
                }));
            })
            .finally(() => {
                this.isLoading = false;
            });
    }
}
```

---

## Events (Child → Parent Communication)

### Child dispatches custom event
```javascript
// childButton.js
handleClick() {
    this.dispatchEvent(new CustomEvent('save', {
        detail: { id: this.recordId, name: this.name },
        bubbles: true,    // propagates up DOM tree
        composed: true    // crosses shadow DOM boundary
    }));
}
```

### Parent listens
```html
<!-- parent.html -->
<c-child-button onsave={handleSave}></c-child-button>
```
```javascript
// parent.js
handleSave(event) {
    const { id, name } = event.detail;
    console.log('Saving:', id, name);
}
```

### Parent → Child (call public method)
```html
<!-- parent.html -->
<c-child-component lwc:ref="myChild"></c-child-component>
<button onclick={handleReset}>Reset</button>
```
```javascript
// parent.js
handleReset() {
    this.refs.myChild.reset(); // Call @api method on child
}

// child.js
@api reset() {
    this.value = '';
}
```

---

## Navigation

```javascript
import { LightningElement } from 'lwc';
import { NavigationMixin } from 'lightning/navigation';

export default class MyNav extends NavigationMixin(LightningElement) {
    navigateToRecord() {
        this[NavigationMixin.Navigate]({
            type: 'standard__recordPage',
            attributes: { recordId: '001...', actionName: 'view' }
        });
    }

    navigateToList() {
        this[NavigationMixin.Navigate]({
            type: 'standard__objectPage',
            attributes: { objectApiName: 'Account', actionName: 'list' },
            state: { filterName: 'Recent' }
        });
    }

    navigateToURL() {
        this[NavigationMixin.Navigate]({
            type: 'standard__webPage',
            attributes: { url: 'https://salesforce.com' }
        });
    }
}
```

---

## Lightning Data Service — Create/Edit/Delete

```javascript
import { createRecord, updateRecord, deleteRecord } from 'lightning/uiRecordApi';
import { getRecordNotifyChange } from 'lightning/uiRecordApi';
import ACCOUNT_OBJECT from '@salesforce/schema/Account';
import NAME_FIELD from '@salesforce/schema/Account.Name';

// Create
const fields = {};
fields[NAME_FIELD.fieldApiName] = 'New Account';
createRecord({ apiName: ACCOUNT_OBJECT.objectApiName, fields })
    .then(record => console.log('Created:', record.id));

// Update
const fields = { Id: this.recordId };
fields[NAME_FIELD.fieldApiName] = 'Updated Name';
updateRecord({ fields })
    .then(() => getRecordNotifyChange([{ recordId: this.recordId }]));

// Delete
deleteRecord(this.recordId).then(() => console.log('Deleted'));
```

---

## lightning-datatable

```html
<lightning-datatable
    key-field="Id"
    data={contacts}
    columns={columns}
    sorted-by={sortedBy}
    sorted-direction={sortedDirection}
    onrowaction={handleRowAction}
    onsort={handleSort}
    onrowselection={handleSelection}
    selected-rows={selectedRows}>
</lightning-datatable>
```

```javascript
columns = [
    { label: 'Name', fieldName: 'Name', type: 'text', sortable: true },
    { label: 'Email', fieldName: 'Email', type: 'email' },
    { label: 'Phone', fieldName: 'Phone', type: 'phone' },
    {
        type: 'action',
        typeAttributes: {
            rowActions: [
                { label: 'View', name: 'view' },
                { label: 'Delete', name: 'delete' }
            ]
        }
    }
];

handleRowAction(event) {
    const action = event.detail.action;
    const row = event.detail.row;
    if (action.name === 'view') this.navigateToRecord(row.Id);
    if (action.name === 'delete') this.deleteRecord(row.Id);
}
```

---

## Common Gotchas

- **Shadow DOM**: `document.querySelector` won't find elements inside LWC. Use
  `this.template.querySelector()` instead. Cross-component queries don't work.
- **@api mutability**: Never mutate `@api` properties directly — they're read-only from inside
  the component. Clone first: `this.localCopy = [...this.apiProp]`.
- **Wire re-fetching**: Wire doesn't re-fetch after DML. Use `refreshApex(this.wiredResult)`
  (import from `lightning/uiRecordApi`) to force a refresh.
- **Cacheable vs non-cacheable**: Wire adapters require `@AuraEnabled(cacheable=true)`.
  Cacheable methods cannot perform DML. Use imperative calls for DML.
- **`lwc:if` vs `if:true`**: `if:true` is deprecated in API 59+. Use `lwc:if` for new work.
- **Event bubbling**: Custom events don't cross shadow DOM by default. Set `composed: true`
  to propagate across component boundaries.
- **`renderedCallback` loops**: Can cause infinite re-render loops if you modify tracked
  properties inside it without a guard flag.
