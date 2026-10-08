---
name: cap-cds-model
description: >
  Expert assistant for writing SAP CAP CDS schema, service definitions, annotations,
  and authorization rules. Use this skill whenever the user mentions CDS, CAP, .cds files,
  entity, aspect, association, composition, service definition, @readonly, @requires,
  @restrict, Fiori annotations, OData, value lists, draft handling, CAP projections,
  views, actions, functions, or any SAP Cloud Application Programming Model topic.
  Also triggers when the user asks about defining data models, exposing entities as OData
  services, adding Fiori UI annotations, securing CAP services, or writing CAP event handlers.
---

# SAP CAP CDS Model Skill

You are an expert in SAP Cloud Application Programming (CAP) Model and Core Data Services (CDS). Help users write correct CDS schema, service definitions, annotations, and authorization rules. Always follow CAP best practices and use the latest CDS syntax.

## Project Structure

```
my-cap-app/
├── db/
│   └── schema.cds        # Data model — entities, types, aspects
├── srv/
│   └── service.cds       # Service definition — what to expose via OData
│   └── service.js        # Optional: custom handlers
├── app/
│   └── fiori.cds         # Fiori UI annotations
└── package.json
```

---

## Data Model (schema.cds)

### Basic Entity

```cds
namespace my.app;

entity Incidents {
  key ID        : UUID;
      title     : String(100);
      descr     : String;
      status    : String enum { open; in_progress; closed; };
      priority  : Integer default 1;
      createdAt : Timestamp;
      createdBy : String;
}
```

### Using Built-in Aspects

CAP provides managed aspects — always prefer these over manual timestamp/user fields:

```cds
using { managed, cuid } from '@sap/cds/common';

entity Incidents : cuid, managed {
  // cuid  → adds key ID : UUID (auto-generated)
  // managed → adds createdAt, createdBy, modifiedAt, modifiedBy (auto-filled)
  title    : String(100);
  descr    : String;
  status   : String enum { open; in_progress; closed; } default 'open';
  priority : Integer default 1;
}
```

### Associations (to-one)

```cds
entity Incidents : cuid, managed {
  title    : String(100);
  customer : Association to Customers;   // foreign key added automatically
  assignee : Association to Users;
}

entity Customers : cuid, managed {
  name  : String(100);
  email : String(100);
}
```

### Compositions (to-many, parent-child)

Use `Composition` when child records are owned by the parent (lifecycle tied together):

```cds
entity Incidents : cuid, managed {
  title       : String(100);
  comments    : Composition of many Comments on comments.incident = $self;
}

entity Comments : cuid, managed {
  incident : Association to Incidents;
  text     : String;
}
```

### Types and Code Lists

```cds
// Reusable type
type Priority : Integer enum {
  low    = 1;
  medium = 2;
  high   = 3;
}

// Code list (generates a value help entity)
entity Statuses : CodeList {
  key code : String enum { open; in_progress; closed; };
}

entity Incidents : cuid, managed {
  title    : String(100);
  priority : Priority default 2;
  status   : Association to Statuses;
}
```

### Aspects (Reusable Mixins)

```cds
aspect Auditable {
  reviewedAt : Timestamp;
  reviewedBy : String;
  notes      : String;
}

entity Incidents : cuid, managed, Auditable {
  title : String(100);
}
```

---

## Service Definition (service.cds)

### Basic Service

```cds
using { my.app as db } from '../db/schema';

service IncidentService @(path: '/incident') {
  entity Incidents as projection on db.Incidents;
  entity Customers as projection on db.Customers;
}
```

### Read-Only Projection

```cds
service IncidentService {
  @readonly
  entity Incidents as projection on db.Incidents;
}
```

### Restricted Projection (expose subset of fields)

```cds
service IncidentService {
  entity Incidents as select from db.Incidents {
    ID, title, status, priority, customer
  } excluding { descr };
}
```

### Actions and Functions

```cds
service IncidentService {
  entity Incidents as projection on db.Incidents;

  // Action — modifies data, bound to an entity instance
  action   closeIncident(reason: String) returns Incidents;

  // Function — read-only, unbound
  function getOpenCount() returns Integer;
}
```

### Draft Handling (for Fiori draft-enabled apps)

```cds
service IncidentService {
  @odata.draft.enabled
  entity Incidents as projection on db.Incidents;
}
```

---

## Authorization (@requires / @restrict)

### Role-Based Access

```cds
service IncidentService @(requires: 'authenticated-user') {

  @restrict: [
    { grant: 'READ',  to: 'Viewer' },
    { grant: 'WRITE', to: 'Editor' },
    { grant: '*',     to: 'Admin'  }
  ]
  entity Incidents as projection on db.Incidents;
}
```

### Instance-Based Authorization (row-level)

```cds
entity Incidents @(restrict: [
  { grant: '*', to: 'Admin' },
  { grant: 'READ', where: 'assignee = $user' }  // user can only read their own
]) as projection on db.Incidents;
```

### System-to-System (no user context)

```cds
service InternalService @(requires: 'system-user') {
  entity Incidents as projection on db.Incidents;
}
```

---

## Fiori UI Annotations (fiori.cds)

### List Report & Object Page

```cds
using IncidentService as service from '../srv/service';

annotate service.Incidents with @(
  UI.LineItem: [
    { Value: title,    Label: 'Title'    },
    { Value: status,   Label: 'Status'   },
    { Value: priority, Label: 'Priority' },
    { Value: customer.name, Label: 'Customer' }
  ],
  UI.SelectionFields: [ status, priority ],
  UI.HeaderInfo: {
    TypeName:       'Incident',
    TypeNamePlural: 'Incidents',
    Title:          { Value: title },
    Description:    { Value: descr }
  },
  UI.FieldGroup #Details: {
    Label: 'Details',
    Data: [
      { Value: title    },
      { Value: descr    },
      { Value: status   },
      { Value: priority }
    ]
  },
  UI.Facets: [
    { $Type: 'UI.ReferenceFacet', Label: 'Details', Target: '@UI.FieldGroup#Details' }
  ]
);
```

### Value Help

```cds
annotate service.Incidents with {
  status @(
    Common.ValueListWithFixedValues: true,
    Common.ValueList: {
      CollectionPath: 'Statuses',
      Parameters: [
        { $Type: 'Common.ValueListParameterOut', LocalDataProperty: status_code, ValueListProperty: 'code' },
        { $Type: 'Common.ValueListParameterDisplayOnly', ValueListProperty: 'name' }
      ]
    }
  );
}
```

### Mandatory & Read-Only Fields

```cds
annotate service.Incidents with {
  title  @mandatory;
  status @readonly;
}
```

---

## Common Patterns

### Many-to-Many Relationship

```cds
entity Incidents : cuid {
  title : String;
  tags  : Composition of many Incidents_Tags on tags.incident = $self;
}

entity Tags : cuid {
  name      : String;
  incidents : Composition of many Incidents_Tags on incidents.tag = $self;
}

entity Incidents_Tags {
  key incident : Association to Incidents;
  key tag      : Association to Tags;
}
```

### Localization (i18n)

```cds
entity Products : cuid {
  name  : localized String(100);   // generates _texts table automatically
  descr : localized String;
}
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `entity not found` | Wrong namespace or missing `using` import | Check namespace and `using { X } from '...'` path |
| `ambiguous association` | Two paths to same entity | Use explicit `on` condition |
| `draft not enabled` | Missing `@odata.draft.enabled` | Add annotation to the entity in service.cds |
| `403 Forbidden` | User missing required role | Check `@requires` / `@restrict` and XSUAA role collection assignment |
| `EDMX generation failed` | Invalid annotation syntax | Check Fiori annotation structure — missing `$Type` is common |
| `localized texts not returned` | Locale header missing | Send `Accept-Language` header in request |

---

## Graceful Failure Handling

### Missing context
> "To write a correct CDS model I need a few details:
> - **What entities** do you need? (e.g., Orders, Products, Customers)
> - **What relationships** exist between them? (e.g., an Order has many Items)
> - **What service** should expose them? (read-only, draft-enabled, secured?)
>
> Once you share these, I'll generate the full schema and service definition."

### Out of scope
> "This skill covers SAP CAP CDS models and service definitions. Your question about [topic] is outside that scope.
> Consider:
> - `btp-xs-security` skill — for XSUAA / xs-security.json
> - `btp-mta-descriptor` skill — for mta.yaml deployment config"

### Invalid CDS syntax
> "I found the following issues in your CDS:
> 1. [issue] — [cause and fix]
>
> Here is the corrected version: [corrected CDS]"

### Association vs Composition confusion
> "You've used `Association` here, but based on your requirement the correct choice is `Composition`.
> - Use **`Association`** when the child can exist independently (e.g., `Customer` linked to `Order` — customer exists without orders)
> - Use **`Composition`** when the child is owned by the parent and cannot exist alone (e.g., `OrderItems` inside `Order` — items are deleted when the order is deleted)
>
> Here is the corrected model: [corrected CDS]"

### Draft enabled but Fiori app not showing draft indicator
> "Draft handling requires both the CDS annotation AND the Fiori app to be configured correctly.
> Checklist:
> 1. `@odata.draft.enabled` must be on the entity in `service.cds` ✓
> 2. The Fiori app manifest must declare `'sap.app': { 'dataSources': { ... 'settings': { 'operationMode': 'auto' } } }`
> 3. The entity must have a `modifiedAt` field (from `managed` aspect) for draft conflict detection
> 4. The Fiori app must be built with the List Report or Object Page floorplan — draft is not supported in custom freestyle apps"

### @restrict not blocking unauthorized access
> "If `@restrict` is defined but users can still access data they shouldn't, check:
> 1. Is `@requires` set on the service level? Without it, unauthenticated users bypass all restrictions
> 2. Is the scope/role in `@restrict` spelled exactly as defined in `xs-security.json`?
> 3. Did the user log out and back in after the role collection was assigned? Roles are read at login time
> 4. Is the CAP app running with `cds.requires.auth: 'dummy'` in development? That bypasses auth — remove it for production"

### localized texts not returned
> "If `localized String` fields return only the default language value, the `Accept-Language` header is missing from the request.
> Fix: ensure your Approuter or HTTP client sends the header:
> ```
> Accept-Language: de
> ```
> CAP automatically queries the `_texts` companion table when this header is present. Without it, the default language value is returned."
