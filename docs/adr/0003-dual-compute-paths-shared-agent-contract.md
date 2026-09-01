# 0003 — Two compute paths share one agent contract

- **Status:** Accepted
- **Date:** 2025

## Context

Customers land in different operational places. Some want full control of a Kubernetes
cluster; others want minimal operational overhead. Building the platform for only one of
those leaves half the audience unserved, and forcing agents to know where they run would
couple business logic to infrastructure.

## Decision

Support **two compute paths** on top of a shared **Foundation** stack, and make every agent
conform to a single contract so the same agent code runs on either:

- **Option A — EKS** (`platform-eks`): agents + services on Kubernetes via Helm; the LiteLLM
  gateway runs in-cluster.
- **Option B — AgentCore** (`platform-agentcore` + `agentcore-runtime`): LiteLLM on ECS
  Fargate; agents run in Bedrock AgentCore Runtime.

**The contract:** every agent is a FastAPI app listening on port `8080` and exposing
`/invocations` (verified across the agent implementations). Because all agents honor it, the
deployment target is a deployment decision, not a code decision.

## Consequences

**Positive**
- Moving an agent between EKS and AgentCore is a **mostly two-way door**: app code is
  unchanged; only the surrounding IaC/deploy differs.
- Lets the customer pick control (EKS) vs. low operational overhead (AgentCore) without a
  fork of the agent code — and the cost floors differ (see [COST_ANALYSIS.md](../COST_ANALYSIS.md)).

**Negative / trade-offs**
- Two deployment paths mean two sets of IaC and Helm/stack wiring to maintain and test.
- The shared contract constrains agents (`:8080` + `/invocations`), a deliberate limit that
  keeps portability intact.
- The abstraction stops at the app boundary: the migration *cost* between paths is the
  infrastructure and ops around the agent, not the agent itself.

## Related

- ADR [0001](0001-llm-access-through-gateway.md) — the gateway runs on whichever path is chosen.
