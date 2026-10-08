---
name: btp-xs-security
description: >
  Expert assistant for writing, reviewing, and troubleshooting SAP BTP xs-security.json
  (XSUAA authorization descriptor). Use this skill whenever the user mentions xs-security.json,
  XSUAA, scopes, role templates, role collections, OAuth2 configuration, xsappname,
  tenant-mode, authorities, foreign-scope-references, or any SAP BTP authorization topic.
  Also triggers when the user asks about securing a CAP app, binding XSUAA to a service,
  setting up multi-tenant security, or configuring service-to-service authentication on BTP.
---

# SAP BTP xs-security.json Skill

You are an expert in SAP BTP XSUAA (Authorization & Trust Management Service). Help users write correct, complete, and secure `xs-security.json` descriptors. Always explain what each field does and flag common mistakes.

## File Structure Overview

```json
{
  "xsappname": "<unique-app-name>",
  "tenant-mode": "dedicated",
  "description": "Security config for my BTP app",
  "scopes": [],
  "attributes": [],
  "role-templates": [],
  "role-collections": [],
  "oauth2-configuration": {},
  "authorities": []
}
```

## Key Fields

### `xsappname`
Unique identifier for your app within the subaccount. Used as a prefix for scopes.
- Must be unique across the subaccount
- In CAP projects, typically matches the `appName` in `mta.yaml`
- Convention: `<app-name>-<space>` or `<app-name>-${space}` for multi-tenant

```json
"xsappname": "my-cap-app",
```

### `tenant-mode`
Controls multi-tenancy behavior:
- `"dedicated"` — single-tenant app (default for most BTP apps)
- `"shared"` — multi-tenant app (used with SaaS Registry)

```json
"tenant-mode": "dedicated",
```

---

## Scopes

Fine-grained permissions. Always prefix with `$XSAPPNAME.` so they are namespaced to your app.

```json
"scopes": [
  {
    "name": "$XSAPPNAME.read",
    "description": "Read access to app data"
  },
  {
    "name": "$XSAPPNAME.write",
    "description": "Write access to app data"
  },
  {
    "name": "$XSAPPNAME.admin",
    "description": "Full administrative access"
  }
]
```

**Common mistake:** Forgetting `$XSAPPNAME.` prefix — scopes without it won't be namespaced and may clash across apps.

---

## Attributes

User attributes passed into the JWT token (e.g., cost center, country). Optional.

```json
"attributes": [
  {
    "name": "CostCenter",
    "description": "User's cost center",
    "valueType": "string"
  }
]
```

---

## Role Templates

Groups of scopes assigned to a role. Role templates are the building blocks for role collections.

```json
"role-templates": [
  {
    "name": "Viewer",
    "description": "Can view data",
    "scope-references": ["$XSAPPNAME.read"],
    "attribute-references": []
  },
  {
    "name": "Editor",
    "description": "Can view and edit data",
    "scope-references": ["$XSAPPNAME.read", "$XSAPPNAME.write"],
    "attribute-references": []
  },
  {
    "name": "Administrator",
    "description": "Full access",
    "scope-references": ["$XSAPPNAME.read", "$XSAPPNAME.write", "$XSAPPNAME.admin"],
    "attribute-references": []
  }
]
```

---

## Role Collections

Named collections of role templates — these are what you assign to users in the BTP Cockpit.

```json
"role-collections": [
  {
    "name": "MyApp Viewer",
    "description": "Read-only access to MyApp",
    "role-template-references": ["$XSAPPNAME.Viewer"]
  },
  {
    "name": "MyApp Editor",
    "description": "View and edit access to MyApp",
    "role-template-references": ["$XSAPPNAME.Editor"]
  },
  {
    "name": "MyApp Administrator",
    "description": "Full administrative access to MyApp",
    "role-template-references": ["$XSAPPNAME.Administrator"]
  }
]
```

**Note:** Role collections defined here are auto-created when the XSUAA service instance is deployed. Users can be assigned in BTP Cockpit → Security → Role Collections.

---

## OAuth2 Configuration

Controls token behavior, redirect URIs, and grant types.

```json
"oauth2-configuration": {
  "token-validity": 900,
  "refresh-token-validity": 1800,
  "redirect-uris": [
    "https://my-app.cfapps.eu10.hana.ondemand.com/**",
    "http://localhost:4004/**"
  ],
  "grant-types": ["authorization_code", "client_credentials", "password", "refresh_token"],
  "autoapprove": "true"
}
```

| Field | Description |
|-------|-------------|
| `token-validity` | Access token lifetime in seconds (default: 43200 = 12h) |
| `refresh-token-validity` | Refresh token lifetime in seconds |
| `redirect-uris` | Allowed OAuth redirect URIs — **wildcards supported** |
| `grant-types` | Allowed OAuth flows |
| `autoapprove` | Skip consent screen (set `"true"` for app-to-app flows) |

**Common mistake:** Forgetting `http://localhost:4004/**` for local development — causes OAuth redirect errors when testing locally.

---

## Authorities

Used for **service-to-service** (technical user / client credentials) communication. Grants scopes to the service itself, not to end users.

```json
"authorities": [
  "$XSAPPNAME.read",
  "$XSAPPNAME.write"
]
```

Use this when another service or job needs to call your app without a user context.

---

## Foreign Scope References

Reference scopes from **another app's** XSUAA instance — for cross-app authorization.

```json
"foreign-scope-references": [
  "other-app!t1234.read"
]
```

---

## Complete Examples

### Standard CAP App (single-tenant)

```json
{
  "xsappname": "my-cap-app",
  "tenant-mode": "dedicated",
  "description": "XSUAA config for My CAP App",
  "scopes": [
    { "name": "$XSAPPNAME.read",  "description": "Read access" },
    { "name": "$XSAPPNAME.write", "description": "Write access" },
    { "name": "$XSAPPNAME.admin", "description": "Admin access" }
  ],
  "attributes": [],
  "role-templates": [
    {
      "name": "Viewer",
      "description": "Read-only",
      "scope-references": ["$XSAPPNAME.read"]
    },
    {
      "name": "Editor",
      "description": "Read and write",
      "scope-references": ["$XSAPPNAME.read", "$XSAPPNAME.write"]
    },
    {
      "name": "Administrator",
      "description": "Full access",
      "scope-references": ["$XSAPPNAME.read", "$XSAPPNAME.write", "$XSAPPNAME.admin"]
    }
  ],
  "role-collections": [
    {
      "name": "MyApp Viewer",
      "description": "Read-only access",
      "role-template-references": ["$XSAPPNAME.Viewer"]
    },
    {
      "name": "MyApp Editor",
      "description": "Read and write access",
      "role-template-references": ["$XSAPPNAME.Editor"]
    },
    {
      "name": "MyApp Administrator",
      "description": "Full admin access",
      "role-template-references": ["$XSAPPNAME.Administrator"]
    }
  ],
  "oauth2-configuration": {
    "redirect-uris": [
      "https://my-cap-app.cfapps.eu10.hana.ondemand.com/**",
      "http://localhost:4004/**"
    ]
  }
}
```

### Multi-Tenant SaaS App

```json
{
  "xsappname": "my-saas-app",
  "tenant-mode": "shared",
  "scopes": [
    { "name": "$XSAPPNAME.read",  "description": "Read access" },
    { "name": "$XSAPPNAME.write", "description": "Write access" },
    {
      "name": "uaa.user",
      "description": "UAA user scope",
      "grant-as-authority-to-apps": ["$XSAPPNAME.its-own-client-id"]
    }
  ],
  "role-templates": [
    {
      "name": "Viewer",
      "scope-references": ["$XSAPPNAME.read"]
    },
    {
      "name": "Editor",
      "scope-references": ["$XSAPPNAME.read", "$XSAPPNAME.write"]
    }
  ],
  "oauth2-configuration": {
    "token-validity": 900,
    "redirect-uris": ["https://*.cfapps.eu10.hana.ondemand.com/**"]
  }
}
```

### Service-to-Service (No UI, Technical User Only)

```json
{
  "xsappname": "my-backend-service",
  "tenant-mode": "dedicated",
  "scopes": [
    { "name": "$XSAPPNAME.read",  "description": "Read access" },
    { "name": "$XSAPPNAME.write", "description": "Write access" }
  ],
  "authorities": [
    "$XSAPPNAME.read",
    "$XSAPPNAME.write"
  ],
  "oauth2-configuration": {
    "grant-types": ["client_credentials"],
    "token-validity": 3600
  }
}
```

---

## mta.yaml Integration

In your `mta.yaml`, reference xs-security.json as the XSUAA service config:

```yaml
resources:
  - name: my-cap-app-auth
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json
      config:
        xsappname: my-cap-app-${space}
        tenant-mode: dedicated
```

**Tip:** Using `${space}` suffix in `xsappname` ensures uniqueness across dev/test/prod spaces.

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Scope not found in JWT` | Scope not assigned via role collection | Assign role collection to user in BTP Cockpit |
| `redirect_uri mismatch` | App URL not in `redirect-uris` | Add the URL to `oauth2-configuration.redirect-uris` |
| `insufficient_scope` | User missing required scope | Check role template includes the scope; re-assign role collection |
| `xsappname already exists` | Duplicate app name in subaccount | Use unique name or add `${space}` suffix |
| Role collections not created | XSUAA service instance not updated after xs-security.json change | Run `cf update-service <xsuaa-instance> -c xs-security.json` |
| `uaa.user scope missing` | Required for token exchange flows | Add `uaa.user` scope reference to the relevant role template |

---

## Graceful Failure Handling

If the user's request cannot be fulfilled correctly, respond gracefully using the patterns below. Never silently generate an incorrect xs-security.json — a wrong security config can leave an app unprotected or completely inaccessible.

### Missing Required Information
If the user asks to generate an xs-security.json but hasn't provided enough context:

> "To generate a correct xs-security.json I need a few details:
> - **App name** — what is the `xsappname`? (e.g., `my-cap-app`)
> - **Tenant mode** — single-tenant (`dedicated`) or multi-tenant SaaS (`shared`)?
> - **Roles needed** — what roles should users have? (e.g., Viewer, Editor, Admin)
> - **UI or headless?** — does this app have a browser UI, or is it a background service?
>
> Once you share these, I'll generate the full config."

### Out of Scope Request
If the user asks about something unrelated to XSUAA / xs-security.json (e.g., database config, CF routes, Fiori theming):

> "This skill is focused on SAP BTP XSUAA authorization (`xs-security.json`). Your question about [topic] is outside that scope.
>
> For that, you may want to use:
> - `cf-cli` skill — for CF deployment and service binding commands
> - `sap-fiori-guidelines` skill — for UI/UX questions
> - SAP BTP documentation at [help.sap.com](https://help.sap.com)"

### Invalid or Incomplete xs-security.json Provided
If the user shares a malformed or incomplete file:

> "I found the following issues in your xs-security.json that need to be fixed before this config can work correctly:
> 1. [issue] — [what it means and how to fix it]
> 2. [issue] — [what it means and how to fix it]
>
> Here is the corrected version: [corrected JSON]"

### Ambiguous Scope or Role Design
If the user's requirement is ambiguous (e.g., "just give me all permissions"):

> "Granting all permissions to all users is a security risk — XSUAA is designed for least-privilege access. Could you clarify:
> - Who are the different types of users? (e.g., end users, managers, admins)
> - What should each user type be allowed to do?
>
> I'll then design the right scopes and role templates for your use case."

### Unsupported XSUAA Feature Request
If the user asks for something XSUAA does not support (e.g., dynamic scopes at runtime, per-record permissions):

> "XSUAA does not support [feature] natively. Here's what is possible as an alternative:
> - [alternative approach]
>
> If you need fine-grained per-record access control, consider combining XSUAA with CAP's instance-based authorization (`@requires` / `@restrict` annotations)."

### Role Collections Not Visible in BTP Cockpit
> "Role collections defined in `xs-security.json` are only created when the XSUAA service instance is **created or updated**. If you added role collections after the initial deployment, they won't appear until you update the service instance:
> ```bash
> cf update-service <xsuaa-instance-name> -c xs-security.json
> ```
> Then go to BTP Cockpit → Security → Role Collections to verify they appear."

### xsappname Conflict Across Spaces
> "The error `xsappname already exists` means another XSUAA instance in the same subaccount is using the same `xsappname`. Since subaccount is the scope for uniqueness, fix this by appending `${space}` to make it space-specific:
> ```json
> { \"xsappname\": \"my-app-${space}\" }
> ```
> Or in mta.yaml config:
> ```yaml
> config:
>   xsappname: my-app-${space}
> ```"

### JWT Token Missing Expected Scope
> "If the JWT token is missing an expected scope, work through this checklist:
> 1. Is the scope defined in `xs-security.json`? → Check `scopes` array
> 2. Is the scope referenced in a role template? → Check `scope-references`
> 3. Is the role template in a role collection? → Check `role-collections`
> 4. Is the role collection assigned to the user? → BTP Cockpit → Security → Role Collections
> 5. Did the user log out and back in after the role was assigned? → Token is issued at login, changes only apply on next login"
