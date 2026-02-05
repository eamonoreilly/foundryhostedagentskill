---
name: foundry-hosted-agents-test
description: Test Foundry Hosted Agents locally and remotely. Use when users ask to test, run, debug, execute, or call hosted agents. Triggers include curl localhost:8088, test agent, run agent locally, test deployed agent, python main.py, agent not responding. USE FOR: test agent, run agent locally, curl localhost:8088, python main.py, test deployed agent, run agent, execute agent, call agent, verify agent works. DO NOT USE FOR: creating agents (use foundry-hosted-agents-create), deploying agents (use foundry-hosted-agents-deploy), fixing errors (use foundry-hosted-agents-troubleshoot), getting started tutorials (use foundry-hosted-agents-quickstart).
---

# Test Foundry Hosted Agents

Use this skill when users want to **test or run hosted agents** either locally or after deployment.

For creating agents, see the `foundry-hosted-agents-create` skill.
For deploying agents, see the `foundry-hosted-agents-deploy` skill.
For troubleshooting, see the `foundry-hosted-agents-troubleshoot` skill.

---

## WHEN USER ASKS TO TEST LOCALLY:

### Prerequisites

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
az login
```

### Requirements for Local Development

The `requirements.txt` should contain:
```
python-dotenv
azure-identity
azure-ai-agentserver-agentframework
```

### Environment Configuration

Ensure `.env` file exists with:
```env
PROJECT_ENDPOINT=https://<account>.services.ai.azure.com/api/projects/<project>
MODEL_DEPLOYMENT_NAME=gpt-4.1
```

### Run Agent (Two Terminals)

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

### Single-Terminal Workaround (for AI Tools)

When you cannot use separate terminals, use this pattern:

```bash
# Kill any existing agent, start in background, wait, then test
pkill -f "python.*main.py" 2>/dev/null; sleep 1
python src/<agent-name>/main.py > /tmp/agent.log 2>&1 &
sleep 5
curl -s -X POST http://localhost:8088/responses -H "Content-Type: application/json" -d '{"input": "Hello"}'
```

Check logs if needed: `cat /tmp/agent.log`

### Stop Local Agent

```bash
# Find and kill process on port 8088
lsof -ti:8088 | xargs kill -9

# Or kill by name
pkill -f "python.*main.py"
```

---

## WHEN USER ASKS TO TEST A DEPLOYED AGENT:

**CRITICAL**: Deployed agents use a DIFFERENT API than local testing. You cannot use `curl` directly.

### test_deployed_agent.py

Create this file to test deployed agents:

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

### Requirements for Remote Testing

Install these packages (different from local testing!):
```bash
pip install azure-ai-projects azure-identity python-dotenv
```

Or add to requirements.txt:
```
azure-ai-projects
azure-identity
python-dotenv
```

### Run Remote Test

```bash
az login
python test_deployed_agent.py
```

---

## WHEN USER ASKS ABOUT TESTING DIFFERENCES:

### API Comparison: Local vs Deployed

| Aspect | Local Testing | Deployed Testing |
|--------|--------------|------------------|
| Endpoint | `http://localhost:8088/responses` | Via `AIProjectClient` |
| Method | Direct HTTP (curl) | `openai_client.responses.create()` |
| Agent reference | Not needed | `extra_body={"agent": {...}}` **required** |
| Auth | DefaultAzureCredential (implicit) | DefaultAzureCredential (explicit) |
| Port | 8088 | N/A (managed by Azure) |

### Common Mistakes in Remote Testing

| Mistake | Why It Fails | Correct Approach |
|---------|--------------|------------------|
| Using `AgentsClient` | No `responses` attribute | Use `AIProjectClient.get_openai_client()` |
| Direct HTTP to deployed agent | Complex auth required | Use SDK with `DefaultAzureCredential` |
| Forgetting `extra_body` | Agent not identified | Always include `extra_body={"agent": {...}}` |
| Using wrong agent name | Agent not found | Must match `name` in agent.yaml exactly |

---

## WHEN USER'S LOCAL TEST ISN'T WORKING:

### Checklist

1. **Is the agent running?**
   ```bash
   lsof -i:8088
   ```

2. **Is .env configured?**
   ```bash
   cat .env
   # Should have PROJECT_ENDPOINT and MODEL_DEPLOYMENT_NAME
   ```

3. **Are you logged into Azure?**
   ```bash
   az account show
   # If not: az login
   ```

4. **Check agent logs:**
   ```bash
   cat /tmp/agent.log  # If using background mode
   ```

5. **Restart the agent:**
   ```bash
   pkill -f "python.*main.py" 2>/dev/null
   sleep 2
   python main.py
   ```

### Common Local Testing Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Connection refused` | Agent not running | Start agent with `python main.py` |
| `AuthenticationError` | Not logged into Azure | Run `az login` |
| `PROJECT_ENDPOINT is required` | Missing .env | Create .env with endpoint |
| Port 8088 in use | Previous agent running | `lsof -ti:8088 \| xargs kill -9` |

---

## WHEN USER'S REMOTE TEST ISN'T WORKING:

See the `foundry-hosted-agents-troubleshoot` skill for detailed error resolution.

Quick checks:
```bash
# Check agent status
az cognitiveservices agent status \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1

# Check agent logs
az cognitiveservices agent logs show \
    --account-name <account> \
    --project-name <project> \
    --name <agent-name> \
    --agent-version 1
```

---

## NEXT STEPS

### After Local Testing Passes

➡️ **Deploy your agent**: Run `azd deploy <agent-name>` and check console output for errors.

See `foundry-hosted-agents-deploy` skill for deployment options.

### After Remote Testing Passes

✅ **Your agent is production-ready!** Consider:
- Adding more tools to your agent
- Setting up CI/CD with `azd pipeline config`
- Adding Application Insights for monitoring

---

## QUICK REFERENCE: Requirements by Scenario

| Scenario | Packages |
|----------|----------|
| Local Development | `python-dotenv`, `azure-identity`, `azure-ai-agentserver-agentframework` |
| Remote Testing | `azure-ai-projects`, `azure-identity`, `python-dotenv` |
| Full Development | All of the above |
