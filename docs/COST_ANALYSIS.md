# Cost Analysis

Estimated monthly cost to run the sample-agentic-platform at three scales, per compute
path. The goal is not a single "what does it cost today" number — it is to show **where
the cost curve bends** and **which components can be swapped** if cost becomes a problem.

> ⚠️ **The dollar figures below are ESTIMATE PLACEHOLDERS.** They are order-of-magnitude
> starting points, not a quote. Before using this with a customer, replace the rate cells
> in [Rate assumptions](#rate-assumptions) with **current** prices from the live pricing
> pages ([EC2](https://aws.amazon.com/ec2/pricing/on-demand/),
> [EKS](https://aws.amazon.com/eks/pricing/),
> [RDS Aurora](https://aws.amazon.com/rds/aurora/pricing/),
> [ElastiCache](https://aws.amazon.com/elasticache/pricing/),
> [Fargate](https://aws.amazon.com/fargate/pricing/),
> [VPC/NAT](https://aws.amazon.com/vpc/pricing/),
> [Bedrock](https://aws.amazon.com/bedrock/pricing/)) for your target region, then let the
> per-service tables recompute. Build this in a spreadsheet so you can adjust live in the
> room (e.g. "what if 50k users but only 60% active?").

Region assumed: **us-west-2**. Prices exclude taxes, data-transfer-out beyond NAT, and any
committed-use discounts (Savings Plans / Reserved Instances / provisioned Bedrock).

---

## Scale definitions

| Scale | Users | Assumption | Requests/month |
|-------|-------|------------|----------------|
| **Pilot** | 100 | 20 agent requests/user/day, 22 working days | ~44,000 |
| **Production** | 10,000 | 20 req/user/day, 30 days | ~6,000,000 |
| **Over-production (10×)** | 100,000 | 20 req/user/day, 30 days | ~60,000,000 |

### LLM usage assumptions (the dominant variable cost)

| Assumption | Value | Notes |
|------------|-------|-------|
| Tokens per request (input) | 1,500 | prompt + retrieved context + history |
| Tokens per request (output) | 500 | typical chat/agent turn |
| Model | Claude Sonnet-class | agents default via `core/models/model_config.py` |
| Prompt caching | not assumed | enabling it can cut input cost substantially |

> These four numbers drive the entire Bedrock line. Change them first when the customer
> pushes on the model.

---

## Rate assumptions

**⚠️ VERIFY AND REPLACE these before presenting.** Values are placeholders.

| Resource | Unit | Rate (PLACEHOLDER) |
|----------|------|--------------------|
| EKS control plane | per cluster-hour | ~$0.10 |
| EC2 `t3.medium` (node) | per hour | ~$0.042 |
| EC2 `t3.large` (bastion) | per hour | ~$0.083 |
| Aurora PostgreSQL `db.t4g.medium` | per instance-hour | ~$0.073 |
| ElastiCache `cache.t4g.micro` | per node-hour | ~$0.016 |
| ElastiCache `cache.t3.micro` | per node-hour | ~$0.017 |
| Fargate vCPU | per vCPU-hour | ~$0.04048 |
| Fargate memory | per GB-hour | ~$0.004445 |
| NAT Gateway | per hour | ~$0.045 |
| NAT Gateway data processing | per GB | ~$0.045 |
| Application Load Balancer | per hour | ~$0.0225 (+ LCU) |
| S3 Standard | per GB-month | ~$0.023 |
| Bedrock Claude Sonnet input | per 1M tokens | **VERIFY** (~$3 order of magnitude) |
| Bedrock Claude Sonnet output | per 1M tokens | **VERIFY** (~$15 order of magnitude) |

Hours/month assumed: **730**.

---

## What actually gets deployed

Pulled from the Terraform defaults (`infrastructure/stacks/*/variables.tf`), so the
fixed-cost lines match the repo rather than a generic guess.

**Foundation stack (both paths):** VPC, subnets, **2× NAT Gateway**, security groups.

**Option A — Platform EKS** (`platform-eks`):
- EKS control plane (1)
- Node group: `t3.medium`, desired **4** (min 2 / max 6), 20 GB disk each
- Aurora PostgreSQL `db.t4g.medium`
- ElastiCache Redis `cache.t4g.micro` × **2**
- Application Load Balancer
- Bastion `t3.large`
- LiteLLM gateway runs **in-cluster** (no extra compute line)

**Option B — Platform AgentCore** (`platform-agentcore`):
- ECS Fargate LiteLLM task: **1 vCPU / 2 GB**, 1 task (autoscale 1→3)
- Aurora PostgreSQL `db.t4g.medium`
- ElastiCache Redis `cache.t3.micro` × **2**
- Bastion `t3.large`
- Agents run in **Bedrock AgentCore Runtime** (managed; consumption-priced — verify separately)

**Shared:** S3, KMS, Cognito, Secrets Manager (all low fixed cost at these scales).

---

## Per-service monthly cost — Option A (EKS)

Fixed infrastructure is **flat across scale**; only Bedrock and node count grow.

| Service | Cost driver | Pilot | Production | Over-prod (10×) |
|---------|-------------|-------|-----------|-----------------|
| EKS control plane | per cluster-hour | ~$73 | ~$73 | ~$73 |
| EC2 nodes (`t3.medium`) | per node-hour × count | ~$123 (4) | ~$184 (6, capped) | **node cap hit → must raise `max_size`** |
| Aurora `db.t4g.medium` | per instance-hour | ~$53 | ~$53 | ~$53 (needs scale-up) |
| ElastiCache ×2 | per node-hour | ~$23 | ~$23 | ~$23 (needs scale-up) |
| ALB | per hour + LCU | ~$16+ | ~$16+ | ~$16+ |
| 2× NAT Gateway | per hour + per GB | ~$66 + data | ~$66 + data | ~$66 + data |
| Bastion `t3.large` | per hour | ~$61 | ~$61 | ~$61 |
| S3 / KMS / Cognito / Secrets | mixed | ~$5 | ~$10 | ~$30 |
| **Fixed subtotal** | | **~$420** | **~$486** | **~$375 + scale-ups** |
| **Bedrock tokens** | per 1M tokens | **VERIFY** | **VERIFY** | **VERIFY** |

## Per-service monthly cost — Option B (AgentCore / ECS)

| Service | Cost driver | Pilot | Production | Over-prod (10×) |
|---------|-------------|-------|-----------|-----------------|
| Fargate LiteLLM (1 vCPU/2 GB) | vCPU-hr + GB-hr | ~$36 (1 task) | ~$72–108 (2–3 tasks) | autoscale cap 3 → raise `max_capacity` |
| Aurora `db.t4g.medium` | per instance-hour | ~$53 | ~$53 | ~$53 (needs scale-up) |
| ElastiCache ×2 | per node-hour | ~$25 | ~$25 | ~$25 (needs scale-up) |
| 2× NAT Gateway | per hour + per GB | ~$66 + data | ~$66 + data | ~$66 + data |
| Bastion `t3.large` | per hour | ~$61 | ~$61 | ~$61 |
| AgentCore Runtime | consumption | **VERIFY** | **VERIFY** | **VERIFY** |
| S3 / KMS / Cognito / Secrets | mixed | ~$5 | ~$10 | ~$30 |
| **Fixed subtotal** | | **~$246** | **~$290+** | **scale-ups** |
| **Bedrock tokens** | per 1M tokens | **VERIFY** | **VERIFY** | **VERIFY** |

---

## Where the curve bends

1. **Bedrock dominates at scale.** Fixed infra is a few hundred dollars/month and barely
   moves. At production/10× the token bill is almost the entire cost and scales **linearly**
   — there is no economy of scale in pay-per-token. If cost must drop at high volume, the
   *model layer* is what changes, not the servers.
2. **Fixed floor makes the pilot look expensive per user.** At 100 users, ~$250–420/month of
   always-on infra dwarfs the token spend. AgentCore (Option B) has the lower floor because
   it has no EKS control-plane or ALB charge and scales compute to near-zero.
3. **Hard caps to watch:** EKS `max_size = 6` nodes and Fargate `max_capacity = 3` tasks are
   the first ceilings you hit before the 10× column is real — both are `terraform.tfvars`
   changes, not code.
4. **NAT Gateway data processing** is an easy surprise: every byte agents send to Bedrock and
   third-party APIs egresses through NAT at ~$0.045/GB on top of the hourly charge.

---

## Cost decisions: one-way vs. two-way doors

For each component that gets expensive at scale, how hard is it to swap?

| Component | Door | Why | Swap path if cost bites |
|-----------|------|-----|-------------------------|
| **LLM / Bedrock model** | 🟢 Two-way | Calls go through the **LiteLLM gateway** + a converter interface; model IDs are centralized in `core/models/model_config.py`. Agents don't hardcode a model. | Switch model, enable prompt caching/batch, route cheap turns to Haiku/Nova, or point the gateway at a self-hosted model — **with robust evals** to confirm quality holds. |
| **Self-host vs. API crossover** | 🟢 Two-way | Same gateway abstraction. | At sustained very-high volume, a self-hosted model behind the gateway can beat per-token pricing — trades operational simplicity for unit cost. The gateway makes this a config/routing change, not a rewrite. |
| **Compute path (EKS ↔ AgentCore)** | 🟡 Mostly two-way | Every agent implements the same **AgentCore contract** (`:8080`, `/invocations`), so the same agent code runs on either path. | Move between EKS and AgentCore by deploying the other stack; app code is unchanged. The IaC/ops around it is the migration cost. |
| **Aurora ↔ serverless / provisioned** | 🟡 Two-way-ish | Postgres accessed via `core/db`; instance class is a variable. | Bump `postgres_instance_class`, or move to Aurora Serverless v2 for spiky load. Config change; validate connection pooling. |
| **ElastiCache node type** | 🟢 Two-way | `redis_node_type` is a variable. | Resize via tfvars. |
| **NAT Gateway topology** | 🟡 Two-way | Foundation stack. | Consolidate to 1 NAT (loses AZ redundancy) or add VPC endpoints for AWS-bound traffic to cut data-processing cost. |
| **Bastion host** | 🟢 Two-way | Single `t3.large`. | Replace with SSM Session Manager (no always-on instance) — removes a ~$61/month floor line. |

**The takeaway for the customer:** the expensive-at-scale component (the LLM) is behind an
interface, so the highest-leverage cost lever is a **two-way door**. Nothing here forces a
rewrite to change cost posture — the decisions that matter are swappable, provided evals
exist to validate a model change.

---

## How to use this in a live conversation

1. Put the [Rate assumptions](#rate-assumptions) and [LLM usage assumptions](#llm-usage-assumptions-the-dominant-variable-cost)
   in a spreadsheet as input cells.
2. Wire the per-service tables to formulas referencing those cells.
3. When the customer changes an assumption ("60% active", "cache 40% of prompts", "use
   Haiku for routing turns"), edit the input cell and the totals move in real time.

---

*Generated as a starting template. Figures are placeholders — verify against live AWS
pricing for the target region before sharing externally.*
