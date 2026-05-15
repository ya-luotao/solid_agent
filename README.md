# Solid Agent

> **Status:** Early draft — open for discussion. Nothing here is implemented yet.
> Comments, issues, and pull requests are very welcome.

**Solid Agent** is a proposed open specification and reference implementation for a layer that sits between coding agents (Claude Code, Codex, AmpCode, Cursor, Pi, and others) and the runtimes they execute in (your laptop, E2B, Daytona, Modal, Cloudflare Containers, Vercel Sandboxes, Runloop, Blaxel, OpenAI's built-in sandbox, …).

The goal is to make these two dimensions — **which agent** and **where it runs** — independently composable, so application authors can swap either side without rewriting their orchestration code.

```
                 Agent (server side)                Runtime (client side)
                 ─────────────────────              ─────────────────────
                 │ Claude Code      │               │ Local              │
                 │ Codex            │               │ E2B                │
                 │ AmpCode          │  ◄── ACP ──►  │ Daytona            │
                 │ Cursor           │               │ Modal              │
                 │ Pi               │               │ Cloudflare         │
                 │ … any ACP agent  │               │ Vercel             │
                 └──────────────────┘               │ Runloop / Blaxel   │
                                                    │ OpenAI built-in    │
                                                    │ … any ACP runtime  │
                                                    └────────────────────┘
```

## Why this exists

Coding-agent SDKs and sandbox runtimes have proliferated. Each combination today requires bespoke glue. The Agent Client Protocol ([ACP](https://agentclientprotocol.com/)) and emerging conventions like [SandboxAgent](https://sandboxagent.dev/) and the OpenAI Agents SDK's `SandboxClient` interface suggest that the field is converging on a small number of well-defined primitives. **Solid Agent does not invent a new protocol** — it aligns with ACP for the agent↔client wire format, borrows the manifest concept from OpenAI's sandbox layer, and adds a small set of extensions for snapshot/pause/resume semantics that ACP does not yet specify.

The output is intended to be:

1. A **protocol draft** — what's on the wire.
2. A **top-level design** — how the abstraction layers compose.
3. A **roadmap** — what gets built, in what order, with explicit open questions for the community.

A reference Ruby implementation is planned, but the protocol is language-agnostic; we expect TypeScript, Python, Rust, and Go implementations to follow.

## Contents

| File | Purpose |
|---|---|
| [`PROTOCOL.md`](./PROTOCOL.md) | Wire protocol draft — message shapes, ACP alignment, SDK extensions, capability negotiation. |
| [`DESIGN.md`](./DESIGN.md) | Top-level architecture — two-axis abstraction, layered design, agent and runtime interfaces, manifests. |
| [`ROADMAP.md`](./ROADMAP.md) | Phased plan, RFC process, open questions, non-goals. |

## Relationship to existing efforts

Solid Agent is **complementary**, not competitive, with:

- **[Agent Client Protocol](https://agentclientprotocol.com/)** (Zed Industries et al.) — adopted as the wire vocabulary; Solid Agent extends ACP with snapshot/manifest semantics and provides a runtime adapter contract.
- **[SandboxAgent](https://sandboxagent.dev/)** (Rivet) — Solid Agent's server surface is intentionally compatible with SandboxAgent's `/v1/acp/*` namespace, so existing SandboxAgent clients should be able to talk to a Solid Agent server with no changes.
- **OpenAI Agents SDK `agents.sandbox`** — the runtime provider list and the `Manifest` concept come from here. We aim for a one-to-one mapping with the seven providers OpenAI shipped (Blaxel, Cloudflare, Daytona, E2B, Modal, Runloop, Vercel), plus Local and a `Mock` runtime for testing.

## Getting involved

This repository is intentionally **specification first, implementation second**. The first useful PRs are likely:

- Clarifications, typos, and ambiguity reports on `PROTOCOL.md`.
- Counter-proposals for any design decision in `DESIGN.md`.
- Items missing from `ROADMAP.md`, or arguments that the order is wrong.
- Sketches of how a specific agent or runtime would adapt to the proposed interface — these are the most valuable form of feedback.

Please file issues for anything you want to discuss. Decisions will be captured as numbered ADRs in `docs/adr/` once that directory exists.

## License

MIT — see [`LICENSE`](./LICENSE).
