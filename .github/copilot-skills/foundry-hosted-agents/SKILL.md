---
name: foundry-hosted-agents
description: Create, test, deploy, and manage Microsoft Foundry Hosted Agents. Use when users ask about hosted agents, agent deployment, azd ai agent, az cognitiveservices agent, agent.yaml, containerized agents, or Foundry Agent Service.
---

# Foundry Hosted Agents

Containerized AI agents running on Foundry Agent Service, exposing HTTP endpoints with tools like web search, file search, code interpreter, MCP servers, and custom functions.

## Quick Reference

### Environment Variable Names (azd vs code)

| azd Environment Variable | Code Environment Variable | Notes |
|-------------------------|---------------------------|-------|
| `AZURE_AI_PROJECT_ENDPOINT` | `PROJECT_ENDPOINT` | agent.yaml maps between them |
| Model set via `{{placeholder}}` | `MODEL_DEPLOYMENT_NAME` | Resolved during `azd ai agent init` |

**CRITICAL**: In agent.yaml, use `${AZURE_AI_PROJECT_ENDPOINT}` (azd variable), which maps to `PROJECT_ENDPOINT` (code variable).

## Decision Tree: Choose Your Path

**Step 1**: Call `aitk-list_foundry_models` tool to check for existing infrastructure.

**Step 2**: ALWAYS ask the user which approach they prefer:

### If existing project is found:
Present both options to the user:
> "I found an existing Foundry project that you can use:
> - **Project**: `<project-name>`
> - **Endpoint**: `<endpoint>`
> - **Models**: `<list of deployed models>`
>
> Would you like to:
> 1. **Use this existing project** - Deploy your agent to the existing infrastructure (faster, uses `az cognitiveservices agent`)
> 2. **Create new infrastructure** - Set up a brand new Foundry project with its own resources (uses `azd ai agent`)
>
> Which would you prefer?"

### If no existing project is found:
Inform the user and proceed with new infrastructure:
> "No existing Foundry project was found. I'll help you create new infrastructure using `azd ai agent`."

**Step 3**: Follow the appropriate path based on user's choice:

| User Choice | Template | Deploy With |
|-------------|----------|-------------|
| Use existing project | Minimal | `az cognitiveservices agent` |
| Create new infrastructure | Full azd | `azd ai agent` |

---

## Path A: New Infrastructure (azd ai agent)

### CRITICAL: Always Use GitHub Samples

**NEVER manually create agent.yaml, main.py, Dockerfile, or requirements.txt files.** The `azd ai agent init` command expects a specific format that is difficult to replicate manually.

**ALWAYS use one of these approaches:**

#### Option 1: Use a GitHub Sample Directly (Recommended)
```bash
azd ai agent init -m https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/agent.yaml
```
This downloads all properly formatted files automatically.

#### Option 2: Fetch Sample as Reference Before Customizing
If the user needs a custom agent, first fetch a sample to use as a reference:
```bash
# Use fetch_webpage tool to get the sample files
# URLs to fetch:
# - https://raw.githubusercontent.com/microsoft-foundry/foundry-samples/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/agent.yaml
# - https://raw.githubusercontent.com/microsoft-foundry/foundry-samples/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/main.py
# - https://raw.githubusercontent.com/microsoft-foundry/foundry-samples/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/requirements.txt
# - https://raw.githubusercontent.com/microsoft-foundry/foundry-samples/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/Dockerfile
```

Then create custom files based on the fetched reference, ensuring the format matches exactly.

### Recommended Workflow

```bash
# Step 1: Initialize project with starter template
azd init -t https://github.com/Azure-Samples/azd-ai-starter-basic

# Step 2: Add agent sample (choose one)
azd ai agent init -m https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/agent.yaml

# Step 3: Provision infrastructure FIRST
# Creates: Foundry account, project, ACR, model deployments
# Populates: .azure/<env-name>/.env with AZURE_AI_PROJECT_ENDPOINT
azd provision

# Step 4: Set up local .env for testing (get values from provisioned environment)
# AZURE_AI_PROJECT_ENDPOINT → PROJECT_ENDPOINT (for local .env)
cat .azure/<env-name>/.env | grep AZURE_AI_PROJECT_ENDPOINT
# Create .env in project root with PROJECT_ENDPOINT=<value>

# Step 5: Test locally
source .venv/bin/activate  # or create: python -m venv .venv
pip install -r src/<agent-name>/requirements.txt
# Use single-terminal workaround (see Local Testing section)

# Step 6: Deploy after local testing passes
azd deploy <agent-name>

# Step 7: Test deployed agent (see Remote Testing section)
```

### CLI Quick Reference

```bash
azd init -t https://github.com/Azure-Samples/azd-ai-starter-basic  # Initialize
azd ai agent init -m <agent.yaml-url>                               # Add agent
azd provision                                                        # Create infrastructure
azd deploy <agent-name>                                              # Deploy agent only
azd up                                                               # Provision + deploy all
azd down                                                             # Clean up
```

### Available Agent Samples

| Sample | URL | Description |
|--------|-----|-------------|
| Local Tools | `.../agent-with-local-tools/agent.yaml` | Python functions as tools |
| Foundry Tools | `.../agent-with-foundry-tools/agent.yaml` | Web search, file search, code interpreter |
| Hello World | `.../agent-hello-world/agent.yaml` | Minimal starting point |

Base URL: `https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/`

---

## Path B: Existing Infrastructure (az cognitiveservices agent)

### Prerequisites
- Existing Foundry account and project
- Container Registry (ACR) connected to project
- Model deployments (e.g., gpt-4.1)

### Deploy Command

**IMPORTANT**: `az cognitiveservices agent` does NOT read agent.yaml. Pass env vars via `--env`.

```bash
# Find your ACR name
az acr list --resource-group <rg> --query "[].name" -o tsv

# Deploy (from directory containing Dockerfile)
az cognitiveservices agent create \
    --account-name <foundry-account> \
    --project-name <project> \
    --name <agent-name> \
    --source . \
    --registry <acr-name> \
    --env PROJECT_ENDPOINT=https://<account>.services.ai.azure.com/api/projects/<project> MODEL_DEPLOYMENT_NAME=gpt-4.1 \
    --show-logs
```

### Management Commands

```bash
az cognitiveservices agent list -a <account> -p <project>
az cognitiveservices agent status -a <account> -p <project> -n <name> --agent-version 1
az cognitiveservices agent logs show -a <account> -p <project> -n <name> --agent-version 1
az cognitiveservices agent stop -a <account> -p <project> -n <name> --agent-version 1
az cognitiveservices agent start -a <account> -p <project> -n <name> --agent-version 1
az cognitiveservices agent delete -a <account> -p <project> -n <name> --agent-version 1
```

---

## Project Structure

### Full azd Template

```
my-project/
├── azure.yaml                    # Service definitions (updated by azd ai agent init)
├── infra/                        # Bicep infrastructure (from starter template)
│   ├── main.bicep
│   ├── main.parameters.json
│   └── core/
├── src/
│   └── <agent-name>/             # Agent code (MUST be in src/ subdirectory)
│       ├── agent.yaml            # Agent manifest
│       ├── main.py               # Entry point
│       ├── Dockerfile
│       ├── requirements.txt
│       └── .dockerignore
├── .azure/
│   └── <env-name>/
│       └── .env                  # Provisioned values (AZURE_AI_PROJECT_ENDPOINT, etc.)
├── .env                          # Local dev values (PROJECT_ENDPOINT, MODEL_DEPLOYMENT_NAME)
├── .gitignore
└── test_deployed_agent.py        # Remote testing script
```

### Minimal Template (existing infrastructure)

```
my-agent/
├── main.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── .env                          # DO NOT commit
└── .gitignore
```

---

## Core Files

### agent.yaml

```yaml
kind: hosted
name: my-agent
description: >
  Description of what your agent does.

metadata:
  authors:
    - Your Name
  tags:
    - Azure AI AgentServer
    - Microsoft Agent Framework

protocols:
  - protocol: responses
    version: ""

environment_variables:
  - name: PROJECT_ENDPOINT
    value: ${AZURE_AI_PROJECT_ENDPOINT}    # Maps azd var → code var
  - name: MODEL_DEPLOYMENT_NAME
    value: "{{chat}}"                       # Placeholder resolved by azd ai agent init

resources:
  - kind: model
    id: gpt-4.1
    name: chat
```

### main.py

```python
import asyncio
import os
from dotenv import load_dotenv
from agent_framework.azure import AzureAIAgentClient
from azure.ai.agentserver.agentframework import from_agent_framework
from azure.identity.aio import DefaultAzureCredential

load_dotenv()
PROJECT_ENDPOINT = os.getenv("PROJECT_ENDPOINT")
MODEL_DEPLOYMENT_NAME = os.getenv("MODEL_DEPLOYMENT_NAME", "gpt-4.1")

async def main():
    async with (
        DefaultAzureCredential() as credential,
        AzureAIAgentClient(
            project_endpoint=PROJECT_ENDPOINT,
            model_deployment_name=MODEL_DEPLOYMENT_NAME,
            credential=credential,
        ) as client,
    ):
        agent = client.create_agent(
            name="MyAgent",
            instructions="You are a helpful assistant.",
            tools=[],
        )
        print("Agent running on http://localhost:8088")
        server = from_agent_framework(agent, credential)
        await server.run_async()

if __name__ == "__main__":
    asyncio.run(main())
```

### Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8088
CMD ["python", "main.py"]
```

### requirements.txt

```
python-dotenv
azure-identity
azure-ai-agentserver-agentframework
```

### .dockerignore

```
.venv/
venv/
.env
__pycache__/
*.py[cod]
.vscode/
.git/
*.md
.DS_Store
.pytest_cache/
```

### .env (local development)

```env
PROJECT_ENDPOINT=https://<account>.services.ai.azure.com/api/projects/<project>
MODEL_DEPLOYMENT_NAME=gpt-4.1
```

---

## Local Testing

### Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
az login
```

### Run Agent (separate terminals)

**Terminal 1** - Start agent:
```bash
python main.py
# Or for azd projects:
python src/<agent-name>/main.py
```

**Terminal 2** - Test:
```bash
curl -X POST http://localhost:8088/responses \
    -H "Content-Type: application/json" \
    -d '{"input": "Hello"}'
```

### Single-Terminal Workaround (for AI tools)

When you cannot use separate terminals, use this pattern:

```bash
# Kill any existing agent, start in background, wait, then test
pkill -f "python.*main.py" 2>/dev/null; sleep 1
python src/<agent-name>/main.py > /tmp/agent.log 2>&1 &
sleep 5
curl -s -X POST http://localhost:8088/responses -H "Content-Type: application/json" -d '{"input": "Hello"}'
```

Check logs if needed: `cat /tmp/agent.log`

---

## Remote Testing (Deployed Agents)

**CRITICAL**: Deployed agents use a DIFFERENT API than local testing.

### test_deployed_agent.py

```python
import os
from dotenv import load_dotenv
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

load_dotenv()
PROJECT_ENDPOINT = os.getenv("PROJECT_ENDPOINT")
AGENT_NAME = "<agent-name>"  # Must match name in agent.yaml

def test_deployed_agent():
    # Create project client
    project_client = AIProjectClient(
        endpoint=PROJECT_ENDPOINT,
        credential=DefaultAzureCredential(),
    )

    # Get OpenAI-compatible client - THIS IS THE KEY STEP
    openai_client = project_client.get_openai_client()

    # Create conversation for multi-turn
    conversation = openai_client.conversations.create()

    # Call hosted agent - MUST use extra_body
    response = openai_client.responses.create(
        conversation=conversation.id,
        extra_body={"agent": {"name": AGENT_NAME, "type": "agent_reference"}},
        input="Hello!",
        store=True,
    )
    print(response.output_text)

if __name__ == "__main__":
    test_deployed_agent()
```

**Dependencies**: `azure-ai-projects`, `azure-identity`, `python-dotenv`

### API Comparison

| Aspect | Local Testing | Deployed Testing |
|--------|--------------|------------------|
| Endpoint | `http://localhost:8088/responses` | Via `AIProjectClient` |
| Method | Direct HTTP (curl) | `openai_client.responses.create()` |
| Agent reference | Not needed | `extra_body={"agent": {...}}` required |
| Auth | DefaultAzureCredential (implicit) | DefaultAzureCredential (explicit) |

**Common mistakes**: Using `AgentsClient` (no `responses` attr), direct HTTP to deployed agent (auth complex), forgetting `extra_body`.

---

## Adding Tools

```python
from typing import Annotated

def get_weather(city: Annotated[str, "The city name"]) -> str:
    """Get current weather for a city."""
    return f"Weather in {city}: 72°F, sunny"

agent = client.create_agent(
    name="WeatherAgent",
    instructions="You help users check weather.",
    tools=[get_weather],
)
```

---

## Required Role Assignments

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

# Grant AcrPull
ACR_ID=$(az acr show --name <acr-name> --resource-group <resource-group> --query id -o tsv)
az role assignment create --assignee $PROJECT_IDENTITY --role "AcrPull" --scope $ACR_ID

# Grant Azure AI User (commonly missed!)
FOUNDRY_ID=$(az cognitiveservices account show --name <foundry-account> --resource-group <resource-group> --query id -o tsv)
az role assignment create --assignee $PROJECT_IDENTITY --role "Azure AI User" --scope $FOUNDRY_ID

# Grant yourself access for local dev
USER_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --assignee $USER_ID --role "Azure AI User" --scope $FOUNDRY_ID
```

---

## Troubleshooting

### Check Agent Logs

```bash
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version <version>
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `Azure AI project endpoint is required` | agent.yaml uses wrong variable | Use `${AZURE_AI_PROJECT_ENDPOINT}` not `${PROJECT_ENDPOINT}` |
| `PROJECT_ENDPOINT environment variable is required` | Missing `--env` in deploy | Add `--env PROJECT_ENDPOINT=... MODEL_DEPLOYMENT_NAME=...` |
| Agent `Failed`/`Unhealthy` status | Various | Check logs with command above |
| `AcrPullUnauthorized` | Missing role | Grant AcrPull to project's managed identity |
| `Model access denied` / 403 | Missing role | Grant Azure AI User to project's managed identity |
| `AuthenticationError` locally | Not logged in | Run `az login` |
| Port 8088 in use | Previous agent running | `lsof -ti:8088 | xargs kill -9` |
| `Invalid connection string` App Insights | Optional component | Agent works without it |

---

## Converting Existing Project to azd

If you have existing agent code and want to use `azd`:

### Step 1: Get infrastructure template

```bash
mkdir -p /tmp/azd-temp && cd /tmp/azd-temp
azd init -t https://github.com/Azure-Samples/azd-ai-starter-basic -e temp
cp -r infra azure.yaml /path/to/your/project/
```

### Step 2: Restructure for azd

Agent files MUST be in `src/<agent-name>/`:

```bash
cd /path/to/your/project
mkdir -p src/my-agent
mv main.py Dockerfile requirements.txt .dockerignore src/my-agent/
```

### Step 3: Create agent.yaml

See agent.yaml template in Core Files section. Use full format with `${AZURE_AI_PROJECT_ENDPOINT}`.

### Step 4: Configure for existing resources (optional)

```bash
azd env new my-env
azd env set AZURE_LOCATION <location>
azd env set AZURE_RESOURCE_GROUP <rg>
azd env set AZURE_AI_ACCOUNT_NAME <account>
azd env set AZURE_AI_PROJECT_NAME <project>
azd env set ENABLE_HOSTED_AGENTS true
```

### Step 5: Initialize and deploy

```bash
azd ai agent init -m ./src/my-agent/agent.yaml
azd provision  # or skip if using existing resources
azd deploy my-agent
```

---

## Resources

- [Quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
- [CLI Reference](https://learn.microsoft.com/en-us/cli/azure/cognitiveservices/agent)
- [Samples](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents)
