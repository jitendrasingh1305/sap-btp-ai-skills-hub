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
├── pom.xml                   # Root Maven POM (multi-module)
├── .gitignore                # Java + Maven + Node ignores
├── xs-security.json          # XSUAA authorization descriptor
├── mta.yaml                  # MTA deployment descriptor
├── db/
│   └── schema.cds            # Data model — entities and relationships
├── srv/
│   ├── pom.xml               # Spring Boot + CAP Java dependencies
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

### File 2: `pom.xml` (root Maven POM)

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

  <build>
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>com.sap.cds</groupId>
          <artifactId>cds-maven-plugin</artifactId>
          <version>${cds.services.version}</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```

**Note:** Check the latest `cds.services.version` at https://mvnrepository.com/artifact/com.sap.cds/cds-services-bom before deploying. Replace `<java-group-id>` with the user's base package (e.g., `com.example`).

---

### File 3: `.gitignore`

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

### File 4: `xs-security.json`

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

### File 5: `mta.yaml`

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
      memory: 512M
      disk-quota: 1024M
      readiness-health-check-type: http
      readiness-health-check-http-endpoint: /actuator/health
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

---

### File 6: `db/schema.cds`

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

### File 7: `srv/<app-name>-service.cds`

Use scope names matching xs-security.json in `@restrict.to` (lowercase, no `$XSAPPNAME.` prefix):

```cds
using { <app.name> as db } from '../db/schema';

@path: '/api/v1'
service <AppName>Service {

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

**Note:** `@path: '/api/v1'` sets the OData base path. The full URL becomes `/api/v1/<Entity>`. Adjust the path as needed. The xs-app.json catch-all route forwards all paths to the backend so any `@path` value works.

---

### File 8: `srv/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId><java-group-id></groupId>
    <artifactId><app-name></artifactId>
    <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId><app-name>-srv</artifactId>
  <packaging>jar</packaging>

  <dependencies>
    <!-- CAP Java Spring Boot starter -->
    <dependency>
      <groupId>com.sap.cds</groupId>
      <artifactId>cds-starter-spring-boot</artifactId>
    </dependency>

    <!-- CF / BTP auto-configuration: XSUAA + HANA bindings -->
    <dependency>
      <groupId>com.sap.cds</groupId>
      <artifactId>cds-starter-cloudfoundry</artifactId>
    </dependency>

    <!-- HANA adapter (production) -->
    <dependency>
      <groupId>com.sap.cds</groupId>
      <artifactId>cds-adapter-hana</artifactId>
      <scope>runtime</scope>
    </dependency>

    <!-- OData v4 adapter -->
    <dependency>
      <groupId>com.sap.cds</groupId>
      <artifactId>cds-adapter-odata-v4</artifactId>
      <scope>runtime</scope>
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
        <executions>
          <execution>
            <id>generate</id>
            <goals>
              <goal>generate</goal>
            </goals>
            <configuration>
              <!-- Point to parent directory so plugin finds db/ and srv/ CDS files -->
              <baseDirectory>${project.basedir}/..</baseDirectory>
            </configuration>
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

---

### File 9: `srv/src/main/resources/application.yaml`

```yaml
server:
  port: 8080

spring:
  config:
    activate:
      on-profile: default

  # H2 in-memory database for local development
  sql:
    init:
      mode: always

  security:
    # Basic auth for local testing — disable in production (handled by XSUAA)
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
      enabled: false   # disable CF auto-config in local profile

---
spring:
  config:
    activate:
      on-profile: cloud

# On BTP CF: HANA + XSUAA are auto-configured via VCAP_SERVICES binding
# cds-starter-cloudfoundry handles all connection setup automatically
management:
  endpoints:
    web:
      exposure:
        include: health
```

**Notes:**
- Local profile (`default`): H2 in-memory + basic auth (admin/admin)
- Cloud profile (`cloud`): activated automatically on CF by `cds-starter-cloudfoundry` — HANA and XSUAA are configured via VCAP_SERVICES
- The `cloud` Spring profile is activated by `cds-starter-cloudfoundry` when `VCAP_SERVICES` is present

---

### File 10: `srv/src/main/java/<package>/Application.java`

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

### File 11: `srv/src/main/java/<package>/handlers/<Entity>Handler.java`

Generate one handler per entity with common lifecycle hooks:

```java
package <java.package>.handlers;

import com.sap.cds.services.handler.EventHandler;
import com.sap.cds.services.handler.annotations.Before;
import com.sap.cds.services.handler.annotations.ServiceName;
import com.sap.cds.services.cds.CdsCreateEventContext;
import cds.gen.<service-package>.<Entity>_;
import org.springframework.stereotype.Component;

@Component
@ServiceName("<AppName>Service")
public class <Entity>Handler implements EventHandler {

    @Before(event = "CREATE", entity = <Entity>_.CDS_NAME)
    public void beforeCreate(CdsCreateEventContext context) {
        // Set defaults or validate before CREATE
        context.getCqn().entries().forEach(entry -> {
            // example: entry.putIfAbsent("status", "open");
        });
    }
}
```

**Note:** The `cds.gen.<service-package>` classes are auto-generated by `cds-maven-plugin` during `mvn package`. The exact package depends on the service name and CDS namespace.

---

### File 12: `app/router/package.json`

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

### File 13: `app/router/xs-app.json`

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
# OData at: http://localhost:8080/odata/v4/<AppName>Service

### Deploy to CF
cd <app-name>
mbt build
cf deploy mta_archives/<app-name>_1.0.0.mtar

### After Deployment
1. BTP Cockpit → Security → Role Collections
2. Assign <AppName> Viewer / Editor / Admin to your user
3. Open the app: cf app <app-name>-router (shows the URL)
4. OData endpoint: https://<router-url>/odata/v4/<AppName>Service
```

---

## Graceful Failure Handling

### User gives vague requirement
> "To generate a complete, deployable CAP Java project I need specifics — please answer the 6 questions in Step 1."

### User asks for CAP Node.js instead
> "This skill generates CAP Java projects. For CAP Node.js, use the `cap-nodejs-project` skill instead."

### `mvn package` fails — cds-maven-plugin cannot find CDS model
> "The `cds-maven-plugin` needs to find the CDS files relative to the `srv/pom.xml`. Check:
> 1. `db/schema.cds` exists at the project root level (not inside `srv/`)
> 2. `srv/<app-name>-service.cds` exists at the `srv/` folder level (alongside `srv/pom.xml`)
> 3. The plugin's `<baseDirectory>${project.basedir}/..</baseDirectory>` points to the correct parent folder
> 4. Run `mvn clean package -DskipTests` from inside `srv/` and check the full error"

### App returns 403 after login
> "403 means XSUAA authenticated but CDS rejected the request.
> 1. Verify role collection is assigned in BTP Cockpit → Security → Role Collections
> 2. Log out, clear cookies, log in fresh in incognito — JWT is cached
> 3. Check `@restrict.to` values match exact scope names from xs-security.json (e.g., `'read'` not `'Viewer'`)
> 4. To test without restrictions temporarily, remove `@restrict` and use only `@path` on the service"

### Deployment fails — org memory quota exceeded
> "Stop unused apps to free quota: `cf stop <app-name>`. The db-deployer task also needs free headroom even though it stops after HDI deploy completes."

### Login fails — redirect_uri does not match
> "Add both `*.eu10.hana.ondemand.com/**` AND `*.eu10-005.hana.ondemand.com/**` to xs-security.json redirect-uris. Rebuild (`mbt build`) and redeploy to update the XSUAA service."

### `/actuator/health` returns 503 — app starts but health check fails
> "The Spring Boot Actuator health endpoint is returning DOWN. Check:
> 1. HANA Cloud instance is running (it auto-stops after inactivity)
> 2. `cf logs <app-name>-srv --recent` for database connection errors
> 3. Verify `leave-management-db` HDI container service binding: `cf services`
> 4. Redeploy the db-deployer if HANA tables are missing: run `cf deploy` again"

### `mbt build` fails — `npx cds build --production` error
> "The root `package.json` must have `@sap/cds-dk` as a devDependency and `npm ci` must run before `cds build`. Check:
> 1. Root `package.json` exists with `@sap/cds-dk` dependency
> 2. `mta.yaml` `before-all` runs `npm ci` before `npx cds build --production`
> 3. Run `npx cds build --production` manually from the project root to see CDS errors"
