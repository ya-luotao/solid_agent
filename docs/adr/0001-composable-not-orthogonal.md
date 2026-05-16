# ADR-0001: Composable, not orthogonal

- **Status:** Accepted
- **Date:** 2026-05-15
- **Driven by:** [Issue #1, P0-1](https://github.com/ya-luotao/solid_agent/issues/1)

## Context

Early drafts of `DESIGN.md` described the Agent and Runtime (and at one point Workspace) axes as "orthogonal." A careful reviewer pushed back: this is provably false.

- Claude Code's `Read/Edit/Bash/TodoWrite` tools are baked into its training and prompts. Changing the runtime's shape — e.g., removing the shell, or substituting a virtual filesystem — would require re-training or significant prompt rework. The Agent's behaviour is not invariant to the Runtime.
- Hypothetical Workspace capabilities like `fs.overlay` or `audit.tool_calls` only have value if the Agent knows about them and uses them. The reverse coupling is also tight.
- ACP itself uses `agentCapabilities` and `clientCapabilities` handshakes precisely because the protocol expects coupling and needs explicit negotiation rather than independence.

Calling the axes "orthogonal" is the kind of claim a serious reader will reject on first read, which undermines the credibility of the whole spec.

## Decision

Replace "orthogonal" framing with **"composable concerns negotiated via explicit capabilities"** throughout the documentation.

- The Agent and Runtime axes are independent in the sense that adapters on each side can be developed in isolation.
- They are **not** independent in the sense that any (Agent, Runtime) pair will work — they couple through capability declarations and a negotiation handshake at session creation.
- The protocol's job is to make this coupling explicit, declared, and version-stable rather than implicit and fragile.

## Consequences

**Intended:**

- `DESIGN.md §1` reframed as "The composition problem" with explicit "composable, not orthogonal" language.
- `README.md` hero text updated to match.
- The capability registry and negotiation logic in `PROTOCOL.md §4` get more prominent positioning, since they are now load-bearing rather than ornamental.

**Accepted downsides:**

- The phrase "two-axis composition with capability negotiation" is wordier than "two orthogonal axes." We accept this for accuracy.
- Implementers cannot assume any Agent + Runtime combination will work — they must check capability negotiation. (This is correct behaviour, but the docs need to make it visible.)

## Alternatives considered

- **Keep "orthogonal"**, rely on prose to qualify it. Rejected — the framing is the first thing a reader sees in the architecture diagram, and qualifying it later doesn't stick.
- **Drop the multi-axis framing entirely**, treat Solid Agent as a single monolithic spec. Rejected — the two-axis decomposition is the most valuable thing the spec adds; we just need to describe it accurately.
