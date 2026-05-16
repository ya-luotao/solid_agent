# Solid Agent

> **Status:** Early draft — open for discussion. Nothing here is implemented yet.
> **Scope:** v0.1 focuses on **wrapping coding-agent SDKs and decoupling them from runtimes**. Richer workspace concerns (filesystem overlays, mounted backends, in-process shells, memory and audit substrates) are parked in [`FUTURE.md`](./FUTURE.md) and may return as RFCs after a reference implementation exists.
> Comments, issues, and pull requests are very welcome.

**Solid Agent** is a proposed open specification and reference implementation for the layer between coding agents (Claude Code, Codex, AmpCode, Cursor, Pi, and others) and the runtimes they execute in (your laptop, E2B, Daytona, Modal, Cloudflare Containers, Vercel Sandboxes, Runloop, Blaxel, OpenAI's built-in sandbox, …).

The goal is to make these two dimensions — **which agent** and **where it runs** — composable through explicit capability negotiation, so application authors can swap either side without rewriting their orchestration code.

```
              ┌────── Agent ──────┐         ┌────── Runtime ──────┐
              │ Claude Code       │         │ Local                │
              │ Codex             │         │ E2B                  │
              │ AmpCode           │ ─ ACP ─ │ Daytona              │
              │ Cursor            │         │ Modal                │
              │ Pi                │         │ Cloudflare           │
              │ ACPDirect         │         │ Vercel               │
              │ Mock              │         │ Runloop / Blaxel     │
              │ … any ACP agent   │         │ OpenAI built-in      │
              └───────────────────┘         │ Mock                 │
                                             │ … any ACP runtime    │
                                             └──────────────────────┘
```

## Why this exists

Coding-agent SDKs and sandbox runtimes have proliferated. Each `(agent, runtime)` combination today requires bespoke glue. The [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) and emerging conventions like [SandboxAgent](https://sandboxagent.dev/) and the OpenAI Agents SDK's `SandboxClient` interface suggest the field is converging on a small number of well-defined primitives.

**Solid Agent does not invent a new protocol.** It aligns with ACP for the on-the-wire format, borrows the Manifest concept from OpenAI's sandbox layer, and specifies a small Runtime lifecycle surface (snapshot, pause, resume, port forward, manifest apply) plus an HTTP/SSE transport that ACP does not yet cover.

The Agent ↔ Runtime decoupling pattern is also already partially in production in single-vendor form. The Ruby SDK [`claude-agent-sdk-ruby`](https://github.com/ya-luotao/claude-agent-sdk-ruby) ships a `Transport` abstract class with pluggable implementations — its [docs](https://github.com/ya-luotao/claude-agent-sdk-ruby/blob/main/docs/client.md) include a worked example routing the Claude Code CLI's stdio through E2B by replacing only the transport class. Solid Agent generalises that decoupling across multiple Agent backends.

The output is intended to be:

1. A **protocol draft** — what's on the wire.
2. A **top-level design** — how the two roles compose.
3. A **roadmap** — what gets built, in what order, with explicit open questions.
4. A **compatibility matrix** — what each agent supports natively today.

A reference Ruby implementation is planned, but the protocol is language-agnostic; we expect TypeScript, Python, Rust, and Go implementations to follow.

## Contents

| File | Purpose |
|---|---|
| [`PROTOCOL.md`](./PROTOCOL.md) | Wire protocol draft — message shapes, ACP alignment, Runtime methods, capability registry, manifest, errors. |
| [`DESIGN.md`](./DESIGN.md) | Top-level architecture — two-axis abstraction, composable not orthogonal, Solid Agent vs MCP boundary, agent / runtime interfaces, manifests. |
| [`COMPATIBILITY.md`](./COMPATIBILITY.md) | Developer reference — ACP-rooted compatibility matrix across Claude Code, Codex, Pi, AmpCode, Cursor. |
| [`ROADMAP.md`](./ROADMAP.md) | Phased plan, 90-day demo target, RFC process, open questions, non-goals. |
| [`FUTURE.md`](./FUTURE.md) | Parked design notes for workspace substrate ideas explicitly **out of scope for v0.1**. |

## Relationship to existing efforts

Solid Agent is **complementary**, not competitive, with:

- **[Agent Client Protocol](https://agentclientprotocol.com/)** (Zed Industries et al.) — adopted as the wire vocabulary; Solid Agent extends ACP with Runtime lifecycle (snapshot/pause/resume) and a manifest concept.
- **[SandboxAgent](https://sandboxagent.dev/)** (Rivet) — Solid Agent's HTTP surface is intentionally compatible with SandboxAgent's `/v1/acp/*` namespacing, so existing SandboxAgent clients should be able to talk to a Solid Agent server with no changes.
- **OpenAI Agents SDK `agents.sandbox`** — the runtime provider list and the `Manifest` concept come from here. We aim for a one-to-one mapping with the seven providers OpenAI shipped (Blaxel, Cloudflare, Daytona, E2B, Modal, Runloop, Vercel), plus Local and a `Mock` runtime for testing.
- **[Model Context Protocol](https://modelcontextprotocol.io/)** — MCP defines what tools an agent can call; SAP defines where the agent runs. They compose at different layers; SAP does not try to replace MCP. See [`DESIGN.md §3`](./DESIGN.md#3-solid-agent-vs-mcp--where-the-boundary-is).

Projects influential for **future** SAP extensions (out of scope for v0.1):

- **[AgentFS](https://github.com/tursodatabase/agentfs)** (Turso) — copy-on-write overlay filesystem with audit log and portable export.
- **[Mirage](https://github.com/strukto-ai/mirage)** (Strukto) — virtual filesystem mounting heterogeneous backends as paths.
- **[just-bash](https://github.com/vercel-labs/just-bash)** (Vercel Labs) — in-process bash with pluggable virtual filesystems.

These projects argue that the agent's operating substrate is differentiable in ways v0.1 deliberately ignores. They are interesting and will likely return as RFCs; see [`FUTURE.md`](./FUTURE.md) for sketches.

## Getting involved

This repository is intentionally **specification first, implementation second**. The first useful PRs are likely:

- Clarifications, typos, and ambiguity reports on `PROTOCOL.md`.
- Counter-proposals for any design decision in `DESIGN.md`.
- Items missing from `ROADMAP.md`, or arguments that the order is wrong.
- Sketches of how a specific agent or runtime would adapt to the proposed interface — these are the most valuable form of feedback.

Architectural decisions are captured as numbered ADRs in `docs/adr/` once that directory exists.

## License

MIT — see [`LICENSE`](./LICENSE).
