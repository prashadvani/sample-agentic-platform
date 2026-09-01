# 0001 — LLM access goes through a gateway behind an interface

- **Status:** Accepted
- **Date:** 2025

## Context

Agents need to call LLMs. The naive approach is for each agent to call `bedrock-runtime`
(or a vendor SDK) directly. That hardcodes the provider, the model, the auth path, and the
request/response shape into every agent. It also makes the single most expensive component
at scale (see [COST_ANALYSIS.md](../COST_ANALYSIS.md)) the hardest thing to change.

The platform also supports multiple agent frameworks (LangGraph, PydanticAI, Strands, and a
DIY path), each of which has its own message format.

## Decision

Route all LLM traffic through a **LiteLLM gateway**, and have agents talk to it through the
core client and converter layer rather than a vendor SDK:

- `core/client/llm_gateway/` — gateway client the agents call.
- `core/converter/` — per-framework converters (`langchain_converters.py`,
  `pydanticai_converters.py`, `strands_converters.py`, `litellm_converters.py`) that adapt
  each framework's messages to/from the shared `AgenticRequest`/`AgenticResponse` models.

Agents depend on the interface, not on Bedrock.

## Consequences

**Positive**
- Swapping models, enabling prompt caching/batch, or routing cheap turns to a smaller model
  is a **two-way door** — a gateway/config change, not an agent rewrite.
- A self-hosted model can be introduced behind the same gateway if per-token pricing stops
  making sense at very high volume, without touching agent code.
- Framework interop: the same core models flow through any supported framework.

**Negative / trade-offs**
- The gateway is another service to run (in-cluster on EKS, or Fargate on AgentCore) and a
  potential single point of failure / latency hop.
- Model swaps are only *safe* two-way doors if **evals** exist to confirm quality holds; the
  abstraction enables the swap but does not validate it.

## Related

- ADR [0002](0002-centralized-model-config.md) (model IDs) and
  [0003](0003-dual-compute-paths-shared-agent-contract.md) (compute paths).
