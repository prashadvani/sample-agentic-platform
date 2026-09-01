# Architecture Decision Records (ADRs)

These records capture **why** the platform looks the way it does, so a future engineer can
tell intentional decisions from accidents. Each ADR is short: context, the decision, and the
consequences (including trade-offs).

Format: [MADR](https://adr.github.io/madr/)-lite. Status is one of `Proposed`, `Accepted`,
`Superseded`.

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-llm-access-through-gateway.md) | LLM access goes through a gateway behind an interface | Accepted |
| [0002](0002-centralized-model-config.md) | Bedrock model IDs are centralized in one module | Accepted |
| [0003](0003-dual-compute-paths-shared-agent-contract.md) | Two compute paths share one agent contract | Accepted |

To add one: copy the structure of an existing ADR, take the next number, and add a row above.
