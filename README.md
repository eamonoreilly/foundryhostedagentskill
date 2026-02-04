# Foundry Hosted Agents Development Kit

This repository provides a Copilot skill and guidance for developing, testing, and deploying **Microsoft Foundry Hosted Agents**.

## Getting Started

### Prerequisites

- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) with `ai` extension
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)
- Python 3.12+
- VS Code with GitHub Copilot extension
- Azure subscription with access to Azure AI Foundry

### Installation

1. Clone this repository
2. Open in VS Code
3. The Copilot skill in `.github/copilot-skills/` will be automatically available

## Sample Prompts for Copilot

Use these prompts to get started with Foundry Hosted Agents. They will trigger the included skill which provides detailed guidance.

### Creating a New Hosted Agent

- "Create a new Foundry hosted agent project with all required infrastructure"
- "Help me set up a hosted agent that can search the web and run Python code"
- "I want to create a hosted agent with custom Python tools"

### Working with Existing Infrastructure

- "I have an existing Foundry project. How do I deploy a hosted agent to it?"
- "Deploy my agent to my existing Foundry account using az cognitiveservices agent"

### Local Development and Testing

- "How do I test my hosted agent locally before deploying?"
- "Run my agent locally and test it with a curl request"
- "Set up the local development environment for my hosted agent"

### Deployment

- "Deploy my hosted agent to Foundry"
- "Use azd to provision infrastructure and deploy my agent"
- "What is the difference between azd deploy and az cognitiveservices agent create?"

### Troubleshooting

- "My deployed agent is failing. How do I check the container logs?"
- "I am getting Azure AI project endpoint is required error"
- "The agent cannot access the model - getting 403 errors"
- "How do I fix AcrPullUnauthorized errors?"

### Testing Deployed Agents

- "How do I test my deployed hosted agent?"
- "Create a Python script to test my hosted agent on Foundry"
- "What is the difference between local and deployed agent testing?"

### Managing Agents

- "List all my deployed hosted agents"
- "Check the status of my hosted agent"
- "Stop and restart my hosted agent"
- "Delete my hosted agent"

### Adding Capabilities

- "How do I add custom tools to my hosted agent?"
- "Add a weather lookup function as a tool for my agent"

### Role Assignments and Security

- "What roles does the project managed identity need for hosted agents?"
- "Grant the required permissions for my hosted agent to work"

## Project Structure

When you create a hosted agent project, it will have this structure:

```
my-project/
├── .github/
│   └── skills/                   # Copilot skills for this project
│       ├── foundry-hosted-agents-create/
│       │   └── SKILL.md          # Creating and scaffolding agents
│       ├── foundry-hosted-agents-test/
│       │   └── SKILL.md          # Local and remote testing
│       ├── foundry-hosted-agents-deploy/
│       │   └── SKILL.md          # Deployment workflows
│       └── foundry-hosted-agents-troubleshoot/
│           └── SKILL.md          # Error diagnosis and fixes
├── azure.yaml                    # azd service definitions
├── infra/                        # Bicep infrastructure
├── src/
│   └── my-agent/                 # Agent code
│       ├── agent.yaml
│       ├── main.py
│       ├── Dockerfile
│       └── requirements.txt
├── .env                          # Local development config
└── README.md
```

## Skills Overview

The Copilot skills are split by workflow for better matching:

| Skill | Triggers When You Ask About |
|-------|----------------------------|
| `foundry-hosted-agents-create` | Creating, scaffolding, initializing new agents |
| `foundry-hosted-agents-test` | Running, testing, debugging agents locally or remotely |
| `foundry-hosted-agents-deploy` | Deploying, publishing agents to Azure |
| `foundry-hosted-agents-troubleshoot` | Errors, failures, logs, permission issues |

## Workflow Overview

1. **Initialize** - `azd init` + `azd ai agent init`
2. **Provision** - `azd provision` (creates Foundry, ACR, models)
3. **Local Test** - `python main.py` + `curl`
4. **Deploy** - `azd deploy <agent-name>`
5. **Remote Test** - `test_deployed_agent.py`

## Quick Commands

| Task | Command |
|------|---------|
| Initialize project | `azd init -t https://github.com/Azure-Samples/azd-ai-starter-basic` |
| Add agent sample | `azd ai agent init -m <agent.yaml-url>` |
| Provision infrastructure | `azd provision` |
| Deploy agent | `azd deploy <agent-name>` |
| Check agent logs | `az cognitiveservices agent logs show -a <account> -p <project> -n <name> --agent-version 1` |
| List agents | `az cognitiveservices agent list -a <account> -p <project>` |

## Resources

- [Foundry Hosted Agents Quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
- [Azure CLI Agent Reference](https://learn.microsoft.com/en-us/cli/azure/cognitiveservices/agent)
- [Sample Agents](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents)

## License

MIT
