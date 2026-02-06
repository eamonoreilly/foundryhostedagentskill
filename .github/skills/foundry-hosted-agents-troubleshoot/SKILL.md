---
name: foundry-hosted-agents-troubleshoot
description: "Troubleshoot Foundry Hosted Agent errors and issues. Use when users encounter errors, failures, problems, or unexpected behavior with hosted agents. Triggers include agent failed, agent unhealthy, AcrPullUnauthorized, 403 error, AuthenticationError, connection refused, logs, debug agent, agent not working, deployment failed. USE FOR: agent failed, agent unhealthy, AcrPullUnauthorized, 403 error, AuthenticationError, connection refused, debug agent, agent not working, deployment failed, check logs, fix agent error. DO NOT USE FOR: creating new agents (use foundry-hosted-agents-create), deploying agents (use foundry-hosted-agents-deploy), normal testing (use foundry-hosted-agents-test), learning basics (use foundry-hosted-agents-quickstart). INVOKES: run_in_terminal for az cognitiveservices agent logs/status commands. FOR SINGLE OPERATIONS: run az cognitiveservices agent logs show directly for quick log checks."
---

# Troubleshoot Foundry Hosted Agents

Use this skill when users are experiencing **errors or issues** with hosted agents.

For creating agents, see the `foundry-hosted-agents-create` skill.
For testing agents, see the `foundry-hosted-agents-test` skill.
For deploying agents, see the `foundry-hosted-agents-deploy` skill.

---

## WHEN USER REPORTS AN ERROR - START HERE:

### Step 1: Check Agent Status

```bash
az cognitiveservices agent status \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

### Step 2: Check Agent Logs

```bash
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

### Step 3: Match Error to Solution Below

---

## WHEN USER SEES: "Azure AI project endpoint is required"

### Cause
agent.yaml is using the wrong environment variable name.

### Solution
In agent.yaml, use `${AZURE_AI_PROJECT_ENDPOINT}` (the azd variable), NOT `${PROJECT_ENDPOINT}`:

```yaml
environment_variables:
  - name: PROJECT_ENDPOINT
    value: ${AZURE_AI_PROJECT_ENDPOINT}    # ✓ Correct
    # value: ${PROJECT_ENDPOINT}           # ✗ Wrong
```

---

## WHEN USER SEES: "PROJECT_ENDPOINT environment variable is required"

### Cause
When using `az cognitiveservices agent create`, environment variables were not passed.

### Solution
Add `--env` flag with required variables:

```bash
az cognitiveservices agent create \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --source . \
    --registry <acr-name> \
    --env PROJECT_ENDPOINT=https://<account>.services.ai.azure.com/api/projects/<project> MODEL_DEPLOYMENT_NAME=gpt-4.1 \
    --show-logs
```

---

## WHEN USER SEES: "AcrPullUnauthorized" or Container Pull Errors

### Cause
The project's managed identity doesn't have permission to pull from the container registry.

### Solution
Grant AcrPull role:

```bash
# Get project managed identity
PROJECT_IDENTITY=$(az cognitiveservices account project show \
    --name <foundry-account> \
    --resource-group <resource-group> \
    --project-name <project-name> \
    --query identity.principalId -o tsv)

# Get ACR resource ID
ACR_ID=$(az acr show --name <acr-name> --resource-group <resource-group> --query id -o tsv)

# Grant AcrPull
az role assignment create \
    --assignee $PROJECT_IDENTITY \
    --role "AcrPull" \
    --scope $ACR_ID
```

---

## WHEN USER SEES: 403 Error, "Model access denied", or Authorization Errors

### Cause
The project's managed identity doesn't have the Azure AI User role on the Foundry account.

### Solution
Grant Azure AI User role:

```bash
# Get project managed identity
PROJECT_IDENTITY=$(az cognitiveservices account project show \
    --name <foundry-account> \
    --resource-group <resource-group> \
    --project-name <project-name> \
    --query identity.principalId -o tsv)

# Get Foundry account resource ID
FOUNDRY_ID=$(az cognitiveservices account show \
    --name <foundry-account> \
    --resource-group <resource-group> \
    --query id -o tsv)

# Grant Azure AI User
az role assignment create \
    --assignee $PROJECT_IDENTITY \
    --role "Azure AI User" \
    --scope $FOUNDRY_ID
```

---

## WHEN USER SEES: "AuthenticationError" During Local Testing

### Cause
User is not logged into Azure CLI.

### Solution
```bash
az login
az account show  # Verify you're logged in
```

If using a specific subscription:
```bash
az account set --subscription <subscription-id>
```

---

## WHEN USER SEES: Agent Status "Failed" or "Unhealthy"

### Diagnosis
Check the logs for specific error:

```bash
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

### Common Causes

| Log Message | Cause | Solution |
|-------------|-------|----------|
| `PROJECT_ENDPOINT is required` | Missing env var | Redeploy with `--env` flag |
| `Model not found` | Wrong model name | Check `MODEL_DEPLOYMENT_NAME` matches deployed model |
| `Import error` | Missing dependency | Add to requirements.txt and redeploy |
| `Connection refused` | Agent crashed on startup | Check main.py for errors |

### Restart Agent

```bash
az cognitiveservices agent stop \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1

az cognitiveservices agent start \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

---

## WHEN USER SEES: "Connection refused" or Port 8088 Issues (Local)

### Cause
Agent is not running, or port is blocked/in use.

### Solution

Check if port is in use:
```bash
lsof -i:8088
```

Kill existing process:
```bash
lsof -ti:8088 | xargs kill -9
```

Restart agent:
```bash
python main.py
# Or for azd projects:
python src/<agent-name>/main.py
```

---

## WHEN USER SEES: "Invalid connection string" for App Insights

### Cause
Application Insights connection string is not set or invalid.

### Impact
**This is usually NOT a critical error.** The agent will work without App Insights, but you lose valuable observability.

### Solution

**Step 1: Check if project has AppInsights connection (auto-injection)**

```bash
# If this returns a result, connection string should be auto-injected
az rest --method GET \
    --url "https://management.azure.com/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>/connections?api-version=2025-06-01" \
    --query "value[?properties.category=='AppInsights'].name" -o tsv
```

**If AppInsights connection exists:** The connection string should be auto-injected. Try redeploying the agent.

**If NO AppInsights connection:** Continue to find and connect Application Insights.

---

**Step 2: Find Application Insights resources**

```bash
# Check resource group first
az resource list --resource-type "Microsoft.Insights/components" \
    --resource-group <resource-group> \
    --query "[].{name:name, id:id}" -o table

# If not found, search entire subscription
az resource list --resource-type "Microsoft.Insights/components" \
    --query "[].{name:name, resourceGroup:resourceGroup, id:id}" -o table
```

---

**Step 3a: If App Insights exists - Create project connection (RECOMMENDED)**

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

# Redeploy agent (connection string will be auto-injected)
```

---

**Step 3b: If NO App Insights exists - Create one first**

```bash
az monitor app-insights component create \
    --app <app-insights-name> \
    --location <location> \
    --resource-group <resource-group> \
    --kind web \
    --application-type web

# Then create the connection (Step 3a)
```

---

**Step 4: Verify observability is working**

Check startup logs for: `Observability setup completed with provided exporters`

---

## WHEN USER SEES: Remote Test Not Working (No Response)

### Cause
Usually one of:
1. Wrong API being used
2. Missing `extra_body` parameter
3. Wrong agent name

### Solution
Use the correct API pattern for deployed agents:

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

project_client = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential(),
)

# Must use get_openai_client()
openai_client = project_client.get_openai_client()

# Must include extra_body
response = openai_client.responses.create(
    conversation=conversation.id,
    extra_body={"agent": {"name": "<agent-name>", "type": "agent_reference"}},  # Required!
    input="Hello!",
    store=True,
)
```

Common mistakes:
- Using `AgentsClient` instead of `AIProjectClient.get_openai_client()`
- Forgetting `extra_body={"agent": {...}}`
- Agent name doesn't match agent.yaml `name` field

---

## WHEN USER ASKS TO VERIFY ROLE ASSIGNMENTS:

### Check All Role Assignments for Project Identity

```bash
# Get project managed identity
PROJECT_IDENTITY=$(az cognitiveservices account project show \
    --name <foundry-account> \
    --resource-group <resource-group> \
    --project-name <project-name> \
    --query identity.principalId -o tsv)

# List all roles
az role assignment list \
    --assignee $PROJECT_IDENTITY \
    --query "[].{Role:roleDefinitionName, Scope:scope}" \
    -o table
```

### Expected Roles

| Role | Scope |
|------|-------|
| AcrPull | Container Registry |
| Azure AI User | Foundry Account |

---

## COMPLETE TROUBLESHOOTING CHECKLIST:

### For Local Testing Issues

- [ ] Azure CLI logged in: `az account show`
- [ ] `.env` file exists with `PROJECT_ENDPOINT` and `MODEL_DEPLOYMENT_NAME`
- [ ] Virtual environment activated: `source .venv/bin/activate`
- [ ] Dependencies installed: `pip install -r requirements.txt`
- [ ] No other process on port 8088: `lsof -i:8088`
- [ ] Agent started successfully: `python main.py`

### For Deployment Issues

- [ ] ACR connected to Foundry project
- [ ] AcrPull role granted to project identity
- [ ] Azure AI User role granted to project identity
- [ ] `--env` includes `PROJECT_ENDPOINT` and `MODEL_DEPLOYMENT_NAME`
- [ ] Model deployment exists and name matches
- [ ] Dockerfile and requirements.txt are correct
- [ ] (Optional) `APPLICATIONINSIGHTS_CONNECTION_STRING` included for observability

### For Remote Testing Issues

- [ ] Agent status is "Running": `az cognitiveservices agent status ...`
- [ ] Using `AIProjectClient.get_openai_client()` (not `AgentsClient`)
- [ ] Including `extra_body={"agent": {...}}`
- [ ] Agent name matches agent.yaml exactly
- [ ] Azure CLI logged in: `az login`

### For Observability Issues

- [ ] Application Insights exists: `az resource list --resource-type "Microsoft.Insights/components" --resource-group <rg>`
- [ ] Agent deployed with `APPLICATIONINSIGHTS_CONNECTION_STRING`
- [ ] Startup logs show: `Observability setup completed with provided exporters`
- [ ] Telemetry appearing: `az monitor app-insights query --app <name> --analytics-query 'traces | take 5'`

---

## WHEN USER ASKS TO DIAGNOSE WITH APPLICATION INSIGHTS:

### Query Agent Request Logs

```bash
az monitor app-insights query \
    --app <app-insights-name> \
    --resource-group <resource-group> \
    --analytics-query 'traces | where timestamp > ago(30m) | where message has "CreateResponse" or message has "Error" or message has "Exception" | project timestamp, message, severityLevel | order by timestamp desc | take 30' \
    -o json
```

### Query for Errors Only

```bash
az monitor app-insights query \
    --app <app-insights-name> \
    --resource-group <resource-group> \
    --analytics-query 'traces | where timestamp > ago(1h) | where severityLevel >= 3 | project timestamp, message | order by timestamp desc | take 50' \
    -o json
```

### Query Model Call Performance

```bash
az monitor app-insights query \
    --app <app-insights-name> \
    --resource-group <resource-group> \
    --analytics-query 'dependencies | where timestamp > ago(1h) | where name has "chat" | summarize avgDuration=avg(duration), count=count() by name' \
    -o json
```

### Query Failed Dependencies

```bash
az monitor app-insights query \
    --app <app-insights-name> \
    --resource-group <resource-group> \
    --analytics-query 'dependencies | where timestamp > ago(1h) | where success == false | project timestamp, name, duration, resultCode | order by timestamp desc' \
    -o json
```

---

## Resources

- [CLI Reference](https://learn.microsoft.com/en-us/cli/azure/cognitiveservices/agent)
- [Quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
