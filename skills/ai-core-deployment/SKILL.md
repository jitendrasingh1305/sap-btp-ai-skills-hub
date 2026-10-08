---
name: ai-core-deployment
description: >
  Expert assistant for configuring and troubleshooting SAP AI Core deployments,
  serving templates, resource groups, and AI API configurations on SAP BTP.
  Use this skill whenever the user mentions SAP AI Core, AI API, serving template,
  resource group, model deployment, deployment configuration, aicore service binding,
  generative AI hub, foundation models, Claude on AI Core, GPT on AI Core,
  AI Core credentials, AI Core destination, prompt templates, orchestration service,
  or any task involving deploying, managing, or calling AI models via SAP AI Core.
  Also triggers when the user asks about setting up AI Core in a CAP app, reading
  AI Core credentials from environment, calling LLMs via AI Core API, or configuring
  the AI Core service in mta.yaml.
---

# SAP AI Core Deployment Skill

You are an expert in SAP AI Core — SAP's managed AI infrastructure on BTP. Help users configure deployments, serving templates, resource groups, and integrate AI Core into CAP and BTP applications. Always use the latest AI Core API and recommend the Generative AI Hub for LLM access.

---

## AI Core Concepts

| Concept | Description |
|---------|-------------|
| **Resource Group** | Isolated namespace for deployments, artifacts, and executions |
| **Serving Template** | YAML descriptor that defines how a model is deployed (what image, what resources) |
| **Deployment** | A running instance of a model — has a `deploymentUrl` for inference |
| **Configuration** | Named set of parameters used to create a deployment or execution |
| **Generative AI Hub** | SAP-managed access to foundation models (Claude, GPT-4, Gemini, etc.) — no serving template needed |
| **AI API** | REST API for managing AI Core resources |

---

## Service Setup

### mta.yaml — Add AI Core as a Resource

```yaml
resources:
  - name: my-app-aicore
    type: org.cloudfoundry.managed-service
    parameters:
      service: aicore
      service-plan: extended        # 'extended' required for Generative AI Hub & foundation models
```

Require it in your backend module:

```yaml
modules:
  - name: my-app-srv
    requires:
      - name: my-app-aicore
```

**Common mistake:** Using `service-plan: standard` — the `extended` plan is required for Generative AI Hub and foundation model access.

---

## AI Core Credentials

When bound to a CF app, credentials are injected via `VCAP_SERVICES`:

```json
{
  "aicore": [{
    "credentials": {
      "clientid": "sb-...",
      "clientsecret": "...",
      "url": "https://<tenant>.authentication.eu10.hana.ondemand.com",
      "serviceurls": {
        "AI_API_URL": "https://api.ai.prod.eu-central-1.aws.ml.hana.ondemand.com"
      }
    }
  }]
}
```

### Reading Credentials in CAP (Node.js)

```javascript
const xsenv = require('@sap/xsenv');

const services = xsenv.getServices({ aicore: { tag: 'aicore' } });
const { clientid, clientsecret, url, serviceurls } = services.aicore;
const aiApiUrl = serviceurls.AI_API_URL;
```

---

## Generative AI Hub — Calling Foundation Models

The Generative AI Hub provides access to Claude, GPT-4, Gemini, and other models without managing serving templates.

### Step 1 — Get an Access Token

```javascript
const axios = require('axios');

async function getToken(clientid, clientsecret, tokenUrl) {
  const response = await axios.post(
    `${tokenUrl}/oauth/token`,
    'grant_type=client_credentials',
    {
      auth: { username: clientid, password: clientsecret },
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
    }
  );
  return response.data.access_token;
}
```

### Step 2 — Call a Foundation Model

```javascript
async function callClaude(token, aiApiUrl, prompt) {
  const response = await axios.post(
    `${aiApiUrl}/v2/inference/deployments/<deployment-id>/invoke`,
    {
      anthropic_version: 'bedrock-2023-05-31',
      max_tokens: 1024,
      messages: [{ role: 'user', content: prompt }]
    },
    {
      headers: {
        Authorization: `Bearer ${token}`,
        'Content-Type': 'application/json',
        'AI-Resource-Group': 'default'
      }
    }
  );
  return response.data.content[0].text;
}
```

### Available Foundation Models (Generative AI Hub)

| Model | Model Name in AI Core |
|-------|-----------------------|
| Claude 3.5 Sonnet | `anthropic--claude-3-5-sonnet` |
| Claude 3 Opus | `anthropic--claude-3-opus` |
| GPT-4o | `azure-openai--gpt-4o` |
| GPT-4 Turbo | `azure-openai--gpt-4-turbo` |
| Gemini 1.5 Pro | `gemini-1.5-pro` |
| Llama 3 70B | `meta--llama3-70b-instruct` |

---

## Resource Groups

Resource groups isolate AI Core workloads. Use separate groups per environment (dev/staging/prod).

### Create a Resource Group (AI API)

```bash
curl -X POST \
  "${AI_API_URL}/v2/admin/resourceGroups" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{ "resourceGroupId": "my-app-dev" }'
```

### List Resource Groups

```bash
curl "${AI_API_URL}/v2/admin/resourceGroups" \
  -H "Authorization: Bearer ${TOKEN}"
```

---

## Serving Templates

Used when deploying **custom models** (not foundation models from Generative AI Hub).

```yaml
apiVersion: ai.sap.com/v1alpha1
kind: ServingTemplate
metadata:
  name: my-model-serving
  annotations:
    scenarios.ai.sap.com/description: "My custom model serving"
    scenarios.ai.sap.com/name: "my-model"
  labels:
    scenarios.ai.sap.com/id: "my-model-scenario"
    ai.sap.com/version: "1.0.0"
spec:
  inputs:
    parameters:
      - name: modelName
        type: string
  template:
    apiVersion: "serving.kserve.io/v1beta1"
    metadata:
      annotations:
        autoscaling.knative.dev/metric: concurrency
        autoscaling.knative.dev/target: "1"
        autoscaling.knative.dev/targetBurstCapacity: "0"
    spec:
      predictor:
        imagePullSecrets:
          - name: my-registry-secret
        containers:
          - name: kserve-container
            image: my-registry/my-model:1.0.0
            ports:
              - containerPort: 8080
                protocol: TCP
            resources:
              requests:
                cpu: "1"
                memory: "2Gi"
              limits:
                cpu: "2"
                memory: "4Gi"
```

---

## Deployment Lifecycle

### Create a Configuration

```bash
curl -X POST \
  "${AI_API_URL}/v2/lm/configurations" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -H "AI-Resource-Group: my-app-dev" \
  -d '{
    "name": "my-model-config",
    "scenarioId": "my-model-scenario",
    "executableId": "my-model-serving",
    "parameterBindings": [
      { "key": "modelName", "value": "my-model-v1" }
    ]
  }'
```

### Create a Deployment

```bash
curl -X POST \
  "${AI_API_URL}/v2/lm/deployments" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -H "AI-Resource-Group: my-app-dev" \
  -d '{ "configurationId": "<config-id>" }'
```

### Check Deployment Status

```bash
curl "${AI_API_URL}/v2/lm/deployments/<deployment-id>" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "AI-Resource-Group: my-app-dev"
```

Deployment status values: `PENDING` → `RUNNING` → `STOPPED` / `DEAD`

### Stop a Deployment

```bash
curl -X PATCH \
  "${AI_API_URL}/v2/lm/deployments/<deployment-id>" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -H "AI-Resource-Group: my-app-dev" \
  -d '{ "targetStatus": "STOPPED" }'
```

---

## AI Core Destination (BTP Destination Service)

To call AI Core from Approuter or external apps via BTP Destination:

```properties
Name=aicore-destination
Type=HTTP
URL=https://api.ai.prod.eu-central-1.aws.ml.hana.ondemand.com
ProxyType=Internet
Authentication=OAuth2ClientCredentials
clientId=<aicore-clientid>
clientSecret=<aicore-clientsecret>
tokenServiceURL=https://<tenant>.authentication.eu10.hana.ondemand.com/oauth/token
URL.headers.AI-Resource-Group=default
```

---

## Orchestration Service

The AI Core Orchestration Service lets you chain LLM calls, grounding, and filtering in a single API call:

```javascript
const response = await axios.post(
  `${aiApiUrl}/v2/inference/deployments/<orchestration-deployment-id>/invoke`,
  {
    orchestration_config: {
      module_configurations: {
        llm_module_config: {
          model_name: 'anthropic--claude-3-5-sonnet',
          model_params: { max_tokens: 1024, temperature: 0.7 }
        }
      }
    },
    input_params: {},
    messages_history: [],
    template: {
      messages: [
        { role: 'system', content: 'You are a helpful SAP assistant.' },
        { role: 'user',   content: '{{?user_query}}' }
      ]
    },
    input_params: { user_query: 'What is SAP BTP?' }
  },
  { headers: { Authorization: `Bearer ${token}`, 'AI-Resource-Group': 'default' } }
);
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `service plan 'extended' not available` | AI Core extended plan not entitled | Check BTP Cockpit → Entitlements → Add AI Core extended plan |
| `401 Unauthorized` | Expired or missing token | Re-fetch OAuth2 token — they expire in 12h |
| `404 Deployment not found` | Wrong deployment ID or resource group | Check `AI-Resource-Group` header matches the group the deployment is in |
| `Deployment status: DEAD` | Model container crashed | Check deployment logs via AI Core Launchpad |
| `Model not available` | Foundation model not enabled in subaccount | Enable model access in AI Core Generative AI Hub settings |
| `VCAP_SERVICES missing aicore` | AI Core not bound to the app | Add `my-app-aicore` to `requires` in mta.yaml and redeploy |
| `Resource group does not exist` | Group not created before deploying | Create resource group first via AI API |

---

## Graceful Failure Handling

### Missing context
> "To help with your AI Core setup I need a few details:
> - **Use case** — are you using Generative AI Hub (foundation models like Claude/GPT) or deploying a custom model?
> - **Environment** — is this for a CAP app on BTP, or a standalone service?
> - **Resource group** — do you have one set up, or do you need to create one?
>
> Once you share these, I'll generate the correct configuration."

### Out of scope
> "This skill covers SAP AI Core deployment and configuration. Your question about [topic] is outside that scope.
> Consider:
> - `btp-destination` skill — for connecting to AI Core via BTP Destination Service
> - `btp-mta-descriptor` skill — for adding AI Core service in mta.yaml
> - `btp-xs-security` skill — for XSUAA security setup"

### Wrong service plan
> "The `standard` plan does not include access to Generative AI Hub or foundation models.
> You need to change your mta.yaml to use `service-plan: extended`:
> ```yaml
> parameters:
>   service: aicore
>   service-plan: extended
> ```
> Also verify the `extended` plan is entitled in your BTP subaccount under Entitlements."

### Deployment stuck in PENDING
> "A deployment stuck in `PENDING` usually means one of:
> 1. **Resource quota exceeded** — your resource group has no more capacity. Check running deployments and stop unused ones: `PATCH /v2/lm/deployments/<id>` with `targetStatus: STOPPED`
> 2. **Invalid serving template** — the container image cannot be pulled. Check the image name and registry secret.
> 3. **Resource group not onboarded** — the resource group exists but has no AI Core scenario synced to it yet. Verify in AI Core Launchpad → Resource Groups.
>
> Check deployment logs in AI Core Launchpad → ML Operations → Deployments for the exact error."

### Calling the wrong deployment URL
> "The `deploymentUrl` returned by AI Core is not meant to be called directly from outside BTP — it requires a valid AI Core OAuth2 token in the `Authorization` header and the `AI-Resource-Group` header.
> Direct browser or Postman calls without these headers will return `401 Unauthorized`.
> Always call it via your CAP backend or use the AI Core Launchpad test console."

### Model not entitled in subaccount
> "The model `[model-name]` is not available because it has not been enabled in your subaccount's Generative AI Hub settings.
> To enable it:
> 1. Go to **AI Core Launchpad** → **Generative AI Hub** → **Models**
> 2. Find `[model-name]` and click **Enable**
> 3. Wait a few minutes for provisioning
>
> Note: Some models (e.g., Claude) may require additional entitlement approval from SAP. Check BTP Cockpit → Entitlements."

### Token expired mid-session
> "AI Core OAuth2 tokens expire after **12 hours**. If you are getting `401 Unauthorized` on calls that were working earlier, your token has expired.
> Fix: implement token refresh in your app — never cache a token indefinitely:
> ```javascript
> let tokenCache = { token: null, expiresAt: 0 };
>
> async function getValidToken(clientid, clientsecret, tokenUrl) {
>   if (Date.now() < tokenCache.expiresAt - 60000) return tokenCache.token;
>   const res = await fetchNewToken(clientid, clientsecret, tokenUrl);
>   tokenCache = { token: res.access_token, expiresAt: Date.now() + res.expires_in * 1000 };
>   return tokenCache.token;
> }
> ```"

### AI Core not bound to the app
> "The error `Cannot read properties of undefined (reading 'AI_API_URL')` means AI Core credentials are not available in `VCAP_SERVICES` — the service is not bound to your CF app.
> Fix: Add AI Core to `requires` in your module in `mta.yaml` and redeploy:
> ```yaml
> modules:
>   - name: my-app-srv
>     requires:
>       - name: my-app-aicore   # ← this was missing
> ```
> Then run `mbt build` and `cf deploy` to apply the change."
