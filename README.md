# Solid Agent

> **Status:** Early draft — open for discussion. Nothing here is implemented yet.
> Comments, issues, and pull requests are very welcome.

**Solid Agent** is a proposed open specification and reference implementation that names and standardises the three orthogonal concerns inside a working coding agent: **which agent** reasons, **what workspace** it operates on, and **which runtime** hosts that workspace.

The goal is to make these three dimensions independently composable, so application authors can swap any one without rewriting the others.

```
              ┌────── Agent ──────┐    ┌────── Workspace ──────┐    ┌────── Runtime ──────┐
              │ Claude Code       │    │ POSIX (default)       │    │ Local               │
              │ Codex             │    │ AgentFS-backed        │    │ E2B                 │
              │ AmpCode           │    │   • overlay / CoW     │    │ Daytona             │
              │ Cursor            │    │   • memory (KV)       │    │ Modal               │
              │ Pi                │    │   • audit log         │    │ Cloudflare          │
              │ ACPDirect         │    │   • portable export   │    │ Vercel              │
              │ Mock              │    │ Mirage-backed         │    │ Runloop / Blaxel    │
              │ … any ACP agent   │    │   • mount.s3 / gdocs  │    │ OpenAI built-in     │
              └─────────┬─────────┘    │   • mount.slack / …   │    │ Mock                │
                        │              │ just-bash-backed      │    │ … any ACP runtime   │
                        │              │   • in-process shell  │    └──────────┬──────────┘
                        │              │ Mock                  │               │
                        │              │ … any ACP workspace   │               │
                        │              └───────────┬───────────┘               │
                        │                          │                            │
                        └─────────── ACP + SAP extensions ─────────────────────┘
```

## Why this exists

It is tempting to think of a coding agent as "an LLM plus some tools." That framing understates what's going on. In practice a working coding agent is the product of three orthogonal concerns:

- **Reasoning** — the LLM, prompt, planner. The underlying Agent SDK's job.
- **Workspace** — the persistent operating substrate the agent reads, writes, executes against, and learns from. Includes the filesystem, the shell, optional memory, and optional audit history. **The workspace's shape is constitutive of agent behaviour**, not a Runtime detail.
- **Runtime** — the execution boundary that hosts the Workspace. Local laptop, microVM, cloud container.

Today every Host application that wants to support N agents × M workspaces × K runtimes pays a multiplicative integration cost. The Agent Client Protocol ([ACP](https://agentclientprotocol.com/)), emerging conventions like [SandboxAgent](https://sandboxagent.dev/), and the OpenAI Agents SDK's `SandboxClient` interface suggest the field is converging on a small number of well-defined primitives. Projects like [AgentFS](https://github.com/tursodatabase/agentfs), [Mirage](https://github.com/strukto-ai/mirage), and [just-bash](https://github.com/vercel-labs/just-bash) show that the Workspace itself deserves first-class treatment — not as a tool but as a substrate.

**Solid Agent does not invent a new protocol.** It aligns with ACP for the on-the-wire format, borrows the Manifest concept from OpenAI's sandbox layer, adds Workspace substrates for overlay/memory/audit/mount inspired by the projects above, and specifies a small set of extensions for snapshot, pause, and resume that ACP does not yet cover.

The pattern is also already partially in production in single-vendor form. The Ruby SDK [`claude-agent-sdk-ruby`](https://github.com/ya-luotao/claude-agent-sdk-ruby) ships a `Transport` abstract class with pluggable implementations — its [docs](https://github.com/ya-luotao/claude-agent-sdk-ruby/blob/main/docs/client.md) include a worked example that routes the Claude Code CLI's stdio through E2B by replacing only the transport class. That works because the agent SDK already decouples *the wire* from *the runtime*. Solid Agent generalises the same decoupling along three more axes: across multiple Agent backends, across pluggable Workspaces, and with the Workspace surface itself made externally addressable.

The output is intended to be:

1. A **protocol draft** — what's on the wire.
2. A **top-level design** — how the abstraction layers compose.
3. A **roadmap** — what gets built, in what order, with explicit open questions for the community.

A reference Ruby implementation is planned, but the protocol is language-agnostic; we expect TypeScript, Python, Rust, and Go implementations to follow.

## Contents

| File | Purpose |
|---|---|
| [`PROTOCOL.md`](./PROTOCOL.md) | Wire protocol draft — message shapes, ACP alignment, Workspace substrate methods, capability registry, manifest, errors. |
| [`DESIGN.md`](./DESIGN.md) | Top-level architecture — three-axis abstraction (Agent × Workspace × Runtime), Workspace as a first-class pillar, agent / workspace / runtime interfaces, manifests. |
| [`ROADMAP.md`](./ROADMAP.md) | Phased plan, RFC process, open questions, non-goals. |

## Relationship to existing efforts

Solid Agent is **complementary**, not competitive, with:

**Wire & host conventions**

- **[Agent Client Protocol](https://agentclientprotocol.com/)** (Zed Industries et al.) — adopted as the wire vocabulary; Solid Agent extends ACP with snapshot/manifest semantics and provides a runtime adapter contract.
- **[SandboxAgent](https://sandboxagent.dev/)** (Rivet) — Solid Agent's server surface is intentionally compatible with SandboxAgent's `/v1/acp/*` namespace, so existing SandboxAgent clients should be able to talk to a Solid Agent server with no changes.
- **OpenAI Agents SDK `agents.sandbox`** — the runtime provider list and the `Manifest` concept come from here. We aim for a one-to-one mapping with the seven providers OpenAI shipped (Blaxel, Cloudflare, Daytona, E2B, Modal, Runloop, Vercel), plus Local and a `Mock` runtime for testing.

**Workspace substrates** (these projects argue that "the agent's operating substrate" is a first-class architectural concern, not Runtime trivia)

- **[AgentFS](https://github.com/tursodatabase/agentfs)** (Turso) — embedded SQLite-backed agent filesystem combining FS + KV + tool-call audit log + copy-on-write overlays in a single portable file. Argues that an agent's filesystem differs in kind from a human's: overlays, audit, and memory are not extras, they define usability. Solid Agent's `fs.overlay`, `fs.tool_isolation`, `memory.kv`, `audit.tool_calls`, and `fs.export` capabilities all trace back to this design.
- **[Mirage](https://github.com/strukto-ai/mirage)** (Strukto) — unified virtual filesystem mounting heterogeneous backends (S3, Drive, Slack, GitHub, Redis, MongoDB, …) into one tree with snapshots and `.tar` export. Argues that external data sources belong in the Workspace as paths rather than as N separate tools. Directly informs Solid Agent's `fs.mount` capability family and the manifest's `mounts:` shape.
- **[just-bash](https://github.com/vercel-labs/just-bash)** (Vercel Labs) — in-process bash interpreter with a pluggable virtual filesystem and optional in-process Python / JavaScript / SQLite. Argues that the shell is the agent's primary verb, and its environment (FS view, network, exec limits) is constitutive of agent behaviour. Directly informs Solid Agent's `exec.in_process`, `exec.languages.*`, `exec.network`, and `exec.limits` capabilities.

## Getting involved

This repository is intentionally **specification first, implementation second**. The first useful PRs are likely:

- Clarifications, typos, and ambiguity reports on `PROTOCOL.md`.
- Counter-proposals for any design decision in `DESIGN.md`.
- Items missing from `ROADMAP.md`, or arguments that the order is wrong.
- Sketches of how a specific agent or runtime would adapt to the proposed interface — these are the most valuable form of feedback.

Please file issues for anything you want to discuss. Decisions will be captured as numbered ADRs in `docs/adr/` once that directory exists.

## License

MIT — see [`LICENSE`](./LICENSE).
