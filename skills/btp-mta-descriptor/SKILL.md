---
name: btp-mta-descriptor
description: >
  Expert assistant for writing, reviewing, and troubleshooting SAP BTP MTA deployment
  descriptors (mta.yaml). Use this skill whenever the user mentions mta.yaml, MTA,
  mbt build, cf deploy, multiapps, modules, resources, requires, provides, mta.yaml
  parameters, CAP deployment, BTP deployment descriptor, deploy config, mtaext,
  or any task involving building or deploying a Multi-Target Application on SAP BTP.
  Also triggers when the user asks about linking services to apps in mta.yaml,
  setting memory/disk quotas, environment variables, build parameters, or deployment
  strategies in the context of SAP BTP.
---

# SAP BTP MTA Descriptor Skill (mta.yaml)

You are an expert in SAP BTP Multi-Target Application (MTA) deployment descriptors. Help users write correct, complete `mta.yaml` files, troubleshoot deployment errors, and understand how modules, resources, and dependencies wire together. Always use MTA spec version `3.1` unless the user specifies otherwise.

## mta.yaml Structure Overview

```yaml
_schema-version: '3.1'
ID: my-app
version: 1.0.0
description: My BTP Application

modules:
  - name: my-app-srv        # CF app / Docker container
    ...

resources:
  - name: my-app-db         # BTP managed service
    ...
```

---

## Modules

Modules represent deployable units — CF apps, approuter, jobs, etc.

### CAP Node.js Backend (srv)

```yaml
modules:
  - name: my-app-srv
    type: nodejs
    path: gen/srv
    parameters:
      buildpack: nodejs_buildpack
      readiness-health-check-type: http
      readiness-health-check-http-endpoint: /health
    build-parameters:
      builder: npm
    requires:
      - name: my-app-db
      - name: my-app-auth
    provides:
      - name: srv-api
        properties:
          srv-url: '${default-url}'
```

### SAP Approuter (ui)

```yaml
  - name: my-app
    type: approuter.nodejs
    path: app/router
    parameters:
      keep-existing-routes: true
      disk-quota: 256M
      memory: 256M
    requires:
      - name: srv-api
        group: destinations
        properties:
          name: srv-api
          url: '~{srv-url}'
          forwardAuthToken: true
      - name: my-app-auth
      - name: my-app-destination
      - name: my-app-html5-repo-runtime
```

### HTML5 App Deployer

```yaml
  - name: my-app-ui-deployer
    type: com.sap.application.content
    path: .
    requires:
      - name: my-app-html5-repo-host
        parameters:
          content-target: true
    build-parameters:
      build-result: resources
      requires:
        - artifacts:
            - app1.zip
          name: app1
          target-path: resources/
```

---

## Resources

Resources represent BTP managed services consumed by modules.

### XSUAA (Authorization)

```yaml
resources:
  - name: my-app-auth
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json
      config:
        xsappname: my-app-${space}
        tenant-mode: dedicated
```

### SAP HANA Cloud (HDI Container)

```yaml
  - name: my-app-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared
    properties:
      hdi-service-name: '${service-name}'
```

### Destination Service

```yaml
  - name: my-app-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite
```

### HTML5 Application Repository (Host)

```yaml
  - name: my-app-html5-repo-host
    type: org.cloudfoundry.managed-service
    parameters:
      service: html5-apps-repo
      service-plan: app-host
```

### HTML5 Application Repository (Runtime)

```yaml
  - name: my-app-html5-repo-runtime
    type: org.cloudfoundry.managed-service
    parameters:
      service: html5-apps-repo
      service-plan: app-runtime
```

### SAP AI Core

```yaml
  - name: my-app-aicore
    type: org.cloudfoundry.managed-service
    parameters:
      service: aicore
      service-plan: extended
```

---

## Build Parameters

Control how `mbt build` compiles each module.

```yaml
build-parameters:
  before-all:
    - builder: custom
      commands:
        - npm ci --production
        - npx cds build --production
```

| Parameter | Description |
|-----------|-------------|
| `builder` | `npm`, `maven`, `custom`, `grunt` |
| `build-result` | Output folder sent to CF (e.g., `gen/srv`) |
| `ignore` | Files/folders to exclude from the build archive |
| `requires` | Other modules whose build output this module needs |

---

## Parameters

Control CF runtime settings per module.

```yaml
parameters:
  memory: 512M
  disk-quota: 1G
  instances: 1
  buildpack: nodejs_buildpack
  keep-existing-routes: true
  no-start: false
  health-check-type: http
  health-check-http-endpoint: /health
  readiness-health-check-type: http
  readiness-health-check-http-endpoint: /health
```

---

## Environment Variables via Properties

Pass values between modules using `properties` and `~{}` references.

```yaml
modules:
  - name: my-app-srv
    provides:
      - name: srv-api
        properties:
          srv-url: '${default-url}'

  - name: my-app-router
    requires:
      - name: srv-api
        properties:
          SRV_URL: '~{srv-url}'     # consumed as env var SRV_URL
```

---

## MTA Extension Descriptor (mtaext)

Override values per environment without changing `mta.yaml`:

```yaml
# config-prod.mtaext
_schema-version: '3.1'
extends: my-app

modules:
  - name: my-app-srv
    parameters:
      memory: 1G
      instances: 2

resources:
  - name: my-app-auth
    parameters:
      config:
        xsappname: my-app-prod
```

Deploy with extension:
```bash
cf deploy mta_archives/my-app_1.0.0.mtar -e config-prod.mtaext
```

---

## Complete Example — Full CAP App

```yaml
_schema-version: '3.1'
ID: incident-management
version: 1.0.0
description: Incident Management CAP Application

build-parameters:
  before-all:
    - builder: custom
      commands:
        - npm ci --production
        - npx cds build --production

modules:
  - name: incident-management-srv
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
      - name: incident-management-db
      - name: incident-management-auth
    provides:
      - name: srv-api
        properties:
          srv-url: '${default-url}'

  - name: incident-management-db-deployer
    type: hdb
    path: gen/db
    parameters:
      buildpack: nodejs_buildpack
    requires:
      - name: incident-management-db

  - name: incident-management-router
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
      - name: incident-management-auth

resources:
  - name: incident-management-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  - name: incident-management-auth
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json
      config:
        xsappname: incident-management-${space}
        tenant-mode: dedicated
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `unknown command 'deploy'` | multiapps CF plugin not installed | `cf install-plugin multiapps -f` |
| `MTA ID already exists` | Previous failed deploy left partial state | `cf undeploy <mta-id>` then retry |
| `service plan not available` | Service not entitled in BTP subaccount | Check BTP Cockpit → Entitlements |
| `Module build failed` | Wrong node version or missing `package.json` | Check `engines.node` matches CF buildpack |
| `Could not find module path` | `path` in mta.yaml points to wrong folder | Verify `path` matches actual folder after `cds build` |
| `No space targeted` | CF not logged in or no space set | `cf target -o ORG -s SPACE` |
| `HDI container creation failed` | HANA Cloud instance not running | Start HANA Cloud instance in BTP Cockpit |

---

## Graceful Failure Handling

### Missing app name or structure
> "To generate a correct mta.yaml I need a few details:
> - **App ID** — what is the MTA ID? (e.g., `incident-management`)
> - **Modules** — what components does your app have? (e.g., CAP backend, Approuter, DB deployer)
> - **Services** — which BTP services does it use? (e.g., HANA, XSUAA, Destination, AI Core)
>
> Once you share these, I'll generate the full mta.yaml."

### Out of scope
> "This skill covers `mta.yaml` descriptors for SAP BTP MTA deployments. Your question about [topic] is outside that scope.
> For that, consider:
> - `btp-xs-security` skill — for xs-security.json / XSUAA config
> - `cf-cli` skill — for CF deployment commands"

### Invalid mta.yaml provided
> "I found the following issues in your mta.yaml:
> 1. [issue] — [cause and fix]
> 2. [issue] — [cause and fix]
>
> Here is the corrected version: [corrected YAML]"

### Build fails — gen/srv not found
> "The error `Could not find module path: gen/srv` means `cds build --production` was not run before `mbt build`, or the output folder doesn't match the `path` in mta.yaml.
> Fix:
> 1. Run `npx cds build --production` manually and verify `gen/srv` and `gen/db` are created
> 2. Ensure your `before-all` build step runs `cds build` before mbt packages the modules:
> ```yaml
> build-parameters:
>   before-all:
>     - builder: custom
>       commands:
>         - npm ci --production
>         - npx cds build --production
> ```"

### Service not entitled in subaccount
> "The error `service plan not available` during `cf deploy` means the BTP service declared in `resources` is not entitled in your subaccount.
> Fix:
> 1. Go to **BTP Cockpit** → **Entitlements** → **Edit**
> 2. Find the service (e.g., `hana`, `aicore`, `xsuaa`) and add the required plan
> 3. Save and retry `cf deploy`"

### Circular dependency between modules
> "A circular dependency in mta.yaml (Module A requires Module B, Module B requires Module A) will cause deployment to fail.
> Fix: identify which dependency is truly a runtime need vs. a configuration value. Use `provides/requires` only for values that must be resolved at deploy time. Move shared config to environment variables or a shared resource instead."

### MTA version mismatch
> "The error `Unsupported _schema-version` means the MTA schema version in your file is not supported by the multiapps CF plugin version installed.
> Fix: update the plugin to the latest version:
> ```bash
> cf install-plugin multiapps -f
> ```
> Then verify with `cf plugins | grep multiapps`. Always use `_schema-version: '3.1'`."
