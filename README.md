# AgentOS AWS Template

Deploy a multi-agent system to production on AWS. Develop locally with Docker, deploy to ECS.

[What is AgentOS?](https://docs.agno.com/agent-os/introduction) · [Agno Docs](https://docs.agno.com) · [Discord](https://agno.com/discord)

## What's Included

| Agent | Pattern | Description |
|-------|---------|-------------|
| **Pal** | Learning + Tools | Your AI-powered second brain |
| Knowledge Agent | RAG | Answers questions from a knowledge base |
| MCP Agent | Tool Use | Connects to external services via MCP |

**Pal** (Personal Agent that Learns) is your AI-powered second brain. It researches, captures, organizes, connects, and retrieves your personal knowledge.

---

## Quick Start: Local Development

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop) installed and running
- [OpenAI API key](https://platform.openai.com/api-keys)
- [Agno CLI](https://docs.agno.com/introduction/installation#agno-cli) (`pip install agno`)

### 1. Clone and configure

```sh
git clone https://github.com/agno-agi/agentos-aws-template.git agentos-aws
cd agentos-aws

cp example.env .env
```

Edit `.env` and add your OpenAI API key:
```
OPENAI_API_KEY=sk-your-key-here
```

### 2. Start locally

```sh
ag infra up --env dev
```

This starts two containers:
- **API server** at http://localhost:8080
- **PostgreSQL** at localhost:5432

**Verify:** Open http://localhost:8080/docs — you should see the API documentation.

### 3. Connect to control plane

1. Open [os.agno.com](https://os.agno.com)
2. Click "Add OS" → "Local"
3. Enter `http://localhost:8080`

**Verify:** You should see your agents listed in the control plane.

### Managing local deployment

```sh
docker logs -f agentos-aws-template-api    # View logs
ag infra up --env dev -y                   # Rebuild after code changes
ag infra down --env dev                    # Stop containers
```

---

## Deploy to AWS (Step-by-Step)

Follow these steps in order. Each step includes a verification command so you know it worked.

> **Time estimate:** ~30 minutes for first deployment

### Step 1: Install and Configure AWS CLI

Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), then configure it:

```sh
aws configure
```

Enter:
- **AWS Access Key ID:** Your access key
- **AWS Secret Access Key:** Your secret key
- **Default region:** `us-east-1` (recommended)
- **Default output format:** `json`

**Verify:**
```sh
aws sts get-caller-identity
```

You should see your account ID and user ARN:
```json
{
    "UserId": "AIDAEXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-user"
}
```

### Step 2: Get Your Subnet IDs

Subnets are network segments in your AWS VPC. You need two subnets from different availability zones for high availability.

List your subnets:
```sh
aws ec2 describe-subnets --query 'Subnets[*].[SubnetId,AvailabilityZone,VpcId]' --output table
```

Example output:
```
---------------------------------------------------------
|                    DescribeSubnets                    |
+---------------------------+---------------+-----------+
|  subnet-0abc123def456789a |  us-east-1a   |  vpc-xxx  |
|  subnet-0def456789abc123b |  us-east-1b   |  vpc-xxx  |
|  subnet-0ghi789abc123def4 |  us-east-1c   |  vpc-xxx  |
+---------------------------+---------------+-----------+
```

Pick **two subnet IDs from different availability zones** (e.g., one from `us-east-1a` and one from `us-east-1b`).

Edit `infra/settings.py` and add your subnet IDs:
```python
aws_subnet_ids=["subnet-0abc123def456789a", "subnet-0def456789abc123b"],
```

**Verify:** The line is uncommented and contains your actual subnet IDs.

> **No subnets?** You may need to create a default VPC. Run: `aws ec2 create-default-vpc`

### Step 3: Set Up Container Registry (ECR)

AWS ECR (Elastic Container Registry) stores your Docker images. Create a repository:

```sh
aws ecr create-repository --repository-name agentos-aws-template --region us-east-1
```

Get your AWS account ID:
```sh
aws sts get-caller-identity --query Account --output text
```

Edit `infra/settings.py` with your account ID:
```python
image_repo="123456789012.dkr.ecr.us-east-1.amazonaws.com",
```

Also uncomment `push_images`:
```python
push_images=True,
```

**Verify:**
```sh
aws ecr describe-repositories --repository-names agentos-aws-template
```

You should see your repository details.

> **Alternative: DockerHub** — If you prefer DockerHub, run `docker login` and set `image_repo="your-dockerhub-username"` instead.

### Step 4: Configure Secrets

Secrets are sensitive values (API keys, passwords) that get stored in AWS Secrets Manager.

Copy the example secrets:
```sh
cp -r infra/example_secrets infra/secrets
```

Edit the secret files:

**`infra/secrets/prd_api_secrets.yml`** — Add your OpenAI API key:
```yaml
SECRET_KEY: "your-random-secret-key"
OPENAI_API_KEY: "sk-your-openai-key"
```

**`infra/secrets/prd_db_secrets.yml`** — Set a database password:
```yaml
MASTER_USERNAME: agno
MASTER_USER_PASSWORD: "YourSecurePassword123!"
```

**Verify:**
```sh
ls infra/secrets/
```

You should see: `dev_api_secrets.yml  prd_api_secrets.yml  prd_db_secrets.yml`

### Step 5: Authenticate Docker to ECR

Before deploying, authenticate Docker to push images to ECR:

```sh
./scripts/auth_ecr.sh
```

Or manually:
```sh
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

**Verify:** You should see "Login Succeeded".

### Step 6: Deploy to AWS

Now deploy:

```sh
ag infra up --env prd
```

This creates the following AWS resources:
- Security Groups (for network access control)
- Secrets (in AWS Secrets Manager)
- RDS PostgreSQL database
- Application Load Balancer
- ECS Cluster with your application

**Time estimate:** 10-15 minutes for first deployment.

**Verify:**
```sh
aws ecs list-services --cluster agentos-aws-template-prd-cluster
```

You should see your service listed.

### Step 7: Get Your Application URL

Get your load balancer DNS:

```sh
aws elbv2 describe-load-balancers --names agentos-aws-template-prd-api-lb --query 'LoadBalancers[0].DNSName' --output text
```

**Verify:** Open `http://YOUR-LOAD-BALANCER-DNS/docs` in a browser. You should see the API documentation.

### Step 8: Connect to Control Plane

1. **Set up HTTPS** for your load balancer — see [HTTPS setup guide](https://docs.agno.com/production/aws/domain-https)
2. Open [os.agno.com](https://os.agno.com)
3. Click "Add OS" → "Live"
4. Enter your load balancer domain (with HTTPS)

### Managing AWS Deployment

```sh
ag infra up --env prd -y      # Update after code changes
ag infra down --env prd       # Tear down all resources (removes database!)
```

---

## Troubleshooting

### "No subnets found" or empty subnet list

Your AWS account may not have a default VPC. Create one:
```sh
aws ec2 create-default-vpc
```

Then re-run the subnet list command from Step 2.

### "ECR authentication failed" or "no basic auth credentials"

Re-run the ECR authentication:
```sh
./scripts/auth_ecr.sh
```

Or check that your AWS CLI is configured correctly:
```sh
aws sts get-caller-identity
```

### "Image push failed" or "repository does not exist"

Make sure the ECR repository exists:
```sh
aws ecr describe-repositories --repository-names agentos-aws-template
```

If not, create it:
```sh
aws ecr create-repository --repository-name agentos-aws-template --region us-east-1
```

### ECS task keeps restarting or failing

Check the task logs:
```sh
aws logs tail /ecs/agentos-aws-template-prd --follow
```

Common causes:
- Missing environment variables → Check your secrets files
- Database connection failed → Check security groups allow traffic
- Application crash → Check for errors in the logs

### Load balancer shows "unhealthy" targets

The health check may be failing. Verify the health check endpoint:
```sh
curl http://YOUR-LOAD-BALANCER-DNS/health
```

Should return `{"status": "healthy"}`.

### "Connection refused" to database

Check that security groups allow traffic between ECS tasks and RDS:
```sh
aws ec2 describe-security-groups --group-names agentos-aws-template-prd-sg agentos-aws-template-prd-db-sg
```

### Deployment is stuck or taking too long

First deployments take 10-15 minutes. If it's been longer:
1. Check CloudFormation for errors: AWS Console → CloudFormation
2. Check ECS service events: AWS Console → ECS → Clusters → Services → Events

---

## The Agents

### Pal (Personal Agent that Learns)

Your AI-powered second brain. Pal researches, captures, organizes, connects, and retrieves your personal knowledge.

**What Pal stores:**

| Type | Examples |
|------|----------|
| **Notes** | Ideas, decisions, snippets, learnings |
| **Bookmarks** | URLs with context - why you saved it |
| **People** | Contacts - who they are, how you know them |
| **Meetings** | Notes, decisions, action items |
| **Projects** | Goals, status, related items |
| **Research** | Findings from web search, saved for later |

**Try it:**
```
Note: decided to use Postgres for the new project - better JSON support
Bookmark https://www.ashpreetbedi.com/articles/lm-technical-design - great intro
Research event sourcing patterns and save the key findings
What notes do I have?
```

**How it works:**
- **DuckDB** stores structured data (notes, bookmarks, people, etc.)
- **Learning system** remembers schemas and research findings
- **Exa search** powers web research (requires `EXA_API_KEY`)

### Knowledge Agent

Answers questions using a vector knowledge base (RAG pattern).

**Try it:**
```
What is Agno?
How do I create my first agent?
```

**Load documents:**
```sh
# Local
docker exec -it agentos-aws-template-api python -m agents.knowledge_agent

# AWS (SSH into container)
ECS_CLUSTER=agentos-aws-template-prd-cluster
TASK_ARN=$(aws ecs list-tasks --cluster $ECS_CLUSTER --query "taskArns[0]" --output text)
aws ecs execute-command --cluster $ECS_CLUSTER --task $TASK_ARN \
    --container agentos-aws-template --interactive --command "python -m agents.knowledge_agent"
```

### MCP Agent

Connects to external tools via the Model Context Protocol.

**Try it:**
```
What tools do you have access to?
Search the docs for how to use LearningMachine
```

---

## Project Structure

```
├── agents/
│   ├── pal.py              # Personal second brain agent
│   ├── knowledge_agent.py  # RAG agent
│   └── mcp_agent.py        # MCP tools agent
├── app/
│   ├── main.py             # AgentOS entry point
│   └── config.yaml         # Quick prompts config
├── db/
│   ├── session.py          # PostgreSQL helpers
│   └── url.py              # Connection URL builder
├── infra/
│   ├── dev_resources.py    # Local Docker configuration
│   ├── prd_resources.py    # AWS infrastructure
│   └── settings.py         # Shared settings (edit this for AWS)
├── scripts/
│   ├── auth_ecr.sh         # ECR authentication
│   └── ...
├── Dockerfile
└── pyproject.toml          # Dependencies
```

---

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
    model=OpenAIResponses(id="gpt-4o"),
    db=get_postgres_db(),
    instructions="You are a helpful assistant.",
)
```

2. Register in `app/main.py`:

```python
from agents.my_agent import my_agent

agent_os = AgentOS(
    agents=[pal, knowledge_agent, mcp_agent, my_agent],
    ...
)
```

3. Rebuild: `ag infra up --env dev -y`

### Add tools to an agent

Agno includes 100+ tool integrations. See the [full list](https://docs.agno.com/tools/toolkits).

```python
from agno.tools.slack import SlackTools
from agno.tools.google_calendar import GoogleCalendarTools

my_agent = Agent(
    ...
    tools=[SlackTools(), GoogleCalendarTools()],
)
```

### Add dependencies

1. Edit `pyproject.toml`
2. Regenerate requirements: `./scripts/generate_requirements.sh`
3. Rebuild: `ag infra up --env dev -y`

### Use a different model provider

1. Add your API key to `.env` (e.g., `ANTHROPIC_API_KEY`)
2. Update agents:

```python
from agno.models.anthropic import Claude

model=Claude(id="claude-sonnet-4-5")
```

3. Add dependency to `pyproject.toml`: `anthropic`

---

## Local Development (Without Docker)

```sh
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Setup environment
./scripts/venv_setup.sh
source .venv/bin/activate

# Start PostgreSQL (still needs Docker)
ag infra up --env dev

# Run app directly (with hot reload)
python -m app.main
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes | - | OpenAI API key |
| `EXA_API_KEY` | No | - | Exa API key (enables Pal's web research) |
| `DB_HOST` | No | `localhost` | Database host |
| `DB_PORT` | No | `5432` | Database port |
| `DB_USER` | No | `ai` | Database user |
| `DB_PASS` | No | `ai` | Database password |
| `DB_DATABASE` | No | `ai` | Database name |
| `DATA_DIR` | No | `/data` | DuckDB storage directory |
| `RUNTIME_ENV` | No | `prd` | Set to `dev` for auto-reload |

---

## Learn More

- [Managing AWS Resources](https://docs.agno.com/production/aws/production-app)
- [HTTPS Setup Guide](https://docs.agno.com/production/aws/domain-https)
- [Agno Documentation](https://docs.agno.com)
- [AgentOS Documentation](https://docs.agno.com/agent-os/introduction)
- [Tools & Integrations](https://docs.agno.com/tools/toolkits)
- [Discord Community](https://agno.link/discord)
