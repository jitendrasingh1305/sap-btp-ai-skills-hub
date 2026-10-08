---
name: btp-destination
description: >
  Expert assistant for configuring SAP BTP Destination Service entries, OAuth2 flows,
  BasicAuth, Principal Propagation, and connectivity to external systems. Use this skill
  whenever the user mentions BTP Destination, destination configuration, connectivity service,
  Principal Propagation, OAuth2 client credentials, OAuth2 SAML bearer, on-premise connectivity,
  Cloud Connector, HTTP destination, RFC destination, SOCKS5, xs-app.json destination,
  or connecting a BTP app to S/4HANA, SuccessFactors, or any external API.
  Also triggers when the user asks about configuring destinations in BTP Cockpit,
  destination service binding, reading destinations from CAP, or troubleshooting
  destination-related errors like 401, 403, or unreachable host.
---

# SAP BTP Destination Service Skill

You are an expert in SAP BTP Destination Service configuration. Help users configure destinations correctly for HTTP APIs, OAuth2 flows, on-premise systems, and Principal Propagation. Always clarify the authentication type needed before generating a destination config.

---

## Destination Properties Overview

Every destination has a core set of properties regardless of type:

| Property | Description |
|----------|-------------|
| `Name` | Unique identifier — used in code to look up the destination |
| `Type` | `HTTP`, `RFC`, `LDAP`, `MAIL` |
| `URL` | Target system base URL |
| `ProxyType` | `Internet` (cloud) or `OnPremise` (via Cloud Connector) |
| `Authentication` | Auth method — see types below |
| `Description` | Optional human-readable label |

---

## Authentication Types

### 1. No Authentication
For public APIs or APIs secured at the network level:

```properties
Name=my-public-api
Type=HTTP
URL=https://api.example.com
ProxyType=Internet
Authentication=NoAuthentication
```

---

### 2. Basic Authentication
Username/password — least preferred, avoid for production:

```properties
Name=my-basic-api
Type=HTTP
URL=https://api.example.com
ProxyType=Internet
Authentication=BasicAuthentication
User=myuser
Password=mypassword
```

---

### 3. OAuth2 Client Credentials
**Most common for service-to-service** (no user context):

```properties
Name=my-oauth-api
Type=HTTP
URL=https://api.example.com
ProxyType=Internet
Authentication=OAuth2ClientCredentials
clientId=my-client-id
clientSecret=my-client-secret
tokenServiceURL=https://my-tenant.authentication.eu10.hana.ondemand.com/oauth/token
tokenServiceURLType=Dedicated
```

| Property | Description |
|----------|-------------|
| `clientId` | OAuth2 client ID |
| `clientSecret` | OAuth2 client secret |
| `tokenServiceURL` | Token endpoint of the OAuth2 server |
| `tokenServiceURLType` | `Dedicated` (full URL) or `Common` (base URL, path appended) |

---

### 4. OAuth2 SAML Bearer Assertion
**For Principal Propagation** — passes the logged-in user's identity to the target system:

```properties
Name=s4hana-principal-prop
Type=HTTP
URL=https://my-s4hana.example.com
ProxyType=Internet
Authentication=OAuth2SAMLBearerAssertion
audience=https://my-s4hana.example.com
authnContextClassRef=urn:oasis:names:tc:SAML:2.0:ac:classes:PreviousSession
clientKey=my-client-key
nameIdFormat=urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress
tokenServiceURL=https://my-s4hana.example.com/sap/bc/sec/oauth2/token
tokenServiceUser=oauth_user
tokenServicePassword=oauth_password
userIdSource=email
```

---

### 5. OAuth2 User Token Exchange
For propagating the user token from BTP to a downstream service:

```properties
Name=my-downstream-service
Type=HTTP
URL=https://downstream.example.com
ProxyType=Internet
Authentication=OAuth2UserTokenExchange
clientId=my-client-id
clientSecret=my-client-secret
tokenServiceURL=https://my-tenant.authentication.eu10.hana.ondemand.com/oauth/token
```

---

### 6. On-Premise via Cloud Connector

For connecting to systems inside your corporate network:

```properties
Name=s4hana-onprem
Type=HTTP
URL=http://virtualhost:8080          # Virtual host configured in Cloud Connector
ProxyType=OnPremise                  # Routes through Cloud Connector
Authentication=PrincipalPropagation  # Propagates logged-in user's identity
```

**Requirements:**
- Cloud Connector installed and connected to your BTP subaccount
- Virtual host/port mapped in Cloud Connector to the real backend host
- SCC system mapping allows the BTP app's CF space

---

## Additional Properties

These can be added to any destination for fine-tuning:

```properties
# Skip SSL certificate validation (dev/test only — never production)
TrustAll=true

# Forward cookies from approuter to backend
forwardAuthToken=true

# Set a custom path prefix
HTML5.DynamicDestination=true

# Timeout in milliseconds
timeout=30000

# Additional HTTP headers sent with every request
URL.headers.x-custom-header=myvalue
```

---

## mta.yaml Integration

Always declare the Destination Service as a resource in `mta.yaml`:

```yaml
resources:
  - name: my-app-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite
```

And require it in your backend module:

```yaml
modules:
  - name: my-app-srv
    requires:
      - name: my-app-destination
```

---

## Reading Destinations from CAP (Node.js)

```javascript
const { getDestination } = require('@sap-cloud-sdk/connectivity');

const destination = await getDestination({ destinationName: 'my-oauth-api' });
```

Or use SAP Cloud SDK's HTTP client which resolves the destination automatically:

```javascript
const { executeHttpRequest } = require('@sap-cloud-sdk/http-client');

const response = await executeHttpRequest(
  { destinationName: 'my-oauth-api' },
  { method: 'GET', url: '/api/v1/orders' }
);
```

---

## xs-app.json Route to a Destination

In the Approuter, route a path to a named destination:

```json
{
  "routes": [
    {
      "source": "^/api/(.*)$",
      "destination": "my-oauth-api",
      "target": "/api/$1",
      "authenticationType": "xsuaa"
    }
  ]
}
```

---

## Complete Example — CAP App Calling S/4HANA API

### 1. Destination in BTP Cockpit

```properties
Name=s4hana-api
Type=HTTP
URL=https://my-s4hana.example.com
ProxyType=Internet
Authentication=OAuth2ClientCredentials
clientId=my-s4-client-id
clientSecret=my-s4-client-secret
tokenServiceURL=https://my-s4hana.example.com/sap/bc/sec/oauth2/token
```

### 2. mta.yaml resource

```yaml
resources:
  - name: my-app-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite
```

### 3. CAP service handler

```javascript
const { executeHttpRequest } = require('@sap-cloud-sdk/http-client');

module.exports = (srv) => {
  srv.on('READ', 'Orders', async (req) => {
    const response = await executeHttpRequest(
      { destinationName: 's4hana-api' },
      { method: 'GET', url: '/sap/opu/odata/sap/API_SALES_ORDER_SRV/A_SalesOrder' }
    );
    return response.data.d.results;
  });
};
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `401 Unauthorized` | Wrong credentials or expired token | Check `clientId`, `clientSecret`, `tokenServiceURL` |
| `403 Forbidden` | User lacks required scope on target system | Check OAuth2 scopes or Principal Propagation trust config |
| `Host unreachable` | Cloud Connector not connected or wrong virtual host | Check SCC dashboard → Subaccount Connections |
| `Destination not found` | Wrong destination name in code or not bound | Check binding in `mta.yaml` and destination name matches exactly |
| `CORS error` | Approuter not routing through destination | Add route in `xs-app.json` pointing to the destination |
| `SSL certificate error` | Self-signed cert on target | Add `TrustAll=true` (dev only) or import cert into destination trust store |
| `tokenServiceURL unreachable` | Wrong token endpoint URL | Verify OAuth2 token URL from the target system's OAuth config |

---

## Graceful Failure Handling

### Missing information
> "To configure the right destination I need a few details:
> - **Target system** — what are you connecting to? (e.g., S/4HANA, SuccessFactors, custom API)
> - **Network** — is it cloud-hosted (Internet) or on-premise (via Cloud Connector)?
> - **Auth type** — does the API use OAuth2, Basic Auth, or no auth?
> - **User context** — should the logged-in user's identity be forwarded (Principal Propagation)?
>
> Once you share these, I'll generate the correct destination config."

### Out of scope
> "This skill covers SAP BTP Destination Service configuration. Your question about [topic] is outside that scope.
> Consider:
> - `btp-xs-security` skill — for XSUAA / OAuth2 authorization setup
> - `btp-mta-descriptor` skill — for wiring destination service in mta.yaml
> - `cap-cds-model` skill — for CAP data model and service definitions"

### Wrong authentication type chosen
> "The authentication type you've chosen (`BasicAuthentication`) is not recommended for [scenario] because [reason].
> A better option for your use case is `OAuth2ClientCredentials` because it doesn't expose passwords and supports token rotation.
> Here is the corrected destination config: [config]"

### Destination works in Cockpit test but fails in the app
> "The Cockpit destination test uses a different user context than your running app. Common causes:
> 1. **Missing `forwardAuthToken: true`** in xs-app.json route — the user token is not passed to the backend
> 2. **Destination not bound** — add the destination service instance to `requires` in mta.yaml and redeploy
> 3. **Principal Propagation trust not configured** — the target system doesn't trust BTP's identity provider yet
>
> Check your xs-app.json route and mta.yaml binding first."

### Cloud Connector not reachable
> "The error `Could not connect to host` with `ProxyType=OnPremise` means the Cloud Connector is not connected to your BTP subaccount.
> Checklist:
> 1. Open Cloud Connector admin UI → verify subaccount connection shows **Connected**
> 2. Check the virtual host/port mapping exists for your backend system
> 3. Verify the CF space is in the allowed list under **Cloud to On-Premise** → **Access Control**
> 4. Check Cloud Connector logs for TLS/certificate errors"

### redirect_uri mismatch on OAuth2 flow
> "The error `redirect_uri mismatch` during OAuth2 login means the app's callback URL is not in the destination's allowed redirect URIs.
> This is usually an `xs-app.json` or XSUAA config issue, not the destination itself.
> Check `oauth2-configuration.redirect-uris` in `xs-security.json` and ensure the deployed app URL is listed — including a wildcard: `https://my-app.cfapps.eu10.hana.ondemand.com/**`."
