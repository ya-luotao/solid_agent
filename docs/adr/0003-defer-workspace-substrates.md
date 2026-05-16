# ADR-0003: Defer workspace substrate capabilities to future SAP versions

- **Status:** Accepted
- **Date:** 2026-05-15
- **Driven by:** [Issue #1, P0-3 (90-day demo risk)](https://github.com/ya-luotao/solid_agent/issues/1) plus project-lead directive to focus v0.1 on the Agent ↔ Runtime contract.

## Context

Earlier drafts of `DESIGN.md` and `PROTOCOL.md` promoted "Workspace" — the agent's persistent operating substrate — to a first-class architectural concern alongside Agent and Runtime, with four substrates (filesystem, shell, memory, audit) and a richer capability surface (overlays, mounts, in-process shells, memory KV, audit logs).

The motivation was real: projects like [AgentFS](https://github.com/tursodatabase/agentfs), [Mirage](https://github.com/strukto-ai/mirage), and [just-bash](https://github.com/vercel-labs/just-bash) demonstrate that agent workspaces benefit from substrates richer than POSIX. The argument that "an agent's filesystem differs in kind from a human's" is defensible.

However, two pressures converged:

1. **The reviewer's P0-3 concern** about spec-first death — projects that ship a maximal spec without a working reference implementation become zombie repos. The minimum viable demo (Phase 2) needs to fit in 90 days, and integrating AgentFS / Mirage / just-bash as part of v0.1 risks slipping that deadline.
2. **The project lead's directive** to scope v0.1 to Agent SDK wrapping + Runtime support, deferring filesystem substrate concerns. The reasoning: the Agent ↔ Runtime decoupling is the load-bearing innovation; workspace substrate is a downstream concern that benefits from real usage data before being specified.

## Decision

**Workspace substrate capabilities are out of scope for SAP v0.1.** They are parked in [`FUTURE.md`](../../FUTURE.md) with full provenance and design sketches, and may return as RFCs once v0.1 has a reference implementation and accumulated usage data.

Specifically deferred:

- Filesystem overlays, snapshots-at-file-level, tool-call isolation, export (`fs.overlay`, `fs.tool_isolation`, `fs.snapshot`, `fs.export`).
- Mounted heterogeneous backends (`fs.mount`, `mount.s3`, `mount.gdocs`, …).
- In-process shell substrates and pluggable interpreters (`exec.in_process`, `exec.languages.*`, `exec.network`, `exec.limits`).
- Memory and audit substrates (`memory.kv`, `memory.namespace`, `memory.ttl`, `audit.tool_calls`, `audit.fs_changes`, `audit.query`).
- A host-side Tools layer for translating function-calling agents to ACP-shaped Runtimes.
- `session/steer` as a wire method.
- Approval-guardian capability.

What v0.1 **does** include for the Runtime-side surface:

- ACP's `fs/read_text_file`, `fs/write_text_file`, plus optional `fs/list` and `fs/watch`.
- ACP's `terminal/create`, `terminal/output`, `terminal/wait_for_exit`, `terminal/kill`, `terminal/release`, plus optional `terminal/resize`.
- SAP lifecycle: `runtime/provision`, `runtime/finalize`, `runtime/snapshot`, `runtime/pause`, `runtime/resume`, `runtime/host_for_port`, `runtime/manifest_apply`.

This is the minimum surface needed for a coding agent to do useful work in a chosen Runtime.

## Consequences

**Intended:**

- Phase 2 (90-day demo) is achievable with two Agents (Claude Code, Codex) and two Runtimes (Local, E2B). No AgentFS dependency, no Mirage dependency, no just-bash dependency.
- The demo's portability story is "swap the Runtime via `runtime/snapshot` + reseat" — proven through a microVM snapshot, not through a workspace-substrate snapshot.
- The spec is approximately 40% smaller than the earlier draft, which improves readability and reduces the surface contributors have to internalise.
- The deferred ideas remain visible in `FUTURE.md` so the research is not lost.

**Accepted downsides:**

- Solid Agent loses some of its most differentiated material from the earlier draft. The pitch becomes "ACP plus runtime lifecycle" rather than "ACP plus everything an agent needs."
- The compatibility matrix's rows about workspace state-with-session, audit, overlay all score "future SAP extension" rather than "v0.1 difference."
- Some prior-art projects (AgentFS, Mirage, just-bash) get demoted from "directly motivates capability X" to "informs potential future extensions."

## Alternatives considered

- **Ship workspace substrate capabilities in v0.1.** Rejected on the basis of Phase 2 demo risk and project-lead directive. The integration work for AgentFS / Mirage / just-bash is non-trivial, and v0.1 needs to ship.
- **Drop workspace substrates entirely** without parking them in `FUTURE.md`. Rejected — the research is valuable and the ideas are likely to return; we want to keep them visible for future RFC cycles.
- **Ship a stripped-down "workspace substrate" v0.1** with overlay + memory but not mount + audit. Rejected — partial workspace substrate is worse than none. The boundary should be clean: either substrate is first-class or it isn't.

## Re-introduction criteria

A future SAP version may bring back workspace substrate capabilities when:

1. v0.1 has shipped, has a reference implementation, and at least three independent Hosts are running it in production.
2. Concrete usage data shows agents would benefit from a richer substrate beyond POSIX.
3. An RFC proposes specific capability additions (informed by the sketches in [`FUTURE.md`](../../FUTURE.md)).
4. At least one reference implementation lands the new capability behind a feature flag.
5. At least two independent Agents or Runtimes adopt it.

This is the same graduation process described at the end of `FUTURE.md`.
