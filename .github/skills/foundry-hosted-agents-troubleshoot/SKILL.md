---
name: foundry-hosted-agents-troubleshoot
description: Troubleshoot Foundry Hosted Agent errors and issues. Use when users encounter errors, failures, problems, or unexpected behavior with hosted agents. Triggers include agent failed, agent unhealthy, AcrPullUnauthorized, 403 error, AuthenticationError, connection refused, logs, debug agent, agent not working, deployment failed.
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
**This is usually NOT a critical error.** The agent will work without App Insights.

### Solution (Optional)
If you want telemetry, set the connection string:

```bash
# Find your App Insights connection string
az monitor app-insights component show \
    --app <app-insights-name> \
    --resource-group <rg> \
    --query connectionString -o tsv

# Add to .env or --env during deployment
APPLICATIONINSIGHTS_CONNECTION_STRING=<connection-string>
```

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

### For Remote Testing Issues

- [ ] Agent status is "Running": `az cognitiveservices agent status ...`
- [ ] Using `AIProjectClient.get_openai_client()` (not `AgentsClient`)
- [ ] Including `extra_body={"agent": {...}}`
- [ ] Agent name matches agent.yaml exactly
- [ ] Azure CLI logged in: `az login`

---

## Resources

- [CLI Reference](https://learn.microsoft.com/en-us/cli/azure/cognitiveservices/agent)
- [Quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
