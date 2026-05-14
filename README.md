# AgentOS AWS Template

An agent platform you build, improve, and run using coding agents. Deploy to AWS with ECS Fargate, RDS PostgreSQL, and an application load balancer.

The platform runs in your cloud, behind your auth, with all your data stored in your database. Because trace data, agent code, system logs, and the iteration tool all live in one place, coding agents like Claude Code can read, update, and improve the platform end-to-end.

## Built for coding agents

This codebase is designed primarily for coding agents. It comes with five prompts that cover the full agent development lifecycle:

1. **Create.** Claude asks a few questions, scaffolds the agent file, registers it in `app/main.py`, adds quick prompts to `app/config.yaml`, restarts the container, and smoke-tests via cURL. Usually 5-10 minutes for a simple agent.
2. **Improve.** Hardens and fine-tunes your agent based on its existing spec. Claude derives probes from the agent's `INSTRUCTIONS`, runs them against the live container, judges the responses, and edits until they pass. No input from you.
3. **Extend.** Add a new feature to an agent. You direct, Claude executes. Add tools, refine prompts, fix bugs. The Agno docs MCP is loaded so toolkit research is grounded in the real API.
4. **Hill Climb.** Claude runs the eval suite, diagnoses failures, and fixes what's in scope. Stops when all cases pass.
5. **Review.** Claude sweeps the repo for drift between docs, code, and config. Auto-fixes mechanical drift like stale paths and missing env vars; flags anything bigger.

3 of 5 run autonomously with no input needed from you.

## What's Included

| Agent | Pattern | Description |
|-------|---------|-------------|
| WebSearch | Direct tools | Search the web using Parallel SDK or keyless MCP. |
| CodeSearch | Context provider | Answer questions about this codebase. |

## Get Started

### Step 1: Run locally

> **Prerequisite:** [Docker](https://www.docker.com/get-started/) installed and running.

```sh
# Clone the repo
git clone https://github.com/agno-agi/agentos-aws-template.git agentos-aws
cd agentos-aws

# Add OPENAI_API_KEY
cp example.env .env
# Edit .env and add your key

# Start the application
docker compose up -d --build
```

Confirm AgentOS is running at [http://localhost:8000/docs](http://localhost:8000/docs).

### Step 2: Connect to the Web UI

1. Open [os.agno.com](https://os.agno.com) and login
2. Add OS → Local → `http://localhost:8000`
3. Click "Connect"

### Step 3: Stop the application

```sh
docker compose down
```

## Deploy to AWS

### Prerequisites

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured
- Docker installed (for building images)

### Configure secrets

```sh
# Copy the example secrets
cp infra/example_secrets/prd_api_secrets.yml infra/secrets/prd_api_secrets.yml

# Edit and add your API keys
# OPENAI_API_KEY is required
```

### Deploy

```sh
ag infra up --env prd
```

The first deploy will fail intentionally — JWT auth is on by default and `JWT_VERIFICATION_KEY` isn't set yet. Get the key from os.agno.com (Add OS → Live → Token Based Authorization), add it to your secrets file, and redeploy.

See the [full AWS deployment guide](https://docs.agno.com/production/templates/aws) for configuring subnets, HTTPS, and connecting to the control plane.

### Manage deployment

```sh
ag infra patch --env prd     # Update after changes
ag infra down --env prd      # Tear down all resources
```

## Project Structure

```
├── agents/                  # Agents
│   ├── web_search.py        # WebSearch — Parallel SDK or keyless MCP
│   └── code_search.py       # CodeSearch — WorkspaceContextProvider
├── app/
│   ├── main.py              # AgentOS entry point
│   ├── settings.py          # default_model() factory
│   └── config.yaml          # Quick prompts config
├── db/
│   ├── session.py           # PostgreSQL database helpers
│   └── url.py               # Connection URL builder
├── docs/                    # Claude Code workflow prompts
├── evals/                   # Agent evaluation suite
├── infra/                   # AWS infrastructure config
│   ├── settings.py          # Region, subnets, image settings
│   ├── dev_resources.py     # Docker resources (local via ag CLI)
│   └── prd_resources.py     # AWS resources (ECS, RDS, ALB)
├── scripts/                 # Helper scripts
├── compose.yaml             # Docker Compose for local development
├── Dockerfile
├── example.env
└── pyproject.toml           # Dependencies
```

## Common Tasks

### Add your own agent

1. **Hand it to Claude Code** — paste `Run docs/create-new-agent.md` into a Claude Code session. Claude asks what the agent should do, generates the file, registers it, smoke-tests it.

2. **Do it manually** — create `agents/my_agent.py`:

```python
from agno.agent import Agent

from app.settings import default_model
from db import get_postgres_db

my_agent = Agent(
    id="my-agent",
    name="My Agent",
    model=default_model(),
    db=get_postgres_db(),
    instructions="You are a helpful assistant.",
    enable_agentic_memory=True,
    add_datetime_to_context=True,
    add_history_to_context=True,
    num_history_runs=5,
    markdown=True,
)
```

Register in `app/main.py` and restart: `docker compose restart agentos-api`

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

```sh
# 1. Edit pyproject.toml
# 2. Regenerate requirements
./scripts/generate_requirements.sh upgrade
# 3. Rebuild
docker compose up -d --build
```

### Use a different model provider

1. Add your API key to `.env` (e.g., `ANTHROPIC_API_KEY`)
2. Update `app/settings.py`:

```python
from agno.models.anthropic import Claude

def default_model():
    return Claude(id="claude-sonnet-4-5")
```

3. Add dependency to `pyproject.toml`

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes | - | OpenAI API key |
| `RUNTIME_ENV` | No | `prd` | `dev` enables hot-reload and disables JWT |
| `JWT_VERIFICATION_KEY` | Prd | - | Public key from os.agno.com |
| `AGENTOS_URL` | No | `http://127.0.0.1:8000` | Scheduler base URL |
| `PARALLEL_API_KEY` | No | - | Parallel SDK key (optional for WebSearch) |
| `SLACK_BOT_TOKEN` | No | - | Enable Slack interface |
| `SLACK_SIGNING_SECRET` | No | - | Enable Slack interface |
| `DB_HOST` | No | `localhost` | Database host |
| `DB_PORT` | No | `5432` | Database port |
| `DB_USER` | No | `ai` | Database user |
| `DB_PASS` | No | `ai` | Database password |
| `DB_DATABASE` | No | `ai` | Database name |

## Learn More

- [AWS Deployment Guide](https://docs.agno.com/production/templates/aws)
- [Agno Documentation](https://docs.agno.com)
- [AgentOS Documentation](https://docs.agno.com/agent-os/introduction)
- [Agno Discord](https://agno.com/discord)
