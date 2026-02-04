---
name: foundry-hosted-agents-create
description: Create and initialize new Foundry Hosted Agent projects. Use when users ask to create, scaffold, initialize, start, or set up a new hosted agent. Triggers include azd ai agent init, agent.yaml, new agent project, create hosted agent, azd init.
---

# Create Foundry Hosted Agents

Use this skill when users want to **create a new hosted agent project** from scratch.

For testing agents, see the `foundry-hosted-agents-test` skill.
For deploying agents, see the `foundry-hosted-agents-deploy` skill.
For troubleshooting, see the `foundry-hosted-agents-troubleshoot` skill.

---

## WHEN USER ASKS TO CREATE A NEW HOSTED AGENT:

### Step 1: Check for Existing Infrastructure

Call `aitk-list_foundry_models` tool to check for existing Foundry projects.

### Step 2: Ask User Which Approach They Prefer

**If existing project is found**, present both options:
> "I found an existing Foundry project:
> - **Project**: `<project-name>`
> - **Endpoint**: `<endpoint>`
> - **Models**: `<list of deployed models>`
>
> Would you like to:
> 1. **Use this existing project** - Deploy to existing infrastructure (faster, uses `az cognitiveservices agent`)
> 2. **Create new infrastructure** - Set up a brand new Foundry project (uses `azd ai agent`)
>
> Which would you prefer?"

**If no existing project is found**:
> "No existing Foundry project was found. I'll help you create new infrastructure using `azd ai agent`."

### Step 3: Follow the Appropriate Path

| User Choice | Template | Deploy With |
|-------------|----------|-------------|
| Use existing project | Minimal template | `az cognitiveservices agent` |
| Create new infrastructure | Full azd template | `azd ai agent` |

---

## Path A: New Infrastructure (azd ai agent)

### CRITICAL: Always Use GitHub Samples

**NEVER manually create agent.yaml, main.py, Dockerfile, or requirements.txt files.** The `azd ai agent init` command expects a specific format that is difficult to replicate manually.

**ALWAYS use one of these approaches:**

#### Option 1: Use a GitHub Sample Directly (Recommended)
```bash
azd ai agent init -m https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/agent.yaml
```

#### Option 2: Fetch Sample as Reference Before Customizing
If the user needs a custom agent, first fetch a sample to use as a reference:
```bash
# Use fetch_webpage tool to get the sample files:
# - https://raw.githubusercontent.com/microsoft-foundry/foundry-samples/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/agent.yaml
# - https://raw.githubusercontent.com/microsoft-foundry/foundry-samples/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/main.py
# - https://raw.githubusercontent.com/microsoft-foundry/foundry-samples/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/requirements.txt
# - https://raw.githubusercontent.com/microsoft-foundry/foundry-samples/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/Dockerfile
```

Then create custom files based on the fetched reference, ensuring the format matches exactly.

### Recommended Workflow for New Infrastructure

```bash
# Step 1: Initialize project with starter template
azd init -t https://github.com/Azure-Samples/azd-ai-starter-basic

# Step 2: Add agent sample (choose one)
azd ai agent init -m https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/agent.yaml

# Step 3: Provision infrastructure FIRST
azd provision

# Step 4: Set up local .env for testing
cat .azure/<env-name>/.env | grep AZURE_AI_PROJECT_ENDPOINT
# Create .env in project root with PROJECT_ENDPOINT=<value>

# Step 5: Test locally (see foundry-hosted-agents-test skill)

# Step 6: Deploy (see foundry-hosted-agents-deploy skill)
```

### Available Agent Samples

| Sample | URL | Description |
|--------|-----|-------------|
| Local Tools | `.../agent-with-local-tools/agent.yaml` | Python functions as tools |
| Foundry Tools | `.../agent-with-foundry-tools/agent.yaml` | Web search, file search, code interpreter |
| Hello World | `.../agent-hello-world/agent.yaml` | Minimal starting point |

Base URL: `https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/`

---

## Path B: Existing Infrastructure (Minimal Template)

When using existing Foundry infrastructure, create only these files:

### Minimal Project Structure

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

## WHEN USER ASKS ABOUT PROJECT STRUCTURE:

### Full azd Template Structure

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
└── .gitignore
```

---

## WHEN USER ASKS ABOUT CORE FILES:

### Environment Variable Names (azd vs code)

| azd Environment Variable | Code Environment Variable | Notes |
|-------------------------|---------------------------|-------|
| `AZURE_AI_PROJECT_ENDPOINT` | `PROJECT_ENDPOINT` | agent.yaml maps between them |
| Model set via `{{placeholder}}` | `MODEL_DEPLOYMENT_NAME` | Resolved during `azd ai agent init` |

**CRITICAL**: In agent.yaml, use `${AZURE_AI_PROJECT_ENDPOINT}` (azd variable), which maps to `PROJECT_ENDPOINT` (code variable).

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

## WHEN USER ASKS TO ADD TOOLS TO AN AGENT:

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

## WHEN USER ASKS TO CONVERT EXISTING PROJECT TO AZD:

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

See agent.yaml template above. Use full format with `${AZURE_AI_PROJECT_ENDPOINT}`.

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

## NEXT STEPS

After creating your agent project:

1. **Test locally first** - Run `python main.py` and test with `curl http://localhost:8088/responses`
2. **Then deploy** - Use `azd deploy <agent-name>` once local tests pass
3. **Test deployed agent** - Use the SDK approach (not curl) for remote testing

See `foundry-hosted-agents-test` skill for detailed testing guidance.

---

## Resources

- [Quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
- [Samples](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents)
