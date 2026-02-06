---
name: foundry-hosted-agents-quickstart
description: "Get started with Foundry Hosted Agents from scratch. Use when users are new to hosted agents, want a tutorial, or ask how to build their first agent. Triggers include get started, help me build, how do I, quickstart, tutorial, walkthrough, step by step, new to hosted agents, first agent, beginner, learn hosted agents. USE FOR: get started, quickstart, tutorial, walkthrough, step by step, new to hosted agents, first agent, beginner, learn hosted agents, how do I build an agent, help me build. DO NOT USE FOR: specific create operations (use foundry-hosted-agents-create), deployment details (use foundry-hosted-agents-deploy), testing specifics (use foundry-hosted-agents-test), error resolution (use foundry-hosted-agents-troubleshoot). INVOKES: foundry-hosted-agents-create, foundry-hosted-agents-test, foundry-hosted-agents-deploy skills as needed. FOR SINGLE OPERATIONS: use specialized skills directly if user knows what phase they need."
---

# Foundry Hosted Agents Quickstart

Use this skill when users are **new to hosted agents** or want a complete walkthrough.

This skill links to the specialized skills for each phase:
- `foundry-hosted-agents-create` - Detailed creation guidance
- `foundry-hosted-agents-test` - Local and remote testing
- `foundry-hosted-agents-deploy` - Deployment options
- `foundry-hosted-agents-troubleshoot` - Error resolution

---

## WHEN USER IS NEW TO HOSTED AGENTS:

### What is a Foundry Hosted Agent?

A **Foundry Hosted Agent** is a containerized AI agent that:
- Runs on Azure Foundry Agent Service
- Exposes an HTTP endpoint for the Responses API
- Can use tools like web search, file search, code interpreter, and custom Python functions
- Scales automatically and integrates with Azure AI models

### Complete Workflow Overview

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   CREATE    │ ──▶ │ TEST LOCAL  │ ──▶ │   DEPLOY    │ ──▶ │ TEST REMOTE │
│             │     │             │     │             │     │             │
│ azd init    │     │ python      │     │ azd deploy  │     │ SDK test    │
│ azd ai agent│     │ main.py     │     │ check logs  │     │ script      │
│ init        │     │ curl test   │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

---

## PHASE 1: CREATE YOUR AGENT PROJECT

### Prerequisites

```bash
# Required tools
az --version          # Azure CLI
azd version           # Azure Developer CLI
python --version      # Python 3.12+

# Login to Azure
az login
```

### Quick Start Commands

```bash
# Step 1: Initialize with starter template
azd init -t https://github.com/Azure-Samples/azd-ai-starter-basic

# Step 2: Add an agent sample
azd ai agent init -m https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/agent-with-local-tools/agent.yaml

# Step 3: Provision Azure infrastructure
azd provision
```

### What Gets Created

| Resource | Purpose |
|----------|---------|
| Foundry Account | Hosts your AI project |
| Foundry Project | Contains agents and models |
| Container Registry (ACR) | Stores agent container images |
| Model Deployment (gpt-4.1) | LLM for your agent |

**For detailed creation options**: See `foundry-hosted-agents-create` skill.

---

## PHASE 2: TEST LOCALLY

### Setup Local Environment

```bash
# Create and activate virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r src/<agent-name>/requirements.txt
```

### Requirements for Local Development

```
python-dotenv
azure-identity
azure-ai-agentserver-agentframework
```

### Configure Environment

```bash
# Get the endpoint from provisioned resources
cat .azure/<env-name>/.env | grep AZURE_AI_PROJECT_ENDPOINT

# Create .env in project root
echo "PROJECT_ENDPOINT=<paste-endpoint-here>" > .env
echo "MODEL_DEPLOYMENT_NAME=gpt-4.1" >> .env
```

### Run and Test

**Terminal 1** - Start agent:
```bash
python src/<agent-name>/main.py
# Output: Agent running on http://localhost:8088
```

**Terminal 2** - Test:
```bash
curl -X POST http://localhost:8088/responses \
    -H "Content-Type: application/json" \
    -d '{"input": "Hello, what can you do?"}'
```

✅ **Success looks like**: JSON response with agent's reply.

**For testing issues**: See `foundry-hosted-agents-test` skill.

---

## PHASE 3: DEPLOY TO AZURE

### Deploy Command

```bash
azd deploy <agent-name>
```

### CRITICAL: Check Console Output

**Always review the console output after deployment.** Look for:
- ✅ "Deployment completed successfully"
- ❌ Container build failures
- ❌ ACR push errors
- ❌ Role assignment issues

### Verify Deployment

```bash
# Check agent status
az cognitiveservices agent status \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1

# If issues, check logs
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

**For deployment options and issues**: See `foundry-hosted-agents-deploy` skill.

---

## PHASE 4: TEST DEPLOYED AGENT

### IMPORTANT: Different API Than Local Testing

Deployed agents **cannot** be tested with `curl`. You must use the Azure SDK.

### Install Test Dependencies

```bash
pip install azure-ai-projects azure-identity python-dotenv
```

### Create test_deployed_agent.py

```python
import os
from dotenv import load_dotenv
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

load_dotenv()
PROJECT_ENDPOINT = os.getenv("PROJECT_ENDPOINT")
AGENT_NAME = "<agent-name>"  # Must match name in agent.yaml

def test_deployed_agent():
    project_client = AIProjectClient(
        endpoint=PROJECT_ENDPOINT,
        credential=DefaultAzureCredential(),
    )
    
    openai_client = project_client.get_openai_client()
    conversation = openai_client.conversations.create()
    
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

### Run Test

```bash
az login
python test_deployed_agent.py
```

✅ **Success looks like**: Agent's response printed to console.

**For remote testing issues**: See `foundry-hosted-agents-test` skill.

---

## QUICK REFERENCE: Requirements by Phase

| Phase | Requirements |
|-------|-------------|
| Local Development | `python-dotenv`, `azure-identity`, `azure-ai-agentserver-agentframework` |
| Remote Testing | `azure-ai-projects`, `azure-identity`, `python-dotenv` |
| Full Development | All of the above |

### Combined requirements.txt for full development:

```
python-dotenv
azure-identity
azure-ai-agentserver-agentframework
azure-ai-projects
```

---

## COMMON ISSUES BY PHASE

| Phase | Common Issue | Quick Fix |
|-------|-------------|-----------|
| Create | `azd` not found | Install Azure Developer CLI |
| Local Test | `Connection refused` | Start agent: `python main.py` |
| Local Test | `AuthenticationError` | Run `az login` |
| Deploy | Console shows errors | Check logs, see troubleshoot skill |
| Remote Test | No response | Use SDK, not curl; include `extra_body` |

**For detailed troubleshooting**: See `foundry-hosted-agents-troubleshoot` skill.

---

## NEXT STEPS AFTER QUICKSTART

Once your agent is working:

1. **Add custom tools** - See `foundry-hosted-agents-create` skill, "WHEN USER ASKS TO ADD TOOLS"
2. **Use Foundry tools** - Web search, file search, code interpreter
3. **Set up CI/CD** - Use `azd pipeline config`
4. **Monitor** - Add Application Insights telemetry

---

## Resources

- [Quickstart Documentation](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
- [Sample Agents](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents)
- [CLI Reference](https://learn.microsoft.com/en-us/cli/azure/cognitiveservices/agent)
