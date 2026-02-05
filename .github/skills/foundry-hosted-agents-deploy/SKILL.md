---
name: foundry-hosted-agents-deploy
description: Deploy Foundry Hosted Agents to Azure. Use when users ask to deploy, publish, push, release, or ship hosted agents. Triggers include azd deploy, azd up, az cognitiveservices agent create, deploy to Azure, push agent, publish agent. USE FOR: azd deploy, azd up, deploy agent, push agent, publish agent, az cognitiveservices agent create, deploy to Azure, release agent, ship agent. DO NOT USE FOR: creating new projects (use foundry-hosted-agents-create), testing agents (use foundry-hosted-agents-test), fixing deployment errors (use foundry-hosted-agents-troubleshoot), learning basics (use foundry-hosted-agents-quickstart).
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

# 3. IMPORTANT: Check the console output for errors
# The deploy command prints logs - look for errors like:
# - Container build failures
# - Missing environment variables
# - Role assignment issues

# 4. Verify deployment status
az cognitiveservices agent status \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1

# 5. If status shows issues, check detailed logs
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

### CRITICAL: Always Check Console Output After Deploy

After running `azd deploy` or `azd up`, **always review the console output** for errors before assuming the deployment succeeded. Common issues that appear in the output:

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

## Resources

- [CLI Reference](https://learn.microsoft.com/en-us/cli/azure/cognitiveservices/agent)
- [Quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
