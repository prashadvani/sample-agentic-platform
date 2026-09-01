# 0002 — Bedrock model IDs are centralized in one module

- **Status:** Accepted
- **Date:** 2025

## Context

Bedrock model IDs are long, versioned strings (e.g.
`us.anthropic.claude-sonnet-4-5-20250929-v1:0`). Two forms are needed: the region-prefixed
form for direct Bedrock calls, and the non-prefixed `model_name` the LiteLLM proxy routes on.

Models get deprecated. AWS marks older models "Legacy" and eventually denies access — a model
that worked last month can start returning `AccessDenied` with no code change. If each agent
hardcodes its own model string, a deprecation means hunting through every agent to update it,
and clones of the repo silently break.

## Decision

Keep all model IDs in a single module, `core/models/model_config.py`, exposing named
constants (`SONNET_MODEL_ID`, `HAIKU_MODEL_ID`, `NOVA_LITE_MODEL_ID`, and the
`*_LITELLM_MODEL_ID` proxy-name variants). Agents import these constants; they do not write
model strings inline. When a model is deprecated, it is bumped in exactly one place.

## Consequences

**Positive**
- One-line, one-file change to move every agent off a deprecated model — the "no magic
  numbers / single source of truth" principle applied to model IDs.
- The two-form convention (direct vs. proxy name) is documented in one place, so the
  distinction can't be misapplied per-agent.

**Negative / trade-offs**
- Agents that genuinely need a different model must intentionally override the constant,
  which is a small amount of extra ceremony.
- The file must actually be kept current; a stale default here is exactly the failure mode
  that surfaces as a runtime `AccessDenied`. (This repo shipped with a since-dormant Sonnet
  default — evidence the mechanism only helps if the value is maintained.)

## Related

- ADR [0001](0001-llm-access-through-gateway.md) — the gateway consumes these IDs.
