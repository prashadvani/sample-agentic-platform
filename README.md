# Agentic Platform

A sample of what an agentic platform might look like on AWS. The repo demonstrates multiple compute layer options, a shared core library, gateway services, and a collection of agents built with different frameworks — all wired together as a working reference architecture.

It's organized as a **monorepo** so that coding agents (Claude Code, Kiro, Cursor, etc.) can see the full picture in one place. A tree of `AGENTS.md` files helps these tools navigate the codebase — the root [AGENTS.md](AGENTS.md) acts as a table of contents pointing to domain-specific guides for [infrastructure](infrastructure/AGENTS.md), [Kubernetes](k8s/AGENTS.md), [application code](src/agentic_platform/AGENTS.md), [bootstrap](bootstrap/AGENTS.md), [tests](tests/AGENTS.md), and [labs](labs/AGENTS.md). The `CLAUDE.md` file simply says "see AGENTS.md" so the same scaffolding works across different tools.

## Project Status

This sample is actively maintained. Contributions welcome — see [Contributing](#contributing).

## Architecture at a Glance

The platform is organized into layered infrastructure stacks that you compose based on your needs:

```
┌─────────────────────────────────────────────────────────────┐
│                      Applications                           │
│  Agents · MCP Servers · Memory Gateway · Retrieval Gateway  │
├──────────────┬──────────────────────────────┬───────────────┤
│   Option A   │                              │   Option B    │
│  Kubernetes  │      Knowledge Layer         │  AgentCore    │
│  (EKS +      │  (Bedrock Knowledge Base)    │  (ECS +       │
│   Helm)      │                              │   AgentCore   │
│              │                              │   Runtime)    │
├──────────────┴──────────────────────────────┴───────────────┤
│                    Foundation Stack                          │
│          VPC · Subnets · NAT · Security Groups              │
└─────────────────────────────────────────────────────────────┘
```

### Infrastructure Stacks

| Stack | Path | Purpose |
|-------|------|---------|
| **Foundation** | `infrastructure/stacks/foundation/` | Networking (VPC, subnets, NAT gateways, security groups). Deployed first — everything else builds on top. |
| **Platform EKS** | `infrastructure/stacks/platform-eks/` | EKS cluster, IRSA roles, Cognito, Aurora PostgreSQL, ElastiCache, bastion host, and the LiteLLM gateway deployed in-cluster. |
| **Platform AgentCore** | `infrastructure/stacks/platform-agentcore/` | "Lite" compute layer using ECS/Fargate. Deploys the LiteLLM gateway in ECS along with Cognito, Aurora, ElastiCache, and a bastion host. |
| **AgentCore Runtime** | `infrastructure/stacks/agentcore-runtime/` | Deploys individual agents into Amazon Bedrock AgentCore Runtime. Uses **Terraform workspaces** — one workspace per agent — to minimize duplicated infrastructure code. |
| **Knowledge Layer** | `infrastructure/stacks/knowledge-layer/` | Sets up a Bedrock Knowledge Base for RAG. Plans to evolve into a more sophisticated retrieval layer as more examples are added. |

### Choosing a Compute Layer

You deploy the **Foundation** stack first, then pick one of two paths:

**Option A — Kubernetes (EKS):** Full control. Deploy the `platform-eks` stack, then deploy agents and services via Helm charts under `k8s/`. The LLM gateway, telemetry collectors, and other platform shims run in the cluster alongside your agents.

**Option B — AgentCore (ECS + Managed Runtime):** Less operational overhead. Deploy the `platform-agentcore` stack to get the LLM gateway running in ECS, then deploy agents into AgentCore Runtime using the `agentcore-runtime` stack (one Terraform workspace per agent).

Both paths share the same agent code. Every agent implements the **AgentCore interface** (port `8080`, `/invocations` endpoint), so it's up to you where you deploy.

### Lifecycle Separation

The platform distinguishes between two lifecycles:

- **Platform infrastructure** (LLM gateway, telemetry, auth shims) — changes infrequently, managed by the platform team.
- **Application code** (agents, MCP servers, supporting services) — changes frequently, deployed independently by application teams.

This separation is reflected in the stack design and the Helm chart layout.

## Architecture Diagrams

### EKS-Based Architecture
![High Level Architecture](media/highlevel-architecture.png)

### Agent Process Architecture
![Agent Process Architecture](media/agent-design.png)

Each agent runs as a FastAPI server sharing a core package with types, client abstractions, and middleware. Agents don't hold IAM roles directly — they connect to AWS resources through gateway services (LLM Gateway, Memory Gateway, Retrieval Gateway) that use IRSA. All inter-service communication is authenticated via JWT tokens validated against the IDP's public cert. Telemetry is collected via OpenTelemetry and pushed to X-Ray, CloudWatch, and OpenSearch.

### AgentCore Architecture
![AgentCore Architecture](media/agentcore-arch.png)

A managed approach using Amazon Bedrock AgentCore primitives with ECS for the gateway layer and AgentCore Runtime for agent execution.

## Repository Layout

| Directory | What's in it |
|-----------|-------------|
| `infrastructure/` | Terraform stacks and 20+ reusable modules |
| `k8s/` | Helm charts and per-application values for EKS deployments |
| `src/agentic_platform/` | Agent implementations, shared core library, gateway services, MCP servers, and tools |
| `labs/` | 5 learning modules with 25+ Jupyter notebooks |
| `bootstrap/` | CloudFormation templates for bootstrapping AWS accounts |
| `deploy/` | Build and deploy scripts |
| `tests/` | Unit and integration tests |

## Agents

Sample agents built with different frameworks — all sharing the same interface (port `8080`, `/invocations`, `/health`):

| Agent | Framework | Description |
|-------|-----------|-------------|
| `agentic_chat` | Strands | General-purpose chat agent |
| `agentic_rag` | Strands | RAG agent backed by Bedrock Knowledge Base |
| `langgraph_chat` | LangGraph | Chat agent using LangGraph workflows |
| `jira_agent` | Strands | Jira integration agent |
| `strands_glue_athena` | Strands | AWS Glue & Athena data agent |

See [AGENTS.md](AGENTS.md) for the full development guide.

## Labs

Five progressive modules that take you from fundamentals to production:

| Module | Topic | What You'll Learn |
|--------|-------|-------------------|
| **1** | Prompt Engineering & Evaluation | Bedrock Converse API, chain-of-thought, few-shot, RAG basics, function calling, evaluation |
| **2** | Common Agentic Patterns | Prompt chaining, routing, parallelization, orchestration, evaluator-optimizer |
| **3** | Building Agentic Applications | Agent memory, tools, retrieval, framework interoperability |
| **4** | Multi-Agent Systems & MCP | MCP servers & clients, multi-agent delegation, agent graphs, AgentCore tools |
| **5** | Deployment & Infrastructure | OpenTelemetry, LLM gateway, memory gateway, streaming, scaling |

Only Module 5 requires the platform to be deployed. To run labs locally:

```bash
uv sync
uv run jupyter lab
```

See [labs/README.md](labs/README.md) for detailed instructions.

## Getting Started

### Local Development Quickstart

**Prerequisites:**

- [Python 3.12.8](https://www.python.org/) — the exact version is pinned in `pyproject.toml` (`requires-python = "==3.12.8"`). `uv` will install it for you (see below).
- [uv](https://github.com/astral-sh/uv) — package manager. This project uses a `uv` workspace (`labs-common` is a workspace member), so `pip` will not work; use `uv`.
- [Docker](https://docs.docker.com/engine/install/) — must be **running** before `make dev:deps`.
- **AWS credentials with Amazon Bedrock access** in your target region (default `us-west-2`). The LiteLLM gateway calls Bedrock using the credentials in your `~/.aws` directory, which is mounted into the gateway container. You must have **enabled model access** for the models you intend to call (see step 4 and the troubleshooting note below) — a brand-new account has not, and calls will fail with `AccessDenied` until you do.

```bash
# 1. Clone
git clone https://github.com/aws-samples/sample-agentic-platform.git
cd sample-agentic-platform

# 2. Install the pinned Python and all dependencies (creates .venv)
uv python pin 3.12.8   # downloads CPython 3.12.8 if you don't have it
uv sync                # installs deps, including the labs-common workspace package

# 3. Create the local environment file (REQUIRED — `make dev:deps` fails without it)
cp .env.example .env   # then edit .env and fill in the values below

# 4. Make sure your AWS credentials are available and Bedrock model access is enabled
#    (the gateway reads ~/.aws; SSO users must have a valid session, e.g. `aws sso login`)

# 5. Start supporting services (Postgres, Redis, LiteLLM, Memory Gateway). Needs Docker running.
make dev:deps

# 6. Run an agent (this BLOCKS — it's a --reload server; leave it running in this terminal)
make dev agentic_chat
#    -> INFO:  Uvicorn running on http://127.0.0.1:8080

# Stop supporting services when done
make dev:deps-stop
```

**Required `.env` values** (local dev; see `.env.example` for the full list):

| Variable | Purpose | Local value |
|----------|---------|-------------|
| `ENVIRONMENT` | Selects local connection paths | `local` |
| `AWS_DEFAULT_REGION` | Bedrock region | e.g. `us-west-2` |
| `PG_*` | Postgres connection | match `docker-compose.yaml` (`dev`/`dev`/`devdb`) |
| `LITELLM_MASTER_KEY` | Protects the LiteLLM proxy; also the key clients send | any string, e.g. `sk-local-dev-1234` |
| `LITELLM_DATABASE_URL` | LiteLLM's metadata DB | `postgresql://dev:dev@postgres:5432/devdb` |

**Test the running agent** (from a **second** terminal — the agent occupies the first):

```bash
curl -X POST http://localhost:8080/api/agentic-chat/invocations \
  -H "Content-Type: application/json" \
  -d '{"message":{"role":"user","content":[{"type":"text","text":"hello"}]},"session_id":"test"}'
```

The request body is a single `AgenticRequest`: one `message` object whose `content` is a list of typed blocks (not a `messages` array and not a bare string).

**Run an MCP server instead:**

```bash
make dev:mcp bedrock_kb_mcp_server
```

Run `make help` to see all available commands and the list of available agents.

### Troubleshooting local dev

| Symptom | Cause | Fix |
|---------|-------|-----|
| `env file ... /.env not found` on `make dev:deps` | No root `.env` | `cp .env.example .env` and fill it in (step 3) |
| `make: *** No rule to make target 'dev'` | Running `make` outside the repo root | `cd` into the cloned repo first |
| Agent returns `Internal Server Error` (500) | LiteLLM can't reach Bedrock | Check `docker compose logs litellm`. Usually missing/expired AWS credentials or Bedrock model access not enabled. |
| litellm log: `'NoneType' object has no attribute 'access_key'` | No AWS credentials visible to the gateway | Ensure `~/.aws` has valid credentials; SSO users run `aws sso login` and recreate: `docker compose up -d --force-recreate litellm` |
| Bedrock error: model marked **Legacy** / `AccessDenied` | The requested model is dormant or not enabled in your account | Enable model access in the Bedrock console, or use an active model (Claude Sonnet 4.5, Haiku 4.5, Nova). Model IDs live in `src/agentic_platform/core/models/model_config.py`. |

### Deploying to AWS

See [DEPLOYMENT.md](DEPLOYMENT.md).

### Teardown

**Local:** stop the supporting containers when you're done. Add `-v` to also delete the Postgres/Redis volumes if you want a clean slate.

```bash
make dev:deps-stop        # stop containers (data volumes preserved)
docker compose down -v    # optional: also remove postgres_data / redis_data volumes
```

**AWS:** destroy the deployed stacks to avoid ongoing charges (orphaned EKS/Aurora/NAT resources cost money). Full teardown order and commands are in [DEPLOYMENT.md → Teardown](DEPLOYMENT.md#teardown) — in short, `terraform destroy` each stack in reverse order (agents/runtime first, Foundation last).

## Security

Run security scans before submitting changes:

- [Checkov](https://www.checkov.io/2.Basics/Installing%20Checkov.html)
- [Bandit](https://bandit.readthedocs.io/en/latest/)
- [Gitleaks](https://github.com/gitleaks/gitleaks)

**Suppressed Warnings**: Review suppressed warnings in the codebase before using any code in your environment.

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## Contributing

We welcome contributions. Items on the roadmap include:

1. Improving the bootstrap experience
2. Structured GitOps for deployments
3. Additional labs on advanced agent topics
4. Test harness & eval suite
5. More agent examples from the labs into the sample platform
6. Expanding the knowledge/retrieval layer

## Authors

- Tanner McRae
- Randy DeFauw
- James Levine

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
