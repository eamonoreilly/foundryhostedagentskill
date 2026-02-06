---
name: foundry-hosted-agents-deploy
description: "Deploy Foundry Hosted Agents to Azure. Use when users ask to deploy, publish, push, release, or ship hosted agents. Triggers include azd deploy, azd up, az cognitiveservices agent create, deploy to Azure, push agent, publish agent. USE FOR: azd deploy, azd up, deploy agent, push agent, publish agent, az cognitiveservices agent create, deploy to Azure, release agent, ship agent. DO NOT USE FOR: creating new projects (use foundry-hosted-agents-create), testing agents (use foundry-hosted-agents-test), fixing deployment errors (use foundry-hosted-agents-troubleshoot), learning basics (use foundry-hosted-agents-quickstart). INVOKES: run_in_terminal for az/azd commands, aitk-list_foundry_models to find ACR. FOR SINGLE OPERATIONS: run az cognitiveservices agent create directly if user has all parameters."
---

# Deploy Foundry Hosted Agents

Use this skill when users want to **deploy hosted agents to Azure**.

For creating agents, see the `foundry-hosted-agents-create` skill.
For testing agents, see the `foundry-hosted-agents-test` skill.
For troubleshooting, see the `foundry-hosted-agents-troubleshoot` skill.

---

## WHEN USER ASKS TO DEPLOY WITH AZD (New Infrastructure):

### Prerequisites

- Completed `azd init` and `azd ai agent init`
- Ran `azd provision` to create infrastructure
- Tested locally (recommended)

### Deploy Commands

```bash
# Deploy single agent
azd deploy <agent-name>

# Deploy all services
azd up

# Provision + deploy in one command (first time)
azd up
```

### Full Deployment Workflow

```bash
# 1. Provision infrastructure (creates Foundry account, project, ACR, models)
azd provision

# 2. Deploy the agent
azd deploy <agent-name>

# 3. REQUIRED: Check agent logs to verify healthy startup
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version <version>

# 4. Verify deployment status
az cognitiveservices agent status \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version <version>
```

### REQUIRED: Check Agent Logs After Deploy

**Always check the agent logs after deployment** - do not skip this step. The deployment may succeed but the agent container may have startup errors, missing environment variables, or configuration issues that only appear in the logs.

```bash
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version <version>
```

**What to look for:**
- ✅ `Application startup complete` - Agent started successfully
- ✅ `GET /readiness 200 OK` - Health checks passing
- ✅ `Uvicorn running on http://0.0.0.0:8088` - Server listening
- ❌ `Invalid connection string` - Check APPLICATIONINSIGHTS_CONNECTION_STRING
- ❌ `AuthenticationError` - Check Azure credentials/roles
- ❌ `Model not found` - Check MODEL_DEPLOYMENT_NAME

**If you skip this step**, issues like environment variable problems, authentication failures, or SDK errors may go unnoticed until the agent fails in production.

### Console Output During Deploy

After running `azd deploy` or `azd up`, also review the console output for errors:

- Container image build failures
- ACR push errors
- Missing or incorrect environment variables
- Role assignment problems
- Agent startup failures

If the console shows errors, see the `foundry-hosted-agents-troubleshoot` skill for resolution steps.

### azd CLI Quick Reference

```bash
azd init -t https://github.com/Azure-Samples/azd-ai-starter-basic  # Initialize
azd ai agent init -m <agent.yaml-url>                               # Add agent
azd provision                                                        # Create infrastructure
azd deploy <agent-name>                                              # Deploy agent only
azd up                                                               # Provision + deploy all
azd down                                                             # Clean up all resources
```

---

## WHEN USER ASKS TO DEPLOY TO EXISTING INFRASTRUCTURE:

Use `az cognitiveservices agent` when you already have a Foundry account, project, and ACR.

### Prerequisites

- Existing Foundry account and project
- Container Registry (ACR) connected to project
- Model deployments (e.g., gpt-4.1)
- Required role assignments (see below)

### Find Your ACR Name

```bash
az acr list --resource-group <rg> --query "[].name" -o tsv
```

### Deploy Command

**IMPORTANT**: `az cognitiveservices agent` does NOT read agent.yaml. Pass env vars via `--env`.

```bash
# Deploy from directory containing Dockerfile
az cognitiveservices agent create \
    --account-name <foundry-account> \
    --project-name <project> \
    --name <agent-name> \
    --source . \
    --registry <acr-name> \
    --env PROJECT_ENDPOINT=https://<account>.services.ai.azure.com/api/projects/<project> MODEL_DEPLOYMENT_NAME=gpt-4.1 \
    --show-logs
```

### Deploy with Application Insights

**First, check if project already has an AppInsights connection:**

```bash
# Check for AppInsights connection (if exists, connection string is auto-injected)
az rest --method GET \
    --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>/connections?api-version=2025-06-01" \
    --query "value[?properties.category=='AppInsights'].name" -o tsv
```

**If AppInsights connection exists:** Deploy normally - no extra env var needed:

```bash
# Connection string is auto-injected by the service
az cognitiveservices agent create \
    --account-name <foundry-account> \
    --project-name <project> \
    --name <agent-name> \
    --source . \
    --registry <acr-name> \
    --env PROJECT_ENDPOINT=https://<account>.services.ai.azure.com/api/projects/<project> MODEL_DEPLOYMENT_NAME=gpt-4.1 \
    --show-logs
```

**If NO AppInsights connection:** Pass the connection string manually:

```bash
# Get the connection string
APPINSIGHTS_CONN=$(az monitor app-insights component show \
    --app <app-insights-name> \
    --resource-group <resource-group> \
    --query connectionString -o tsv)

# Deploy with Application Insights
az cognitiveservices agent create \
    --account-name <foundry-account> \
    --project-name <project> \
    --name <agent-name> \
    --source . \
    --registry <acr-name> \
    --env PROJECT_ENDPOINT=https://<account>.services.ai.azure.com/api/projects/<project> MODEL_DEPLOYMENT_NAME=gpt-4.1 APPLICATIONINSIGHTS_CONNECTION_STRING="$APPINSIGHTS_CONN" \
    --show-logs
```

**What to look for in startup logs:**
- ✅ `Observability setup completed with provided exporters` - App Insights connected
- ⚠️ `APPLICATIONINSIGHTS_CONNECTION_STRING not set` - Telemetry disabled (add connection or pass env var)

### Deploy with All Options

```bash
az cognitiveservices agent create \
    --account-name <foundry-account> \
    --project-name <project> \
    --name <agent-name> \
    --source . \
    --registry <acr-name> \
    --env PROJECT_ENDPOINT=https://<account>.services.ai.azure.com/api/projects/<project> MODEL_DEPLOYMENT_NAME=gpt-4.1 \
    --show-logs \
    --resource-group <rg>  # Optional if account is unique
```

---

## WHEN USER ASKS ABOUT MANAGING DEPLOYED AGENTS:

### List Agents

```bash
az cognitiveservices agent list \
    --account-name <account> \
    --project-name <project>
```

### Check Agent Status

```bash
az cognitiveservices agent status \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

### View Agent Logs

```bash
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

### Stop Agent

```bash
az cognitiveservices agent stop \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

### Start Agent

```bash
az cognitiveservices agent start \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

### Delete Agent

```bash
az cognitiveservices agent delete \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

---

## WHEN USER ASKS ABOUT ROLE ASSIGNMENTS:

**CRITICAL**: Hosted agents run under the **project's managed identity**. If you create your own Foundry project (not via azd), you MUST grant these roles manually.

### Required Roles

| Role | Scope | Purpose |
|------|-------|---------|
| **AcrPull** | Container Registry | Pull agent container images |
| **Azure AI User** | Foundry Account | Access deployed models |

### Grant Roles Script

```bash
# Get project managed identity
PROJECT_IDENTITY=$(az cognitiveservices account project show \
    --name <foundry-account> \
    --resource-group <resource-group> \
    --project-name <project-name> \
    --query identity.principalId -o tsv)

# Grant AcrPull to project identity
ACR_ID=$(az acr show --name <acr-name> --resource-group <resource-group> --query id -o tsv)
az role assignment create \
    --assignee $PROJECT_IDENTITY \
    --role "AcrPull" \
    --scope $ACR_ID

# Grant Azure AI User to project identity (commonly missed!)
FOUNDRY_ID=$(az cognitiveservices account show --name <foundry-account> --resource-group <resource-group> --query id -o tsv)
az role assignment create \
    --assignee $PROJECT_IDENTITY \
    --role "Azure AI User" \
    --scope $FOUNDRY_ID

# Grant yourself access for local dev
USER_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create \
    --assignee $USER_ID \
    --role "Azure AI User" \
    --scope $FOUNDRY_ID
```

### Verify Role Assignments

```bash
# Check project identity roles
az role assignment list --assignee $PROJECT_IDENTITY --query "[].{Role:roleDefinitionName, Scope:scope}" -o table
```

---

## WHEN DEPLOYMENT FAILS:

See the `foundry-hosted-agents-troubleshoot` skill for detailed error resolution.

Quick checks:
```bash
# Check deployment logs
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1

# Check agent status
az cognitiveservices agent status \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

Common deployment issues:
- `AcrPullUnauthorized` → Grant AcrPull role to project identity
- `Model access denied` / 403 → Grant Azure AI User role to project identity
- Missing env vars → Add `--env PROJECT_ENDPOINT=... MODEL_DEPLOYMENT_NAME=...`

---

## NEXT STEPS

### After Deployment Succeeds

➡️ **Test your deployed agent using the SDK** (not curl!).

Deployed agents require a different testing approach than local testing:

```bash
# Install test dependencies
pip install azure-ai-projects azure-identity python-dotenv
```

Then use `AIProjectClient.get_openai_client()` with `extra_body={"agent": {...}}`.

See `foundry-hosted-agents-test` skill for the complete remote testing script.

### If Deployment Failed

1. Check the console output for errors
2. Run `az cognitiveservices agent logs show ...` for detailed logs
3. See `foundry-hosted-agents-troubleshoot` skill for error resolution

---

## WHEN USER ASKS ABOUT APPLICATION INSIGHTS:

### Step 1: Check if Project Has AppInsights Connection

Foundry projects can have an Application Insights connection configured. When this exists, the service **automatically injects** `APPLICATIONINSIGHTS_CONNECTION_STRING` at runtime - you don't need to pass it manually.

```bash
# Query project connections for AppInsights
az rest --method GET \
    --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>/connections?api-version=2025-06-01" \
    --query "value[?properties.category=='AppInsights'].{name:name, target:properties.target}" \
    -o table
```

**If AppInsights connection exists:**
- ✅ No action needed - `APPLICATIONINSIGHTS_CONNECTION_STRING` is auto-injected at runtime
- ✅ Deploy without passing the connection string manually
- ✅ Verify logs show: `Observability setup completed with provided exporters`

**If NO AppInsights connection:** Continue to Step 2.

---

### Step 2: Find Application Insights Resources

**First, check the same resource group:**

```bash
az resource list --resource-type "Microsoft.Insights/components" \
    --resource-group <resource-group> \
    --query "[].{name:name, id:id}" -o table
```

**If not found, search the entire subscription:**

```bash
az resource list --resource-type "Microsoft.Insights/components" \
    --query "[].{name:name, resourceGroup:resourceGroup, id:id}" -o table
```

**If no Application Insights exists anywhere:** Create one first (see Step 4).

---

### Step 3: Create AppInsights Connection on Project (RECOMMENDED)

Once you have an Application Insights resource, create a connection to enable auto-injection:

```bash
# Set variables
SUBSCRIPTION_ID="<subscription-id>"
RESOURCE_GROUP="<resource-group>"
ACCOUNT_NAME="<foundry-account>"
PROJECT_NAME="<project>"
APPINSIGHTS_NAME="<app-insights-name>"
CONNECTION_NAME="${APPINSIGHTS_NAME}-connection"

# Get App Insights resource ID and connection string
APPINSIGHTS_ID=$(az monitor app-insights component show \
    --app $APPINSIGHTS_NAME \
    --resource-group $RESOURCE_GROUP \
    --query id -o tsv)

CONN_STRING=$(az monitor app-insights component show \
    --app $APPINSIGHTS_NAME \
    --resource-group $RESOURCE_GROUP \
    --query connectionString -o tsv)

# Create JSON body file (avoids shell escaping issues)
cat > /tmp/appinsights-connection.json << EOF
{
    "properties": {
        "authType": "ApiKey",
        "category": "AppInsights",
        "credentials": {
            "key": "${CONN_STRING}"
        },
        "group": "ServicesAndApps",
        "isDefault": true,
        "metadata": {
            "ApiType": "Azure",
            "ResourceId": "${APPINSIGHTS_ID}"
        },
        "target": "${APPINSIGHTS_ID}"
    }
}
EOF

# Create the connection
az rest --method PUT \
    --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.CognitiveServices/accounts/${ACCOUNT_NAME}/projects/${PROJECT_NAME}/connections/${CONNECTION_NAME}?api-version=2025-06-01" \
    --body @/tmp/appinsights-connection.json

# Clean up
rm /tmp/appinsights-connection.json
```

**After creating connection:**
- Redeploy the agent (connection string will be auto-injected)
- Verify startup logs show: `Observability setup completed with provided exporters`

---

### Step 4: If No Application Insights Resource Exists - Create One

```bash
# Create Application Insights
az monitor app-insights component create \
    --app <app-insights-name> \
    --location <location> \
    --resource-group <resource-group> \
    --kind web \
    --application-type web

# Then create the connection (Step 3)
```

---

### Alternative: Pass Connection String Manually (Not Recommended)

If you prefer not to create a project connection, you can pass the connection string during deployment:

```bash
# Get the connection string
APPINSIGHTS_CONN=$(az monitor app-insights component show \
    --app <app-insights-name> \
    --resource-group <resource-group> \
    --query connectionString -o tsv)

# Deploy with connection string
az cognitiveservices agent create \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --source . \
    --registry <acr-name> \
    --env PROJECT_ENDPOINT=<endpoint> MODEL_DEPLOYMENT_NAME=gpt-4.1 APPLICATIONINSIGHTS_CONNECTION_STRING="$APPINSIGHTS_CONN" \
    --show-logs
```

**Note:** This approach requires passing the connection string on every deployment. Creating a project connection (Step 3) is preferred.

### Verify Logs Are Flowing After Deployment

```bash
# Check for agent traces (wait 2-5 min after deployment)
az monitor app-insights query \
    --app <app-insights-name> \
    --resource-group <resource-group> \
    --analytics-query 'traces | where timestamp > ago(10m) | where message has "CreateResponse" or message has "agent" | project timestamp, message | order by timestamp desc | take 10'

# Check telemetry types being captured
az monitor app-insights query \
    --app <app-insights-name> \
    --resource-group <resource-group> \
    --analytics-query 'union traces, requests, dependencies | where timestamp > ago(10m) | summarize count() by itemType'
```

---

## Resources

- [CLI Reference](https://learn.microsoft.com/en-us/cli/azure/cognitiveservices/agent)
- [Quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
