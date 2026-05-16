# Solid Agent — Top-Level Design

> **Status:** Draft 0.1 — companion to [PROTOCOL.md](./PROTOCOL.md). Subject to change.
> **Scope:** v0.1 covers the Agent ↔ Runtime contract. Ideas for richer Workspace substrates are parked in [FUTURE.md](./FUTURE.md).

This document explains the architecture behind the Solid Agent Protocol. The protocol is intentionally compact; this document is where the trade-offs live.

---

## 1. The composition problem

The coding-agent ecosystem has two largely independent concerns:

- **Agents** — Claude Code, Codex, AmpCode, Cursor, Pi, and future entrants. Each ships its own SDK or CLI, its own wire format, its own assumptions about tools.
- **Runtimes** — Local, E2B, Daytona, Modal, Cloudflare Containers, Vercel Sandboxes, Runloop, Blaxel, OpenAI built-in. Each provides a different execution boundary, a different price/performance profile, and a different feature set (snapshot, pause, port forward, GPU).

Today every host application that wants to support N agents on M runtimes pays an `N × M` integration cost. **Solid Agent's premise is that the cost should be `N + M`.** Each Agent implements one ACP-shaped server. Each Runtime implements one ACP-shaped client plus a small lifecycle surface. The pair composes through a capability handshake.

### 1.1 Composable, not orthogonal

We deliberately do not call the Agent and Runtime axes "orthogonal." They are not. An Agent's training and tool schema couple to what its Runtime can do — Claude Code's `Edit` tool assumes a filesystem with a particular write semantics; Codex's `shell` tool assumes a working subprocess; a Runtime that only exposes a virtual filesystem can break either. The right framing is **composable with explicit capability negotiation**: an Agent declares what it needs, a Runtime declares what it offers, and the session is accepted or rejected up front based on the intersection. No silent failures, no hidden coupling.

This is the same pattern the Language Server Protocol uses for editor↔server capabilities and the same pattern the Agent Client Protocol uses for client↔agent capabilities. SAP applies it to the Agent×Runtime pair.

---

## 2. Architecture

```
                Host application
                       │
                       │  session/new + session/prompt
                       ▼
        ┌─────────── Session ───────────┐
        │   ACP messages + SAP runtime  │
        │   methods, capability-negotiated │
        └───────────────────────────────┘
                 │                 │
                 │                 │
        ┌────────▼─────────┐  ┌───▼──────────────┐
        │      Agent       │  │     Runtime       │
        │ (server side)    │  │ (client side +    │
        │                  │  │  lifecycle)       │
        │ ClaudeCode       │  │ Local              │
        │ Codex            │  │ E2B                │
        │ AmpCode          │  │ Daytona            │
        │ Cursor           │  │ Modal              │
        │ Pi               │  │ Cloudflare         │
        │ ACPDirect        │  │ Vercel             │
        │ Custom (any ACP) │  │ Runloop / Blaxel   │
        └──────────────────┘  │ OpenAI built-in    │
                              │ Custom (any ACP)   │
                              └────────────────────┘
```

Three observations:

1. **The Agent and Runtime adapters implement opposite sides of the same protocol.** This is what makes pairing arbitrary.
2. **A "Custom" entry exists in both columns.** Any process that speaks ACP correctly is automatically compatible — no per-vendor code on Solid Agent's side. This is the most important leverage point in the design.
3. **Workflows, persistence, UI rendering, and observability are out of scope.** They compose above SAP. The protocol surface stays small on purpose.

---

## 3. Solid Agent vs MCP — where the boundary is

The most reasonable first question about this spec is: *"My runtime can expose filesystem and shell as MCP tools. Why isn't a coding-agent runtime just a fleet of MCP servers?"*

The answer is that **MCP and SAP answer different questions**, and they compose.

### 3.1 MCP answers "where do the tools come from"

MCP defines a discoverable surface of tools, resources, and prompts that an agent's reasoning loop can call. The protocol's primitives — `tools/list`, `tools/call`, `resources/read`, `prompts/get` — are about **what** an agent can do, and where those capabilities are registered.

MCP servers compose at the **tool catalogue** layer. An agent that has three MCP servers attached can call tools from any of them; the agent's reasoning loop is the merge point.

### 3.2 SAP answers "where does the agent run"

SAP defines the lifecycle and boundary of the Agent's run itself. The primitives — `session/new`, `session/prompt`, `session/cancel`, `runtime/provision`, `runtime/snapshot`, `runtime/pause`, `runtime/resume` — are about **where** an agent runs, **how long** it runs, **how its host stays connected to it**, and **what infrastructure** materialises it.

These concerns are not the same shape as "what tools are available":

- MCP does not specify how an Agent is started, paused, snapshotted, resumed, or moved between runtimes.
- MCP does not specify a Manifest for seeding workspace state before the Agent boots.
- MCP does not specify how to recover an Agent's event stream after a network drop.
- MCP does not specify the Agent's permission flow at the session level — only at the per-tool level via `elicit`.

### 3.3 They compose

A typical Solid Agent session may include several MCP servers as parts of the Agent's tool catalogue. Concretely:

- The Agent declares its MCP server configuration (URLs, transports, capabilities).
- The Runtime hosts those MCP servers in its process tree, on the same network, with manifest-seeded secrets.
- The Manifest's `env:` section provides the credentials those MCP servers need.
- The Host receives `session/update.tool_call` events whose `name` may correspond to MCP-provided tools — SAP doesn't care; it just routes events.

In this layering, MCP defines what the Agent can call; SAP defines the Runtime in which the calling happens.

### 3.4 What SAP doesn't try to be

- **SAP doesn't replace MCP.** Tool registries, resource discovery, and prompt distribution all stay in MCP's lane.
- **SAP doesn't define a tool taxonomy.** Whether an Agent's filesystem access happens through ACP's `fs/read_text_file` (built into the runtime) or through an MCP-provided `read_file` tool is up to the Agent and its configuration.
- **SAP doesn't try to subsume FS, terminal, or anything else into MCP shapes.** ACP made fs and terminal first-class client-side methods specifically because every coding agent uses them, and every agent benefits from skipping tool-discovery roundtrips for those operations. SAP inherits that choice.

### 3.5 Where the boundary feels fuzzy

There is a real overlap: an MCP server could expose `fs.read(path)` as a tool, duplicating what ACP's `fs/read_text_file` already covers. Both are valid implementations. The rough guideline:

- **Universal, every-agent operations** — file reads, file writes, shell — fit ACP's client-side methods. They're built into the Runtime surface and don't need to be discovered.
- **Domain-specific, opt-in capabilities** — GitHub access, database queries, Slack integration, vector search — fit MCP. They're external services that some agents need and others don't, with their own auth and lifecycles.

The reason this split exists, instead of one big tool catalogue, is the cost of discovery. An agent that has to call `tools/list` to find a file-reader before reading a file pays a roundtrip per session for an operation that is universal. ACP's choice to bake fs and terminal into the client side eliminates that overhead. SAP keeps that decision intact.

---

## 4. Agent interface

A language-agnostic sketch.

```
interface Agent {
  agent_kind() -> string
  capabilities() -> { thinking, structured_output, image_input, tools_external, session_load, interrupt, ... }

  initialize(host_capabilities) -> negotiated_capabilities
  open_session(options, manifest, runtime) -> session_id
  close_session(session_id)

  prompt(session_id, content_blocks) -> emits session_update notifications
  cancel(session_id)
  respond_permission(session_id, request_id, decision)
}
```

An Agent is bound to a Runtime at session creation. All filesystem and terminal calls the Agent makes go through ACP's client-side methods (`fs/read_text_file`, `fs/write_text_file`, `terminal/*`), which the Host routes to the bound Runtime.

Agents fall into two implementation styles:

- **ACP-native agents** — they speak ACP `session/update.tool_call` and call back into the Runtime via ACP. Claude Code (via its Ruby SDK) and any agent built directly on ACP fit here.
- **Function-calling agents** — they speak a vendor-specific tool schema (`bash(command)`, `read_file(path)`, …). The Agent adapter translates between that schema and ACP. Codex, Vercel AI SDK agents, OpenAI Agents SDK agents, and most "general function-calling" agents fit here. The translation logic lives in the Agent adapter; SAP does not standardise it.

### 4.1 The `ACPDirect` agent

`ACPDirect` is a special Agent kind that launches a child process that already speaks ACP and pipes JSON-RPC through. Any agent that adopts ACP — Claude Code, Codex, AmpCode, Cursor, Pi, future entrants — is automatically supported with zero custom code in Solid Agent.

This is the design's long-tail strategy: ship specialised adapters for agents that need translation today, and trust ACP adoption to handle the rest.

### 4.2 Reusing existing SDK transports

Several agent SDKs already expose internal transport seams. [`claude-agent-sdk-ruby`](https://github.com/ya-luotao/claude-agent-sdk-ruby), for example, defines an abstract `Transport` class (`connect`, `write`, `read_messages`, `close`, `ready?`, `end_input`) and a `Client.new(transport_class:, transport_args:)` constructor; its [`docs/client.md`](https://github.com/ya-luotao/claude-agent-sdk-ruby/blob/main/docs/client.md) ships a worked example that swaps the default subprocess transport for an `E2BCliTransport` streaming stdio through an E2B microVM. The Codex SDK has a similar low-level seam.

This is good news for Solid Agent: `Agents::ClaudeCode` and `Agents::Codex` do **not** need to re-implement the agent's wire protocol. They reuse the SDK's existing transport hook and supply a Solid-Agent-shaped transport whose `read_messages` / `write` calls are translated into ACP `session/update` events and Runtime method invocations.

The single-SDK transport-replacement pattern is the existence proof that Agent↔Runtime decoupling works in production. Solid Agent's contribution is to lift that pattern from per-pair custom code to "implement an Agent adapter once, implement a Runtime adapter once, and they compose."

---

## 5. Runtime interface

```
interface Runtime {
  runtime_kind() -> string
  capabilities() -> { fs, fs.list, terminal, snapshot, pause, port_forward, gpu, region, ... }

  provision(manifest) -> ready
  finalize()
  pause()                          // capability: pause
  resume(timeout) -> ready          // capability: pause
  snapshot(label) -> snapshot_id    // capability: snapshot
  host_for_port(port) -> { host, url }  // capability: port_forward
  manifest_apply(manifest)              // mid-session reconfiguration

  // ACP client surface (required when `fs` and `terminal` are advertised)
  read_text_file(path, line?, limit?) -> content
  write_text_file(path, content)
  create_terminal(command, args, env?, cwd?) -> terminal_id
  terminal_output(terminal_id) -> chunks (stream)
  terminal_wait_for_exit(terminal_id) -> exit_code
  terminal_kill(terminal_id)
  terminal_release(terminal_id)

  // optional
  list_entries(path, depth) -> entries  // capability: fs.list
  watch(path, recursive)    -> events    // capability: fs.watch
}
```

Implementer guidance:

- A Runtime wraps the provider's SDK or HTTP API. The most important method is `provision(manifest)` — the rest of the surface depends on a working workspace existing.
- The `provision` step materialises the manifest: clone repos, seed files, export env, attach mounts as configured. Implementations SHOULD be idempotent so that pause/resume cycles can re-validate without re-doing expensive work.
- Reconnection: cloud Runtimes occasionally lose their underlying compute. The adapter distinguishes transient blips from permanent loss; on permanent loss it emits `runtime_status: lost` and the Host decides whether to re-provision.

### 5.1 The `Local` runtime

Reference Local runtime executes child processes on the host machine. Simplest implementation, most useful for testing, every other Runtime is functionally a remote version of it.

### 5.2 The `Mock` runtime

For testing. In-memory filesystem, deterministic terminal that tests can prime with canned output. Bundled with the reference implementation.

---

## 6. Manifest

The Manifest is the only stateful payload that crosses the boundary at session creation. Everything else — prompts, tool calls, responses — is per-turn.

```
- repos:                  git repositories to clone
- files:                  file contents to seed (inline / URL / local / secret)
- env:                    environment variables (literal or secret-store references)
- capabilities_required:  forward-declared minimum capability set
```

The manifest is declarative, wire-portable JSON. Loading from YAML is a Host convenience. The same manifest works whether the Host is Ruby, Python, TypeScript, or Go.

Richer manifest features (mounted heterogeneous backends, memory seeds, skill bundles) are deferred to a future SAP version; see [FUTURE.md](./FUTURE.md).

---

## 7. Capabilities are the negotiation primitive

There is no "SAP 1.0 compliance level." Implementations advertise capabilities; hosts require them; session creation either succeeds or fails with `capability_unsatisfied`. This pattern:

- Lets Runtimes adopt SAP without implementing the entire surface — a fast-spinning serverless runtime with no pause/resume is still useful.
- Lets Hosts get a precise reason when a pairing won't work, instead of a runtime crash deep into a tool call.
- Lets new capabilities ship without a protocol version bump.

The trade-off: capability sprawl. The roadmap proposes a small **capability registry** maintained in this repository, with RFC review before new capabilities are added.

---

## 8. Composition with workflow engines

SAP is per-session. A workflow engine — Ruby DAG library, Temporal workflow, Inngest function, Airflow DAG — composes sessions by:

1. Creating a session for each workflow node.
2. Subscribing to the session's `session/update` stream and persisting it.
3. Pinning multiple nodes to the **same Runtime handle** (via reference) so they share workspace state.
4. Calling `runtime/snapshot` between nodes for rollback points.
5. Closing the session when the node is done.

A multi-agent pipeline falls out naturally: Codex implements a feature on an E2B sandbox; Claude Code reviews the same sandbox; a final mock-agent smoke-tests it. All three see the same files because they share the Runtime. SAP does not bundle a workflow engine; the protocol is the meeting point.

---

## 9. Reference implementation plan

A Ruby reference implementation is planned in parallel with this specification, but **the spec is the authoritative artifact** in this repository. Concretely:

- `lib/solid_agent/agents/` — built-in Agent adapters (Claude Code, Codex, ACPDirect, Mock).
- `lib/solid_agent/runtimes/` — built-in Runtime adapters (Local, E2B, Mock initially).
- `lib/solid_agent/wire/` — ACP and SAP message types, serializers, transport implementations.
- `lib/solid_agent/manifest.rb` — Manifest type, validation, application orchestration.
- `lib/solid_agent/server.rb` — Rack-based HTTP/SSE server exposing the surface in [PROTOCOL §2.2](./PROTOCOL.md#22-http-transport).

Independent implementations in TypeScript, Python, Rust, and Go are welcome. They should be wire-compatible with the Ruby reference, not API-compatible.

---

## 10. What's in scope vs deferred

**In scope for v0.1.**

- Agent adapter contract (wrapping existing agent SDKs).
- Runtime adapter contract (provisioning workspaces, hosting agents, lifecycle methods).
- ACP-aligned wire format (sessions, tool calls, permissions, fs/terminal client-side methods).
- Manifest with repos, files, env, capability requirements.
- HTTP/SSE transport with reconnection.
- Capability registry.

**Deferred to future SAP versions** (parked in [FUTURE.md](./FUTURE.md)):

- Filesystem overlays, snapshots-at-file-level, tool-call isolation, export.
- Mounted heterogeneous backends (S3, Drive, Slack, GitHub, …).
- In-process shell substrates with execution limits and pluggable interpreters.
- Memory and audit stores as Runtime-served substrates.
- A host-side Tools layer for bridging function-calling agents to ACP-shaped Runtimes.
- `session/steer` for mid-turn input injection.
- Approval-guardian capability.

The deferral is intentional. v0.1 wants to ship a defensible Agent ↔ Runtime contract first; richer Workspace substrates can return as RFCs once a reference implementation exists and real usage data informs the design.

---

## 11. Known limitations and explicit non-goals

- **No standardisation of Reasoning.** SAP does not specify how an Agent reaches an LLM or how it plans.
- **No multi-tenant auth model.** A Host running Solid Agent on behalf of multiple end-users handles its own auth boundary.
- **No built-in persistence.** Sessions are stateful in memory while running; persisting the event stream is the Host's job.
- **No editor/UI rendering rules.** SAP says nothing about how a Host should display diffs, terminals, or thinking blocks.
- **No "skills registry."** Skill bundles are deferred; how skills are authored or distributed is out of scope.

---

## 12. Influences and prior art

- **[Agent Client Protocol](https://agentclientprotocol.com/)** — wire vocabulary, capability handshake, permission-request model.
- **[SandboxAgent](https://sandboxagent.dev/) (Rivet)** — ACP-over-HTTP/SSE, two-namespace HTTP surfaces, proof that the pattern works in practice.
- **OpenAI Agents SDK `agents.sandbox`** — the `Manifest` concept and the canonical Runtime provider list.
- **[Model Context Protocol](https://modelcontextprotocol.io/) (MCP)** — content-block shapes, JSON-RPC stdio framing, the tool-registry layer that complements SAP (see §3).
- **[Language Server Protocol](https://microsoft.github.io/language-server-protocol/)** — capability-negotiation pattern over JSON-RPC.

Projects whose ideas inform potential **future** SAP extensions (see [FUTURE.md](./FUTURE.md)) include [AgentFS](https://github.com/tursodatabase/agentfs), [Mirage](https://github.com/strukto-ai/mirage), and [just-bash](https://github.com/vercel-labs/just-bash). v0.1 does not adopt their substrate-level concepts.

If you have a related effort that should be acknowledged or cross-referenced, please open a PR.
