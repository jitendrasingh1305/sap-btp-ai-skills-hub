---
name: cap-nodejs-project
description: >
  Generates a complete, deployable SAP CAP Node.js project for Cloud Foundry on SAP BTP.
  Use this skill whenever the user asks to scaffold, generate, or create a full CAP Node.js
  project, a deployable BTP app, a CAP CF project from scratch, or a starter CAP application.
  Also triggers when the user asks for a complete project with mta.yaml, xs-security.json,
  CDS schema, service definition, approuter, and package.json all wired together and ready
  to deploy with mbt build and cf deploy.
---

# CAP Node.js Project Generator

You are an expert SAP BTP architect. When asked to generate a CAP Node.js project, follow this skill exactly — collect requirements first, then generate every file in the correct order. The output must be a fully wired, deployable project that works with `mbt build` and `cf deploy` without any manual changes.

---

## Step 1 — Collect Requirements

Before generating any files, ask the user for the following. Do NOT generate anything until you have all answers:

```
To generate your complete CAP Node.js project I need a few details:

1. **App name** — short, lowercase, hyphenated (e.g., `incident-management`)
2. **Entities** — list the main data entities (e.g., Incidents, Customers, Comments)
3. **Roles** — what user roles are needed? (e.g., Viewer, Editor, Admin)
4. **BTP Services** — which services does the app need?
   - [ ] SAP HANA Cloud (HDI container) — for persistent data
   - [ ] Destination Service — for connecting to external systems
   - [ ] AI Core — for LLM / Generative AI Hub access
5. **CF Region** — where will it be deployed? (e.g., eu10, us10, ap10)
```

Once the user provides answers, proceed to Step 2.

---

## Step 2 — Generate Project Structure

Announce the structure before generating files:

```
I'll generate the following project structure for <app-name>:

<app-name>/
├── package.json             # Root dependencies and CDS config
├── .cdsrc.json              # CDS build and runtime configuration
├── .gitignore               # Standard CF + Node.js ignores
├── xs-security.json         # XSUAA authorization descriptor
├── mta.yaml                 # MTA deployment descriptor
├── db/
│   └── schema.cds           # Data model — entities and relationships
├── srv/
│   ├── service.cds          # OData service definition with authorization
│   └── service.js           # Custom event handlers
└── app/
    └── router/
        ├── package.json     # Approuter dependencies
        └── xs-app.json      # Approuter routing rules
```

---

## Step 3 — Generate Files (in this exact order)

Generate each file completely. Never truncate or use placeholders like `// add more here`.

---

### File 1: `package.json`

```json
{
  "name": "<app-name>",
  "version": "1.0.0",
  "description": "<App Name> — SAP CAP Node.js Application",
  "engines": {
    "node": ">=20"
  },
  "dependencies": {
    "@sap/cds": "^8",
    "@sap/xssec": "^4",
    "express": "^4",
    "hdb": "^0.19",
    "passport": "^0.7"
  },
  "devDependencies": {
    "@cap-js/sqlite": "^1",
    "@sap/cds-dk": "^8"
  },
  "scripts": {
    "start": "cds-serve",
    "dev": "cds watch",
    "build": "cds build --production"
  },
  "cds": {
    "requires": {
      "auth": {
        "[production]": {
          "kind": "xsuaa"
        },
        "[development]": {
          "kind": "mocked",
          "users": {
            "viewer@test.com": { "roles": ["Viewer"] },
            "editor@test.com": { "roles": ["Editor"] },
            "admin@test.com":  { "roles": ["Admin"]  }
          }
        }
      },
      "db": {
        "[production]": { "kind": "hana" },
        "[development]": { "kind": "sqlite", "credentials": { "url": ":memory:" } }
      }
    }
  }
}
```

**Notes:**
- `[production]` profile activates on CF (env `NODE_ENV=production`)
- `[development]` profile uses SQLite in-memory and mocked auth for local `cds watch`
- Add `"@sap-cloud-sdk/http-client": "^3"` if Destination Service is selected
- Add `"@sap/generative-ai-hub-sdk": "^1"` if AI Core is selected

---

### File 2: `.cdsrc.json`

```json
{
  "build": {
    "target": "gen",
    "tasks": [
      { "for": "hana",  "src": "db",  "options": { "model": ["db", "srv"] } },
      { "for": "nodejs","src": "srv", "options": { "model": ["db", "srv"] } }
    ]
  },
  "hana": {
    "deploy-format": "hdbtable"
  },
  "requires": {
    "db": {
      "[production]": { "kind": "hana" }
    }
  }
}
```

---

### File 3: `.gitignore`

```
# Dependencies
node_modules/
package-lock.json

# CDS build output
gen/

# SQLite local DB
*.db

# MTA build output
mta_archives/
resources/

# Environment
default-env.json
.env

# macOS
.DS_Store

# Editor
.vscode/
.idea/
```

---

### File 4: `xs-security.json`

Generate based on the roles the user provided. Template:

```json
{
  "xsappname": "<app-name>-${space}",
  "tenant-mode": "dedicated",
  "description": "XSUAA security config for <App Name>",
  "scopes": [
    { "name": "$XSAPPNAME.read",  "description": "Read access"  },
    { "name": "$XSAPPNAME.write", "description": "Write access" },
    { "name": "$XSAPPNAME.admin", "description": "Admin access" }
  ],
  "attributes": [],
  "role-templates": [
    {
      "name": "Viewer",
      "description": "Read-only access",
      "scope-references": ["$XSAPPNAME.read"]
    },
    {
      "name": "Editor",
      "description": "Read and write access",
      "scope-references": ["$XSAPPNAME.read", "$XSAPPNAME.write"]
    },
    {
      "name": "Admin",
      "description": "Full access",
      "scope-references": ["$XSAPPNAME.read", "$XSAPPNAME.write", "$XSAPPNAME.admin"]
    }
  ],
  "role-collections": [
    {
      "name": "<AppName> Viewer",
      "description": "Read-only access to <App Name>",
      "role-template-references": ["$XSAPPNAME.Viewer"]
    },
    {
      "name": "<AppName> Editor",
      "description": "Read and write access to <App Name>",
      "role-template-references": ["$XSAPPNAME.Editor"]
    },
    {
      "name": "<AppName> Admin",
      "description": "Full access to <App Name>",
      "role-template-references": ["$XSAPPNAME.Admin"]
    }
  ],
  "oauth2-configuration": {
    "redirect-uris": [
      "https://*.<cf-region>.hana.ondemand.com/**",
      "https://*.<cf-region>-005.hana.ondemand.com/**",
      "http://localhost:4004/**"
    ]
  }
}
```

Adjust scopes and roles to match exactly what the user specified.

---

### File 5: `mta.yaml`

Generate based on selected services. Always include XSUAA. Add HANA, Destination, AI Core if selected:

```yaml
_schema-version: '3.1'
ID: <app-name>
version: 1.0.0
description: <App Name> — SAP CAP Node.js Application on BTP CF

build-parameters:
  before-all:
    - builder: custom
      commands:
        - npm ci --production
        - npx cds build --production

modules:

  - name: <app-name>-srv
    type: nodejs
    path: gen/srv
    parameters:
      buildpack: nodejs_buildpack
      readiness-health-check-type: http
      readiness-health-check-http-endpoint: /health
    build-parameters:
      builder: npm
      ignore: [node_modules/]
    requires:
      - name: <app-name>-auth
      - name: <app-name>-db          # remove if HANA not selected
      - name: <app-name>-destination  # remove if Destination not selected
      - name: <app-name>-aicore       # remove if AI Core not selected
    provides:
      - name: srv-api
        properties:
          srv-url: '${default-url}'

  - name: <app-name>-db-deployer
    type: hdb
    path: gen/db
    parameters:
      buildpack: nodejs_buildpack
      memory: 256M
      disk-quota: 512M
    requires:
      - name: <app-name>-db           # remove if HANA not selected

  - name: <app-name>-router
    type: approuter.nodejs
    path: app/router
    parameters:
      memory: 256M
      disk-quota: 256M
      keep-existing-routes: true
    requires:
      - name: srv-api
        group: destinations
        properties:
          name: srv-api
          url: '~{srv-url}'
          forwardAuthToken: true
      - name: <app-name>-auth

resources:

  - name: <app-name>-auth
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json
      config:
        xsappname: <app-name>-${space}
        tenant-mode: dedicated

  # Include if HANA selected
  - name: <app-name>-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  # Include if Destination selected
  - name: <app-name>-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite

  # Include if AI Core selected
  - name: <app-name>-aicore
    type: org.cloudfoundry.managed-service
    parameters:
      service: aicore
      service-plan: extended
```

---

### File 6: `db/schema.cds`

Generate entities based on what the user specified. Always use `cuid` and `managed` aspects:

```cds
namespace <app.name>;
using { cuid, managed } from '@sap/cds/common';

entity <Entity1> : cuid, managed {
  // fields based on user requirement
}

entity <Entity2> : cuid, managed {
  // fields based on user requirement
}
```

---

### File 7: `srv/service.cds`

Expose entities with authentication at the service level. Use the **exact scope names** from `xs-security.json` (lowercase, without the `$XSAPPNAME.` prefix) in `@restrict.to`:

```cds
using { <app.name> as db } from '../db/schema';

service <AppName>Service @(path: '/<app-name>', requires: 'authenticated-user') {

  @restrict: [
    { grant: 'READ',  to: ['read', 'write', 'admin'] },
    { grant: 'WRITE', to: ['write', 'admin'] },
    { grant: '*',     to: 'admin' }
  ]
  entity <Entity1> as projection on db.<Entity1>;

  @restrict: [
    { grant: 'READ',  to: ['read', 'write', 'admin'] },
    { grant: 'WRITE', to: ['write', 'admin'] }
  ]
  entity <Entity2> as projection on db.<Entity2>;
}
```

**Critical:** The `@restrict.to` values (`'read'`, `'write'`, `'admin'`) must match the **scope names** in `xs-security.json` exactly (not role template names like `'Viewer'`). CDS calls `checkLocalScope(value)` against the XSUAA JWT — it strips the `$XSAPPNAME.` prefix, so scope `$XSAPPNAME.read` → check value `'read'`.

If you are not sure about the scope names, use only service-level auth and skip entity-level `@restrict`:
```cds
service <AppName>Service @(path: '/<app-name>', requires: 'authenticated-user') {
  entity <Entity1> as projection on db.<Entity1>;
  entity <Entity2> as projection on db.<Entity2>;
}
```

---

### File 8: `srv/service.js`

Minimal handler with lifecycle hooks — extend based on user requirement:

```javascript
const cds = require('@sap/cds');

module.exports = cds.service.impl(async function (srv) {

  const { <Entity1> } = srv.entities;

  // Before CREATE — set defaults or validate
  srv.before('CREATE', <Entity1>, async (req) => {
    // Example: set status default
    req.data.status ??= 'open';
  });

  // After READ — transform or enrich data
  srv.after('READ', <Entity1>, (data) => {
    // Example: add computed fields
  });

  // Custom action handler (if actions defined in service.cds)
  // srv.on('<actionName>', <Entity1>, async (req) => { ... });

});
```

---

### File 9: `app/router/package.json`

```json
{
  "name": "<app-name>-router",
  "version": "1.0.0",
  "description": "Approuter for <App Name>",
  "dependencies": {
    "@sap/approuter": "^16"
  },
  "scripts": {
    "start": "node node_modules/@sap/approuter/approuter.js"
  }
}
```

---

### File 10: `app/router/xs-app.json`

```json
{
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^/(.*)$",
      "destination": "srv-api",
      "target": "/$1",
      "authenticationType": "xsuaa",
      "csrfProtection": false
    }
  ]
}
```

**Why catch-all:** CDS mounts services at their `@path` value directly (e.g., `/<app-name>`), not under `/odata/v4/`. A catch-all route with `destination: srv-api` is valid and forwards all paths to the backend. Do NOT include a `welcomeFile` unless you have a static `index.html` in the router folder.

**Do NOT use** path-specific routes like `^/odata/(.*)$` — CDS 8 does not serve under `/odata/v4/` by default when `@path` is explicitly set on the service.

---

## Step 4 — Post-Generation Instructions

After generating all files, always provide these instructions:

```
## How to Deploy

### Prerequisites
- CF CLI v8 installed and logged in: `cf login -a https://api.cf.<region>.hana.ondemand.com --sso`
- MBT build tool: `npm install -g mbt`
- Multiapps CF plugin: `cf install-plugin multiapps -f`
- HANA Cloud instance running in your BTP subaccount

### Local Development
cd <app-name>
npm install
cds watch
# Opens at http://localhost:4004
# Uses SQLite in-memory + mocked auth (no BTP needed)

### Deploy to CF
mbt build
cf deploy mta_archives/<app-name>_1.0.0.mtar

### After Deployment
1. Go to BTP Cockpit → Security → Role Collections
2. Assign <AppName> Viewer / Editor / Admin to your user
3. Open the app URL from: cf app <app-name>-router
```

---

## Graceful Failure Handling

### User gives vague requirement
> "To generate a complete, deployable project I need a few specifics — otherwise the generated files won't match your actual use case. Please answer the 5 questions in Step 1 above."

### User asks for something outside CAP Node.js scope
> "This skill generates CAP Node.js projects for CF on BTP. For CAP Java projects, use the `cap-java-project` skill instead. For individual file generation (just mta.yaml, just xs-security.json), use the `btp-mta-descriptor` or `btp-xs-security` skills."

### User asks to add a feature after generation
> "To add [feature] to the generated project:
> - For data model changes → update `db/schema.cds`
> - For new endpoints or actions → update `srv/service.cds` and `srv/service.js`
> - For new BTP services → add resource to `mta.yaml` and require it in the srv module
> - For new roles → update both `xs-security.json` and `@restrict` annotations in `srv/service.cds`"

### Deployment fails — org memory quota exceeded
> "The BTP CF org has a shared memory quota. If staging or the db-deployer task fails with `memory_in_mb exceeds organization memory quota`:
> 1. Check how many apps are running: `cf apps`
> 2. Stop apps you own that are not needed: `cf stop <app-name>`
> 3. Retry the deploy. The db-deployer runs as a one-time task and needs free quota headroom even though it stops after deployment.
> 4. After deploy completes, restart any apps you stopped."

### Login fails — redirect_uri does not match
> "The XSUAA error `redirect_uri does not match the configuration` means the CF app domain is not in `xs-security.json`'s `redirect-uris`.
> BTP CF uses subregion-specific domains (e.g., `eu10-005`, `us10-001`). Always include both patterns:
> ```json
> 'https://*.<region>.hana.ondemand.com/**',
> 'https://*.<region>-005.hana.ondemand.com/**'
> ```
> After updating xs-security.json, rebuild the MTA (`mbt build`) and redeploy (`cf deploy`) — the XSUAA service must be updated with the new redirect URIs."

### App returns 403 Forbidden after successful login
> "403 after login means XSUAA authenticated the user but CDS rejected them at the `@restrict` check.
> **Step 1:** Verify the role collection is assigned to your user in BTP Cockpit → Security → Role Collections.
> **Step 2:** Log out and log in again in a fresh incognito window — the JWT is cached by the approuter and old tokens won't have new scopes.
> **Step 3:** Verify `@restrict.to` values match the scope names in xs-security.json exactly (without `$XSAPPNAME.` prefix). Example: scope `$XSAPPNAME.read` → `to: 'read'`.
> **Step 4:** If 403 persists, temporarily remove `@restrict` and use only `requires: 'authenticated-user'` at the service level to confirm the rest of the stack works, then add restrictions back."

### Cannot GET /<app-name> — service returns 404
> "CDS mounts the service at the `@path` value you defined in `service.cds`. Check:
> 1. What path is CDS using? Look at startup logs: `cf logs <app-name>-srv --recent` and search for `serving <ServiceName>` — it shows the actual path.
> 2. Verify `xs-app.json` routes ALL paths to `srv-api`. Use the catch-all route `^/(.*)$` with `destination: srv-api`.
> 3. Do NOT assume the path is `/odata/v4/<ServiceName>` — that only applies if no `@path` annotation is set."

### Deployment fails — general
> "If `mbt build` or `cf deploy` fails, check:
> 1. `cf target` — are you pointing to the right org and space?
> 2. BTP Cockpit → Entitlements — are HANA, XSUAA, and other services entitled?
> 3. HANA Cloud — is the instance running? (it stops automatically after inactivity)
> 4. Run `npx cds build --production` manually and check for CDS compile errors before `mbt build`"
