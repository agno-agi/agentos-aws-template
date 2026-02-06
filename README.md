# Agent OS AWS Template

Run agents, teams, and workflows as a production-ready API. Develop on Docker, deploy to AWS.

## Quickstart

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop) installed and running
- [OpenAI API key](https://platform.openai.com/api-keys)

### Clone and configure

```sh
git clone https://github.com/agno-agi/agentos-aws-template.git agentos-aws
cd agentos-aws

cp example.env .env
# Add OPENAI_API_KEY to .env
```

> Agno works with any model provider. Update the agents in `/agents` and add dependencies to `pyproject.toml`.

### Start the application

This template supports 2 environments, `dev` and `prd`.

### Run the application locally in docker:

```sh
ag infra up --env dev
```

This command starts:

- The **AgentOS instance**, which is a FastAPI server, running on [http://localhost:8080](http://localhost:8080).
- The **PostgreSQL database**, accessible on `localhost:5432`.

Open http://localhost:8000/docs to see the API.

### Connect to the control plane

1. Open [os.agno.com](https://os.agno.com)
2. Click "Add OS" and select "Local"
3. Enter `http://localhost:8000`

### Stop the application

When you're done, stop the application using:

```sh
ag infra down
```

### Run the application in AWS:

```sh
ag infra up --env prd
```

### This command will create the following resources:

- AWS Security Groups
- AWS Secrets
- AWS Db Subnet Group
- AWS RDS Instance
- AWS Load Balancer
- AWS Target Group
- AWS Listener
- AWS ECS Cluster
- AWS ECS Service
- AWS ECS Task
- AWS ECS Task Definition

### Connect to the control plane

1. In order to connect your load balancer to the Control plane, you need to make sure its HTTPS. Read more [here](https://docs.agno.com/production/aws/domain-https)
1. Open [os.agno.com](https://os.agno.com)
2. Click "Add OS" and select "Live"
3. Enter the domain of your load balancer

## Project Structure

```
agentos-aws/
├── agents/              # Your agents
├── app/                 # AgentOS entry point
├── db/                  # Database connection
├── scripts/             # Helper scripts
├── infra/               # Infrastructure configuration
├── Dockerfile           # Container build
├── example.env          # Example environment variables
└── pyproject.toml       # Python dependencies
```

## Common Tasks

### Load a knowledge base

Locally:
```sh
docker exec -it agentos-aws-template-api python -m agents.knowledge_agent
```

On AWS:
```sh
ECS_CLUSTER=agentos-aws-template-prd-cluster
TASK_ARN=$(aws ecs list-tasks --cluster agentos-aws-template-prd-cluster --query "taskArns[0]" --output text)
CONTAINER_NAME=agentos-aws-template

aws ecs execute-command --cluster $ECS_CLUSTER \
    --task $TASK_ARN \
    --container $CONTAINER_NAME \
    --interactive \
    --command "zsh"
```

After SSHing into the container, run the following command to load the knowledge base:

```sh
python -m agents.knowledge_agent
```

### View logs
```sh
docker compose logs -f
```

### Restart after code changes
```sh
docker compose restart
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes | - | OpenAI API key |
| `EXA_API_KEY` | No | - | Exa API key for web research |
| `DB_HOST` | No | `localhost` | Database host |
| `DB_PORT` | No | `5432` | Database port |
| `DB_USER` | No | `ai` | Database user |
| `DB_PASS` | No | `ai` | Database password |
| `DB_DATABASE` | No | `ai` | Database name |
| `DATA_DIR` | No | `/data` | Directory for DuckDB storage |
| `RUNTIME_ENV` | No | `prd` | Set to `dev` for auto-reload |

## Local Development

For development without Docker:

### Install uv
```sh
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Setup environment
```sh
./scripts/venv_setup.sh
source .venv/bin/activate
```

### Add dependencies

1. Edit `pyproject.toml`
2. Regenerate requirements:
```sh
./scripts/generate_requirements.sh
```
3. Rebuild:
```sh
docker compose up -d --build
```

## Learn More

- [Managing AWS Resources](https://docs.agno.com/production/aws/production-app)
- [Agno Documentation](https://docs.agno.com)
- [AgentOS Documentation](https://docs.agno.com/agent-os)
- [Discord Community](https://agno.link/discord)