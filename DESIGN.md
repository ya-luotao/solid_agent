# Solid Agent — Top-Level Design

> **Status:** Draft 0.1 — companion to [PROTOCOL.md](./PROTOCOL.md). Subject to change.

This document explains the architecture behind the Solid Agent Protocol. It is intended to give implementers, integrators, and reviewers a single place to understand *why* the protocol is shaped the way it is. The protocol itself is intentionally compact; this document is where the trade-offs live.

---

## 1. The two-axis problem

The coding-agent ecosystem has two largely-independent axes:

```
                  Where the agent runs (Runtime)
                  ──────────────────────────────────────────────────
                  │ Local │ E2B │ Daytona │ Modal │ Cloudflare │ … │
        ──────────┼───────┼─────┼─────────┼───────┼────────────┼───┤
   Claude Code   │   ✓   │  ✓  │   ?     │   ?   │     ?      │ … │
   Codex         │   ✓   │  ✓  │   ?     │   ?   │     ?      │ … │
A  AmpCode       │   ✓   │  ?  │   ?     │   ?   │     ?      │ … │
g  Cursor        │   ✓   │  ?  │   ?     │   ?   │     ?      │ … │
e  Pi            │   ✓   │  ?  │   ?     │   ?   │     ?      │ … │
n  Custom        │   ?   │  ?  │   ?     │   ?   │     ?      │ … │
t  …             │   …   │  …  │   …     │   …   │     …      │ … │
                  ──────────────────────────────────────────────────
```

Today the question marks are filled in case-by-case by hand-written integrations. Every host application that wants to support N agents and M runtimes pays an `N × M` integration cost.

**Solid Agent's premise is that the integration cost should be `N + M`.** Each Agent implements one ACP-shaped server. Each Runtime implements one ACP-shaped client. Any pair composes.

This is not a new insight — it's the same insight that motivated ACP itself, and that motivated the `SandboxClient` interface in the OpenAI Agents SDK. Solid Agent's contribution is to **make both axes explicit at the protocol layer** and to fill the gaps that neither spec addresses on its own.

---

## 2. Layered architecture

```
┌──────────────────────────────────────────────────────────────────┐
│ Workflows (out of scope for SAP)                                  │
│   DAG engines, retries, human approval gates, fan-out             │
├──────────────────────────────────────────────────────────────────┤
│ Sessions                                                          │
│   One Agent × one Runtime, bound at creation, ACP-shaped           │
├──────────────────────────────────────────────────────────────────┤
│ Wire layer                                                        │
│   ACP + SAP extensions, over stdio or HTTP/SSE                    │
├──────────────────────────────────────────────────────────────────┤
│ Transport                                                         │
│   newline-delimited JSON-RPC (stdio) or HTTP+SSE                  │
└──────────────────────────────────────────────────────────────────┘

         ↑ horizontal split between agent and runtime ↓

┌────────────────────────────┐        ┌────────────────────────────┐
│ Agent adapter              │        │ Runtime adapter            │
│   ClaudeCode               │        │   Local                    │
│   Codex                    │        │   E2B                      │
│   AmpCode                  │        │   Daytona                  │
│   Cursor                   │        │   Modal                    │
│   Pi                       │        │   Cloudflare Containers    │
│   ACPDirect                │  ───►  │   Vercel Sandboxes         │
│   Custom (any ACP server)  │        │   Runloop                  │
│                            │        │   Blaxel                   │
│   Translates backend       │        │   OpenAI built-in          │
│   protocol into ACP        │        │   Custom (any ACP client)  │
└────────────────────────────┘        └────────────────────────────┘
```

Three observations:

1. **Workflows are above SAP, not inside it.** A host can build a DAG engine, but SAP itself only knows about sessions. This keeps the surface small.
2. **The two adapter columns implement the same protocol from opposite sides.** This is what makes pairing arbitrary.
3. **A "Custom" entry exists in both columns.** Any process that speaks ACP correctly is automatically compatible — no per-provider code on Solid Agent's side. This is the most important leverage point in the design.

---

## 3. Agent interface

A language-agnostic sketch. The Ruby reference implementation will look broadly similar.

```
interface Agent {
  // identity and capabilities
  agent_kind() -> string
  capabilities() -> { thinking, structured_output, image_input, ... }

  // lifecycle
  initialize(host_capabilities) -> negotiated_capabilities
  open_session(options, manifest, runtime) -> session_id
  close_session(session_id)

  // turn handling
  prompt(session_id, content_blocks) -> emits session_update notifications
  cancel(session_id)
  respond_permission(session_id, request_id, decision)
}
```

Implementer guidance:

- An Agent adapter wraps an existing SDK or CLI. The author's job is to translate that backend's stream of events into ACP `session/update` notifications.
- The Agent calls back into the Runtime via `runtime.read_text_file`, `runtime.write_text_file`, `runtime.create_terminal`, etc. — these are the ACP client-side methods. The Host transparently routes them.
- Most production agents have richer event vocabularies than ACP defines. Authors should choose mappings carefully; events with no clean mapping go into `tool_call_update.content` as `content` blocks, and proprietary metadata can ride in the optional `metadata` field on every update. Avoid inventing new top-level update variants without proposing them via RFC.

### 3.1 The `ACPDirect` agent

`ACPDirect` is a special agent kind that simply launches a child process that already speaks ACP and pipes JSON-RPC through. It exists so that any agent which adopts ACP (Claude Code, Codex, AmpCode, Cursor, Pi, and any future entrant) is automatically supported with **zero custom code** in Solid Agent.

This is the design's "long tail" strategy: Solid Agent ships specialised adapters for the agents that need translation today, and trusts ACP adoption to handle the rest.

---

## 4. Runtime interface

```
interface Runtime {
  // identity and capabilities
  runtime_kind() -> string
  capabilities() -> { fs, fs.list, terminal, snapshot, pause, port_forward, gpu, ... }

  // lifecycle
  provision(manifest) -> ready
  pause()                  // capability: pause
  resume(timeout) -> ready  // capability: pause
  snapshot(label) -> snapshot_id  // capability: snapshot
  finalize()

  // ACP client surface (mandatory once `fs` and `terminal` are advertised)
  read_text_file(path, line?, limit?) -> content
  write_text_file(path, content)
  create_terminal(command, args, env?, cwd?) -> terminal_id
  terminal_output(terminal_id) -> chunks (stream)
  terminal_wait_for_exit(terminal_id) -> exit_code
  terminal_kill(terminal_id)
  terminal_release(terminal_id)

  // SAP extensions
  list_entries(path, depth) -> entries     // capability: fs.list
  watch(path, recursive)    -> events      // capability: fs.watch
  host_for_port(port) -> { host, url }      // capability: port_forward
  apply_manifest(manifest)                 // capability: manifest
}
```

Implementer guidance:

- A Runtime adapter wraps the provider's SDK or HTTP API.
- The `provision` step is where the manifest is materialised — repos cloned, files written, env exported, mounts attached, skills synced. Implementations SHOULD be idempotent so that pause/resume cycles can re-validate without re-doing expensive work.
- Reconnection: cloud Runtimes lose their underlying VM occasionally. The adapter is responsible for distinguishing transient network blips from permanent loss; on permanent loss it should emit a `runtime_status: lost` event. Hosts can then decide whether to re-provision.

### 3.1 The `Local` runtime

The reference Local runtime executes child processes on the host machine, with the working directory and environment as configured. It's both the simplest implementation and the most useful for testing — every other Runtime is, in effect, a remote version of Local.

### 3.2 The `Mock` runtime

For testing. Holds an in-memory filesystem and a deterministic terminal that the test can prime with canned output. Bundled with the reference implementation.

---

## 5. Manifest

The Manifest is the only stateful thing that crosses the boundary at session creation. Everything else — prompt, tool calls, responses — is per-turn.

A Manifest declares:

- **Repos** to clone (with ref and optional shallow depth).
- **Files** to seed (inline, fetched from URL, copied from the host, or referenced from a secret store).
- **Env vars** to set (including secret-store references).
- **Mounts** for persistent storage that survives sandbox restarts.
- **Skills / context** — bundled instructional content (akin to `AGENTS.md` or skill packs).
- **Required capabilities** — a forward-declared minimum capability set.

The manifest is **declarative, wire-portable JSON**. It can be loaded from a YAML file, generated dynamically by a host application, or stored alongside a workflow definition. Crucially, it does not depend on any host-language object model, so the same manifest works whether the Host is Ruby, Python, TypeScript, or Go.

The manifest is influenced by, but not identical to, the OpenAI Agents SDK's `Manifest` concept. SAP's version is shaped to round-trip cleanly over JSON, and to be applicable mid-session via `runtime/manifest_apply` for cases where a workflow needs to swap in new context.

---

## 6. Capabilities are the negotiation primitive

There is no "Solid Agent compliance level 1.0". Implementations advertise capabilities; hosts require them; the session creation either succeeds or fails with `capability_unsatisfied`. This pattern:

- Lets Runtimes adopt SAP without implementing the entire surface (a fast-spinning serverless runtime with no pause/resume is still useful).
- Lets new capabilities ship without a protocol version bump.
- Gives Hosts a precise reason when a pairing won't work, instead of a runtime crash deep into a tool call.

The trade-off: capability sprawl. The roadmap proposes a small **capability registry** maintained in this repository, with RFC review before new capabilities are added. The registry starts at the table in [PROTOCOL.md §4](./PROTOCOL.md#4-capability-negotiation) and grows from there.

---

## 7. Composition with workflow engines

SAP itself is per-session. A workflow engine — be it a Ruby DAG library, a Temporal workflow, an Inngest function, an Airflow DAG, or anything else — composes sessions by:

1. Creating a session for each workflow node that runs an agent.
2. Subscribing to the session's update stream and persisting it.
3. Optionally pinning multiple nodes to the **same Runtime** (via a runtime reference handle) so they share filesystem state.
4. Optionally calling `runtime/snapshot` between nodes for rollback points.
5. Closing the session when the node is done.

Multi-agent workflows fall out naturally: one node can run Codex on E2B for implementation, the next can run Claude Code on the same E2B sandbox for review, and a third can run Mock locally for a smoke test. All three see the same workspace because they share the Runtime.

This is why Solid Agent does not bundle a workflow engine: the protocol is the meeting point, and workflow concerns sit cleanly on top.

---

## 8. Reference implementation plan

A Ruby reference implementation is planned in parallel with this specification, but **the spec is the authoritative artifact** in this repository. Concretely:

- `lib/solid_agent/agents/` — built-in agent adapters (Claude Code, Codex, ACPDirect, Mock).
- `lib/solid_agent/runtimes/` — built-in runtime adapters (Local, E2B, Mock initially; more later).
- `lib/solid_agent/wire/` — ACP and SAP message types, serializers, transport implementations.
- `lib/solid_agent/manifest.rb` — manifest type, validation, application orchestration.
- `lib/solid_agent/server.rb` — Rack-based HTTP/SSE server exposing the surface in [PROTOCOL §2.2](./PROTOCOL.md#22-http-transport-remote-used-for-cloud-runtimes).

Independent implementations in TypeScript, Python, Rust, and Go would be very welcome. They should be wire-compatible with the Ruby reference, not API-compatible.

---

## 9. Known limitations and explicit non-goals

- **No streaming model output below the message level.** Solid Agent treats the LLM as a black box owned by the Agent backend; partial-token streaming inside an `agent_message_chunk` is the Agent's affair.
- **No multi-tenant authorization model.** A Host running Solid Agent on behalf of multiple end-users is responsible for its own auth boundary. Sessions are isolated by `session_id` and that's it.
- **No built-in persistence.** Sessions are stateful in memory while running; persisting the event stream is the Host's job. Reference implementations may ship optional adapters (e.g., ActiveRecord, Sequel, S3), but they are not part of the protocol.
- **No "agent skills" registry.** Skill bundles live inside manifests; how a skill is authored or distributed is out of scope.
- **No editor/UI rendering rules.** SAP says nothing about how a Host should display diffs, terminals, or thinking blocks.

---

## 10. Influences and prior art

The shape of this design owes specific debts to:

- **Agent Client Protocol** — wire vocabulary, capability handshake, permission-request model.
- **SandboxAgent (Rivet)** — proof that ACP can be tunnelled over HTTP/SSE and that two-namespace HTTP surfaces (agent + runtime) work in practice.
- **OpenAI Agents SDK `agents.sandbox`** — the `Manifest` concept and the canonical seven-provider Runtime list.
- **Model Context Protocol (MCP)** — content-block shapes, particularly the JSON-RPC stdio framing.
- **LSP** — capability-negotiation pattern over JSON-RPC.

If you have a related effort that should be acknowledged or cross-referenced, please open a PR against this section.
