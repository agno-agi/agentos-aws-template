# AgentOS AWS Template

Deploy a multi-agent system on AWS with ECS Fargate, RDS PostgreSQL, and an application load balancer.

## What's Included

| Agent | Pattern | Description |
|-------|---------|-------------|
| Knowledge Agent | Agentic RAG | Answers questions from a knowledge base. |
| MCP Agent | MCP Tool Use | Connects to external services via MCP. |

## Get Started

```sh
# Clone the repo
git clone https://github.com/agno-agi/agentos-aws-template.git agentos-aws
cd agentos-aws

# Add OPENAI_API_KEY
cp example.env .env
# Edit .env and add your key

# Start the application
ag infra up --env dev

# Load documents for the knowledge agent
docker exec -it agentos-aws-template-api python -m agents.knowledge_agent
```

Confirm AgentOS is running at [http://localhost:8000/docs](http://localhost:8000/docs).

### Connect to the Web UI

1. Open [os.agno.com](https://os.agno.com) and login
2. Add OS → Local → `http://localhost:8000`
3. Click "Connect"

### Stop the application

```sh
ag infra down --env dev
```

## Deploy to AWS

Requires:
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured
- `OPENAI_API_KEY` set in your environment

```sh
ag infra up --env prd
```

See the [full AWS deployment guide](https://docs.agno.com/production/templates/aws) for configuring subnets, HTTPS, and connecting to the control plane.

### Manage deployment

```sh
ag infra patch --env prd     # Update after changes
ag infra down --env prd      # Tear down all resources
```

## The Agents

### Knowledge Agent

Answers questions using hybrid search over a vector database (Agentic RAG).

**Load documents:**

```sh
# Local
docker exec -it agentos-aws-template-api python -m agents.knowledge_agent

# AWS
ECS_CLUSTER=agentos-aws-template-prd
TASK_ARN=$(aws ecs list-tasks --cluster $ECS_CLUSTER --query "taskArns[0]" --output text)

aws ecs execute-command --cluster $ECS_CLUSTER \
    --task $TASK_ARN \
    --container agentos-aws-template \
    --interactive \
    --command "python -m agents.knowledge_agent"
```

**Try it:**

```
What is Agno?
How do I create my first agent?
What documents are in your knowledge base?
```

### MCP Agent

Connects to external tools via the Model Context Protocol.

**Try it:**

```
What tools do you have access to?
Search the docs for how to use LearningMachine
Find examples of agents with memory
```

## Project Structure
```
├── agents/                  # Agents
│   ├── knowledge_agent.py   # Agentic RAG
│   └── mcp_agent.py         # MCP Tool Use
├── app/
│   ├── main.py              # AgentOS entry point
│   └── config.yaml          # Quick prompts config
├── db/
│   ├── session.py           # PostgreSQL database helpers
│   └── url.py               # Connection URL builder
├── infra/                   # AWS infrastructure config
│   ├── settings.py          # Region, subnets, image settings
│   ├── dev_resources.py     # Docker resources (local)
│   └── prd_resources.py     # AWS resources (ECS, RDS, ALB)
├── scripts/                 # Helper scripts
├── Dockerfile
├── example.env
└── pyproject.toml           # Dependencies
```

## Common Tasks

### Add your own agent

1. Create `agents/my_agent.py`:

```python
from agno.agent import Agent
from agno.models.openai import OpenAIResponses
from db import get_postgres_db

my_agent = Agent(
    id="my-agent",
    name="My Agent",
    model=OpenAIResponses(id="gpt-5.2"),
    db=get_postgres_db(),
    instructions="You are a helpful assistant.",
)
```

2. Register in `app/main.py`:

```python
from agents.my_agent import my_agent

agent_os = AgentOS(
    name="AgentOS",
    agents=[knowledge_agent, mcp_agent, my_agent],
    ...
)
```

3. Restart: `ag infra up --env dev`

### Add tools to an agent

Agno includes 100+ tool integrations. See the [full list](https://docs.agno.com/tools/toolkits).

```python
from agno.tools.slack import SlackTools
from agno.tools.google_calendar import GoogleCalendarTools

my_agent = Agent(
    ...
    tools=[
        SlackTools(),
        GoogleCalendarTools(),
    ],
)
```

### Add dependencies

1. Edit `pyproject.toml`
2. Regenerate requirements: `./scripts/generate_requirements.sh`
3. Rebuild: `ag infra up --env dev`

### Use a different model provider

1. Add your API key to `.env` (e.g., `ANTHROPIC_API_KEY`)
2. Update agents to use the new provider:

```python
from agno.models.anthropic import Claude

model=Claude(id="claude-sonnet-4-5")
```
3. Add dependency: `anthropic` in `pyproject.toml`

---

## Local Development

For development without Docker:

```sh
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Setup environment
./scripts/venv_setup.sh
source .venv/bin/activate
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes | - | OpenAI API key |
| `DB_HOST` | No | `localhost` | Database host |
| `DB_PORT` | No | `5432` | Database port |
| `DB_USER` | No | `ai` | Database user |
| `DB_PASS` | No | `ai` | Database password |
| `DB_DATABASE` | No | `ai` | Database name |
| `RUNTIME_ENV` | No | `prd` | Set to `dev` for auto-reload |

## Learn More

- [AWS Deployment Guide](https://docs.agno.com/production/templates/aws)
- [Agno Documentation](https://docs.agno.com)
- [AgentOS Documentation](https://docs.agno.com/agent-os/introduction)
- [Agno Discord](https://agno.com/discord)
