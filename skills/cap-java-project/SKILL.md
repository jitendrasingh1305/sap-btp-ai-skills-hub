---
name: cap-java-project
description: >
  Generates a complete, deployable SAP CAP Java project for Cloud Foundry on SAP BTP.
  Use this skill whenever the user asks to scaffold, generate, or create a full CAP Java
  project, a deployable BTP Java app, a Spring Boot CAP project, or a CAP Java CF starter.
  Also triggers when the user asks for a Java-based CAP project with mta.yaml, xs-security.json,
  CDS schema, service definition, Spring Boot application, and approuter — all wired together
  and ready to deploy with mbt build and cf deploy.
---

# CAP Java Project Generator

You are an expert SAP BTP Java architect. When asked to generate a CAP Java project, follow this skill exactly — collect requirements first, then generate every file in the correct order. The output must be a fully wired, deployable project that works with `mbt build` and `cf deploy` without any manual changes.

---

## Step 1 — Collect Requirements

Before generating any files, ask the user for the following. Do NOT generate anything until you have all answers:

```
To generate your complete CAP Java project I need a few details:

1. **App name** — short, lowercase, hyphenated (e.g., `incident-management`)
2. **Java package** — base package name (e.g., `com.example.incident` or `com.sap.demo`)
3. **Entities** — list the main data entities (e.g., Incidents, Customers, Comments)
4. **Roles** — what user roles are needed? (e.g., Viewer, Editor, Admin)
5. **BTP Services** — which services does the app need?
   - [ ] SAP HANA Cloud (HDI container) — for persistent data
   - [ ] Destination Service — for connecting to external systems
   - [ ] AI Core — for LLM / Generative AI Hub access
6. **CF Region** — where will it be deployed? (e.g., eu10, us10, ap10)
```

Once the user provides answers, proceed to Step 2.

---

## Step 2 — Generate Project Structure

Announce the structure before generating files:

```
I'll generate the following project structure for <app-name>:

<app-name>/
├── package.json              # Root — CDS CLI for HANA schema build
├── .cdsrc.json               # CDS build config — HANA task only
├── pom.xml                   # Root Maven POM (multi-module container)
├── .gitignore                # Java + Maven + Node ignores
├── xs-security.json          # XSUAA authorization descriptor
├── mta.yaml                  # MTA deployment descriptor
├── db/
│   └── schema.cds            # Data model — entities and relationships
├── srv/
│   ├── pom.xml               # Spring Boot + CAP Java dependencies (self-contained)
│   ├── <app-name>-service.cds  # OData service definition
│   └── src/main/
│       ├── java/<package>/
│       │   ├── Application.java           # Spring Boot main class
│       │   └── handlers/
│       │       └── <Entity>Handler.java   # Custom event handlers
│       └── resources/
│           └── application.yaml          # Spring Boot configuration
└── app/
    └── router/
        ├── package.json      # Approuter dependencies
        └── xs-app.json       # Approuter routing rules
```

---

## Step 3 — Generate Files (in this exact order)

Generate each file completely. Never truncate or use placeholders like `// add more here`.

---

### File 1: `package.json` (root — CDS CLI only)

```json
{
  "name": "<app-name>",
  "version": "1.0.0",
  "description": "Root package for CDS build tooling",
  "private": true,
  "devDependencies": {
    "@sap/cds-dk": "^8"
  }
}
```

**Note:** This root `package.json` is only for the CDS CLI used by `mbt build` to generate HANA artifacts (`gen/db`). It does NOT contain the Java app dependencies — those are in `srv/pom.xml`.

---

### File 2: `.cdsrc.json`

```json
{
  "build": {
    "target": "gen",
    "tasks": [
      { "for": "hana", "src": "db", "options": { "model": ["db", "srv"] } }
    ]
  },
  "hana": {
    "deploy-format": "hdbtable"
  }
}
```

**Critical:** Include ONLY the `hana` build task — NOT `java`. The `cds-maven-plugin` in `srv/pom.xml` handles Java code generation during `mvn package`. If you include the `java` task, CDS copies sources to `gen/srv/` which is NOT what `mbt build` expects (it runs `mvn` from `path: srv`).

---

### File 3: `pom.xml` (root Maven POM — multi-module container only)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId><java-group-id></groupId>
  <artifactId><app-name></artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>pom</packaging>
  <name><App Name></name>

  <modules>
    <module>srv</module>
  </modules>
</project>
```

**Note:** The root pom.xml is a thin multi-module container only. It does NOT manage Spring Boot or CDS dependencies — `srv/pom.xml` handles all of that via its own parent (`spring-boot-starter-parent`).

---

### File 4: `.gitignore`

```
# Maven
target/
*.class
*.jar
*.war

# CDS build output
gen/
resources/

# MTA build output
mta_archives/

# Node (CDS CLI)
node_modules/
package-lock.json

# Environment
default-env.json
.env

# IDE
.idea/
.vscode/
*.iml
.classpath
.project
.settings/

# macOS
.DS_Store
```

---

### File 5: `xs-security.json`

Generate based on the roles the user provided:

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
      "http://localhost:8080/**"
    ]
  }
}
```

---

### File 6: `mta.yaml`

```yaml
_schema-version: '3.1'
ID: <app-name>
version: 1.0.0
description: <App Name> — SAP CAP Java Application on BTP CF

build-parameters:
  before-all:
    - builder: custom
      commands:
        - npm ci
        - npx cds build --production

modules:

  - name: <app-name>-srv
    type: java
    path: srv
    parameters:
      buildpack: java_buildpack
      memory: 1024M
      disk-quota: 1024M
      readiness-health-check-type: process
    properties:
      JBP_CONFIG_OPEN_JDK_JRE: "[jre: {version: 21.+}]"
    build-parameters:
      builder: custom
      commands:
        - mvn clean package -DskipTests
      build-result: target/*.jar
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

**Critical production notes:**
- `memory: 1024M` is the minimum for Java buildpack — 512M fails because JVM pre-allocates ~640M for code cache + metaspace + thread stacks, leaving no heap.
- `JBP_CONFIG_OPEN_JDK_JRE: "[jre: {version: 21.+}]"` is REQUIRED — `java_buildpack` defaults to Java 8 which cannot run Spring Boot 3.x (compiled for Java 17+).
- `readiness-health-check-type: process` — NOT `http`. Using `/actuator/health` causes the deploy to hang because Spring Boot reports HANA as DOWN until the db-deployer task completes (race condition).

---

### File 7: `db/schema.cds`

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

### File 8: `srv/<app-name>-service.cds`

Use scope names matching xs-security.json in `@restrict.to` (lowercase, no `$XSAPPNAME.` prefix):

```cds
using { <app.name> as db } from '../db/schema';

@path: '/<app-name>'
service <AppName>Service @(requires: 'authenticated-user') {

  @restrict: [
    { grant: 'READ',   to: ['read', 'write', 'admin'] },
    { grant: 'CREATE', to: ['write', 'admin'] },
    { grant: 'UPDATE', to: ['write', 'admin'] },
    { grant: 'DELETE', to: ['admin'] }
  ]
  entity <Entity1> as projection on db.<Entity1>;

  @restrict: [
    { grant: 'READ',  to: ['read', 'write', 'admin'] },
    { grant: 'WRITE', to: ['write', 'admin'] }
  ]
  entity <Entity2> as projection on db.<Entity2>;
}
```

**Note:** In CAP Java 3.4.0, the OData V4 servlet is always mounted at `/odata/v4`. The `@path: '/<app-name>'` annotation sets the service-specific sub-path, so the actual OData URL is `/odata/v4/<app-name>/$metadata`, `/odata/v4/<app-name>/Employees`, etc. This differs from CAP Node.js where `@path` is the full standalone path. The catch-all xs-app.json route forwards all paths to the backend, so no special routing is needed for the `/odata/v4/` prefix.

---

### File 9: `srv/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.1</version>
    <relativePath/>
  </parent>

  <groupId><java-group-id></groupId>
  <artifactId><app-name>-srv</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <properties>
    <java.version>21</java.version>
    <cds.services.version>3.4.0</cds.services.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.sap.cds</groupId>
        <artifactId>cds-services-bom</artifactId>
        <version>${cds.services.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- CAP Java Spring Boot starter -->
    <dependency>
      <groupId>com.sap.cds</groupId>
      <artifactId>cds-starter-spring-boot</artifactId>
    </dependency>

    <!-- OData V4 adapter — NOT included in cds-starter-spring-boot 3.4.0; must be explicit -->
    <dependency>
      <groupId>com.sap.cds</groupId>
      <artifactId>cds-adapter-odata-v4</artifactId>
    </dependency>

    <!-- CF / BTP auto-configuration: XSUAA + HANA bindings (includes cds-feature-hana) -->
    <dependency>
      <groupId>com.sap.cds</groupId>
      <artifactId>cds-starter-cloudfoundry</artifactId>
    </dependency>

    <!-- H2 for local development -->
    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <scope>runtime</scope>
    </dependency>

    <!-- Spring Boot Actuator for /actuator/health health check -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- Spring Boot executable JAR -->
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>

      <!-- CAP Java: generates Java entity classes from CDS model -->
      <plugin>
        <groupId>com.sap.cds</groupId>
        <artifactId>cds-maven-plugin</artifactId>
        <version>${cds.services.version}</version>
        <executions>
          <execution>
            <id>generate</id>
            <goals>
              <goal>generate</goal>
            </goals>
          </execution>
        </executions>
      </plugin>

      <!-- Java compiler -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>${java.version}</source>
          <target>${java.version}</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

**Critical production notes:**
- `srv/pom.xml` MUST use `spring-boot-starter-parent` as its parent (not the root pom.xml). This gives Spring Boot BOM which manages `h2`, `spring-boot-starter-actuator`, and compiler versions.
- **`cds-adapter-odata-v4` MUST be added explicitly.** `cds-starter-spring-boot:3.4.0` does NOT include it in its transitive dependencies — without it, `CdsODataV4Servlet` is never registered and all OData requests return 404.
- `cds-starter-cloudfoundry` already includes `cds-feature-hana` transitively. Do NOT add `cds-adapter-hana` separately — it does not exist as a standalone artifact in CAP Java 3.4.0.
- The `cds-maven-plugin` generates Java entity classes in `src/gen/java/<service-package>/`. The generated package name is the service name in lowercase (e.g., `timesheetservice.*`), NOT `cds.gen.<service>.*`.

---

### File 10: `srv/src/main/resources/application.yaml`

```yaml
server:
  port: 8080

spring:
  config:
    activate:
      on-profile: default

  sql:
    init:
      mode: always

  security:
    user:
      name: admin
      password: admin

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always

cds:
  datasource:
    auto-config:
      enabled: false

---
spring:
  config:
    activate:
      on-profile: cloud

management:
  endpoints:
    web:
      exposure:
        include: health
```

**Notes:**
- Local profile (`default`): H2 in-memory + basic auth (admin/admin)
- Cloud profile (`cloud`): activated automatically on CF by `cds-starter-cloudfoundry` when `VCAP_SERVICES` is present — HANA and XSUAA are configured automatically via VCAP_SERVICES bindings

---

### File 11: `srv/src/main/java/<package>/Application.java`

```java
package <java.package>;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

### File 12: `srv/src/main/java/<package>/handlers/<Entity>Handler.java`

**Important:** The generated CDS entity classes live in package `<service-name-lowercase>.*` (e.g., `timesheetservice.*`), NOT `cds.gen.<service>.*`. Always check `srv/src/gen/java/` after first `mvn package` to confirm the actual package name.

```java
package <java.package>.handlers;

import com.sap.cds.services.handler.EventHandler;
import com.sap.cds.services.handler.annotations.Before;
import com.sap.cds.services.handler.annotations.ServiceName;
import com.sap.cds.services.cds.CdsCreateEventContext;

import <service-package>.<ServiceName>_;
import <service-package>.<Entity>_;

import org.springframework.stereotype.Component;

@Component
@ServiceName(<ServiceName>_.CDS_NAME)
public class <Entity>Handler implements EventHandler {

    @Before(event = "CREATE", entity = <Entity>_.CDS_NAME)
    public void beforeCreate(CdsCreateEventContext context) {
        context.getCqn().entries().forEach(entry -> {
            // Set defaults or validate before CREATE
            // example: entry.putIfAbsent("status", "open");
        });
    }
}
```

**Note:** The `<service-package>` is the service name in lowercase (e.g., for service `TimesheetService` → package `timesheetservice`).

---

### File 13: `app/router/package.json`

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

### File 14: `app/router/xs-app.json`

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

---

## Step 4 — Post-Generation Instructions

After generating all files, always provide these instructions:

```
## How to Deploy

### Prerequisites
- JDK 21+: java -version
- Maven 3.9+: mvn -version
- CF CLI v8: cf login -a https://api.cf.<region>.hana.ondemand.com --sso
- MBT build tool: npm install -g mbt
- Multiapps CF plugin: cf install-plugin multiapps -f
- HANA Cloud instance running in your BTP subaccount

### Local Development
cd <app-name>/srv
mvn spring-boot:run
# Opens at http://localhost:8080
# Uses H2 in-memory + basic auth (admin/admin)
# OData at: http://localhost:8080/<app-name>/<Entity>

### Deploy to CF
cd <app-name>
npm install       # generates package-lock.json
mbt build
cf deploy mta_archives/<app-name>_1.0.0.mtar

### After Deployment
1. BTP Cockpit → Security → Role Collections
2. Assign <AppName> Viewer / Editor / Admin to your user
3. Open the app: cf app <app-name>-router (shows the URL)
4. OData endpoint: https://<router-url>/<app-name>/<Entity>
```

---

## Graceful Failure Handling

### User gives vague requirement
> "To generate a complete, deployable CAP Java project I need specifics — please answer the 6 questions in Step 1."

### User asks for CAP Node.js instead
> "This skill generates CAP Java projects. For CAP Node.js, use the `cap-nodejs-project` skill instead."

### `mvn package` fails — cds-maven-plugin cannot find CDS model
> "The `cds-maven-plugin` runs from `srv/` and auto-discovers CDS files. Check:
> 1. `db/schema.cds` exists at the project root level (sibling of `srv/`)
> 2. `srv/<app-name>-service.cds` exists inside `srv/`
> 3. Run `mvn clean package -DskipTests` from `srv/` and check the full error
> 4. The generated classes land in `srv/src/gen/java/<service-package>/` — do NOT add `<baseDirectory>` to plugin config, it is not a valid parameter in 3.4.0"

### Compilation fails — cannot find generated CDS entity classes
> "The generated entity classes are in `srv/src/gen/java/<service-package>/` where `<service-package>` is the service name in lowercase (e.g., `TimesheetService` → `timesheetservice`).
> Import them as `import timesheetservice.Timesheets;` — NOT `import cds.gen.timesheetservice.Timesheets;`
> Run `mvn clean generate-sources` first if the gen folder is missing, then check what was generated."

### Deploy fails — JVM memory error (heap calculation)
> "Error: `There is insufficient memory remaining for heap. Memory available for allocation 512M is less than allocated memory 646152K`
> Fix: set `memory: 1024M` in mta.yaml for the Java srv module. Java buildpack pre-allocates ~640M for code cache + metaspace + thread stacks, leaving no room for heap at 512M."

### Deploy fails — UnsupportedClassVersionError (Java 8 vs Spring Boot 3.x)
> "Error: `class file version 61.0 ... this version of the Java Runtime only recognizes class file versions up to 52.0`
> Fix: Add `JBP_CONFIG_OPEN_JDK_JRE: \"[jre: {version: 21.+}]\"` to the srv module's `properties` in mta.yaml. The `java_buildpack` defaults to Java 8; Spring Boot 3.x requires Java 17+."

### Deploy hangs at 'Starting application' — readiness check fails
> "The `/actuator/health` endpoint returns DOWN because HANA tables don't exist yet — the db-deployer task hasn't run yet (MTA deploys srv and db-deployer in parallel, a race condition).
> Fix: use `readiness-health-check-type: process` (not `http`) in mta.yaml for the srv module. This lets the srv be marked ready as soon as the JVM process starts. The db-deployer will complete in the background and deploy the HANA schema."

### App returns 403 after login
> "403 means XSUAA authenticated but CDS rejected the request.
> 1. Verify role collection is assigned in BTP Cockpit → Security → Role Collections
> 2. Log out, clear cookies, log in fresh in incognito — JWT is cached
> 3. Check `@restrict.to` values match exact scope names from xs-security.json (e.g., `'read'` not `'Viewer'`)
> 4. To test without restrictions temporarily, remove `@restrict` and use only `requires: 'authenticated-user'` on the service"

### Deployment fails — org memory quota exceeded
> "Stop unused apps to free quota: `cf stop <app-name>`. The db-deployer task also needs free headroom even though it stops after HDI deploy completes. The Java srv needs 1024M — check free quota with `cf org <org-name>`."

### Login fails — redirect_uri does not match
> "Add both `*.eu10.hana.ondemand.com/**` AND `*.eu10-005.hana.ondemand.com/**` to xs-security.json redirect-uris. Rebuild (`mbt build`) and redeploy to update the XSUAA service."

### `/actuator/health` returns 503 — app starts but health check fails
> "Spring Boot Actuator reports DOWN because HANA Cloud instance is stopped (it auto-stops after inactivity).
> 1. Start HANA Cloud instance in BTP Cockpit → SAP HANA Cloud → Start
> 2. `cf logs <app-name>-srv --recent` to see connection errors
> 3. `cf restart <app-name>-srv` after HANA starts"

### `mbt build` fails — `npx cds build --production` error
> "The root `package.json` must have `@sap/cds-dk` as a devDependency. Check:
> 1. Run `npm install` first (generates package-lock.json), then `npm ci` works
> 2. `mta.yaml` `before-all` runs `npm ci` then `npx cds build --production`
> 3. Run `npx cds build --production` manually from the project root to see CDS errors"
