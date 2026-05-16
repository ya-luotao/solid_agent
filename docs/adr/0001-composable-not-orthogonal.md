# ADR-0001: Composable, not orthogonal

- **Status:** Accepted
- **Date:** 2026-05-15

## Context

Early drafts described the Agent and Runtime axes as "orthogonal." This is wrong and a careful reader will reject it.

- Claude Code's tool schema (`Read`, `Edit`, `Bash`, `TodoWrite`) is baked into its training. A runtime that drops the shell or replaces the filesystem with something exotic breaks Claude Code, full stop.
- Capabilities like `snapshot` only matter if the Host knows about them and uses them. The reverse coupling is real.
- ACP itself uses `agentCapabilities` and `clientCapabilities` handshakes precisely because the protocol expects coupling and needs explicit negotiation rather than independence.

Calling the axes "orthogonal" is the kind of claim a serious reader will reject on first read, which undermines the credibility of the whole spec.

## Decision

Replace "orthogonal" framing with **"composable with explicit capability negotiation"** throughout the documentation.

- The Agent and Runtime axes are independent in the sense that adapters on each side can be developed in isolation.
- They are **not** independent in the sense that any (Agent, Runtime) pair will work — they couple through capability declarations and a negotiation handshake at session creation.
- The protocol's job is to make this coupling explicit, declared, and version-stable rather than implicit and fragile.

## Consequences

**Intended:**

- `DESIGN.md` reframed as "Composable, not orthogonal" with an explicit section on capability negotiation.
- `README.md` hero text updated to match.
- The capability registry and negotiation logic in `PROTOCOL.md` get prominent positioning, since they are load-bearing rather than ornamental.

**Accepted downsides:**

- The phrase "composable with capability negotiation" is wordier than "orthogonal." We accept this for accuracy.
- Implementers cannot assume any Agent + Runtime combination will work — they must check capability negotiation. (This is correct behaviour, but the docs need to make it visible.)

## Alternatives considered

- **Keep "orthogonal", rely on prose to qualify it.** Rejected — the framing is the first thing a reader sees in the architecture diagram, and qualifying it later doesn't stick.
- **Drop the multi-axis framing entirely.** Rejected — the two-axis decomposition is the most valuable thing the spec adds; we just need to describe it accurately.
