# Solid Agent — Top-Level Design

> **Status:** Draft 0.1 — companion to [PROTOCOL.md](./PROTOCOL.md). Subject to change.

This document explains the architecture behind the Solid Agent Protocol. It is intended to give implementers, integrators, and reviewers a single place to understand *why* the protocol is shaped the way it is. The protocol itself is intentionally compact; this document is where the trade-offs live.

---

## 1. What an agent really is

A coding agent looks at first glance like "an LLM plus some tools." That framing is convenient for SDK demos but it understates what is going on. In practice, a working coding agent is the composition of three orthogonal concerns:

1. **Reasoning** — the LLM, the prompt, the planner, the model's chain-of-thought, the temperature settings. This is what the underlying Agent SDK (Claude Agent SDK, Codex SDK, AmpCode, Cursor, Pi, …) is *primarily* about.
2. **Workspace** — the persistent operating substrate the agent reads, writes, executes against, and learns from. This is far richer than "the current directory." It contains the filesystem the agent sees, the shell or exec primitive that is its main verb, the memory it accumulates across turns, and the audit log that makes its behaviour reproducible and debuggable.
3. **Runtime** — the execution boundary that hosts the Workspace and the agent's processes. Local laptop, an E2B microVM, a Daytona workspace, a Modal container, a Cloudflare Container, a Vercel sandbox, Runloop, Blaxel, the OpenAI built-in sandbox — each is a different shape of Runtime, distinguished by isolation guarantees, performance characteristics, and operational features (snapshot, pause, port forwarding).

```
                ┌────────────── AGENT ──────────────┐
                │                                    │
                │   ┌──────────┐   actions    ┌────┴────────┐
                │   │ Reasoning ├─────────────▶│ Workspace  │
                │   │  (LLM)   │◀──observations│             │
                │   └──────────┘               │ • FS        │
                │                              │ • Shell/exec│
                │                              │ • Memory    │
                │                              │ • Audit     │
                │                              └────┬────────┘
                │                                   │
                │                  ┌────────────────┴──────────┐
                │                  │        RUNTIME              │
                │                  │  (Local / E2B / Daytona /   │
                │                  │   Modal / Cloudflare / …)   │
                │                  └─────────────────────────────┘
                └─────────────────────────────────────────────────┘
```

The current ecosystem largely conflates these three concerns inside a single product. Claude Code, Codex, and Cursor each ship their own implicit Workspace shape and their own Runtime defaults. Sandbox providers (E2B, Daytona, Modal, …) ship a Runtime but leave Workspace shape entirely up to the caller. AgentFS, Mirage, and just-bash each address one Workspace substrate but are not yet wired into the surrounding spec.

**Solid Agent's contribution is to name the three concerns and define a wire contract between them.** Solid Agent does not try to standardise reasoning (the model can be anything). It does try to standardise the contract between an agent's reasoning loop and the Workspace it operates on, and between a Workspace and the Runtime that hosts it.

---

## 2. Why "Workspace" is first-class

It is tempting to call the Workspace concerns — filesystem, shell, memory, audit — "Runtime details" or "tool details" and move on. We deliberately do not. The reasons:

- **Filesystems for agents are different in kind from filesystems for humans.** [AgentFS](https://github.com/tursodatabase/agentfs) makes this explicit: an agent's filesystem benefits from copy-on-write overlays (so each tool call can be tried, rolled back, or committed), from an insert-only audit log (so the agent's history is queryable), from a co-located key-value store (so structured memory lives alongside files), and from the property of fitting in a single portable file (so a workspace can move between hosts). None of these are "extras." They define what makes a workspace usable for an agent rather than for a human.

- **Heterogeneous data sources should be reachable through the workspace, not bolted on as N separate tools.** [Mirage](https://github.com/strukto-ai/mirage) demonstrates this: mounting S3, Slack, Google Drive, GitHub, Redis, and SSH as paths under one tree turns "wire up N SDKs and M MCP servers" into "the workspace has more paths." The Workspace is the natural home of that abstraction; it cannot live inside a single tool.

- **The shell is the agent's primary verb.** [just-bash](https://github.com/vercel-labs/just-bash) makes this concrete by showing how much agent behaviour can be controlled through the shell's environment: which filesystem it sees, whether networking is on, what execution limits apply, whether `python` and `sqlite` are in-process or out. Treating shell as "one tool among many" misses that almost every coding agent's training assumes a `bash` (or `shell`) primitive as the main action surface.

- **Workspaces outlive sessions and outlive Runtimes.** A workspace can be paused, snapshotted, exported, and re-materialised on a different Runtime later. A workspace contract that is opaque inside the Runtime cannot do this; one that is first-class in the protocol can.

So Solid Agent treats Workspace as an architectural pillar alongside Agent and Runtime, and explicitly enumerates the substrates a Workspace can offer.

---

## 3. The Workspace contract

A Workspace exposes up to four substrates. Implementations advertise which substrates they support; capability negotiation handles the rest (see §7).

### 3.1 Filesystem substrate

The minimum useful Workspace has files. The reference operations are ACP's:

- `fs/read_text_file(path, line?, limit?)`
- `fs/write_text_file(path, content)`

SAP adds optional capabilities for richer filesystem behaviour:

| Capability | Adds |
|---|---|
| `fs.list` | Directory listing. |
| `fs.watch` | Subscribe to change events. |
| `fs.overlay` | Begin / commit / discard a copy-on-write layer. Inspired by AgentFS. |
| `fs.tool_isolation` | Each tool call gets its own implicit overlay; merge on success, discard on failure. |
| `fs.snapshot` | Named filesystem checkpoints with restore. |
| `fs.mount` | Pluggable backends mounted as paths. Inspired by Mirage. Each backend kind is a sub-capability (`mount.s3`, `mount.gdocs`, `mount.slack`, …). |
| `fs.export` | Dump the entire workspace as a portable artefact (`.tar`, `.db`, or implementation-defined). |

A "Local" Workspace advertises only `fs`, `fs.list`, and `fs.watch`. An "AgentFS-backed" Workspace also advertises `fs.overlay`, `fs.tool_isolation`, `fs.snapshot`, `fs.export`. A "Mirage-backed" Workspace adds `fs.mount` plus a list of backend kinds. These compose: a Workspace can be both AgentFS-backed (for overlay and audit) and Mirage-backed (for mounted external data).

### 3.2 Shell substrate

Most coding agents reason about action by writing shell commands. The reference operations are ACP's terminal methods:

- `terminal/create(command, args, env?, cwd?)` → `terminal_id`
- `terminal/output(terminal_id)` → streamed chunks
- `terminal/wait_for_exit(terminal_id)` → exit code
- `terminal/kill(terminal_id)`
- `terminal/release(terminal_id)`

These operations are deliberately stateful: a `terminal_id` represents a live PTY-like handle. Many agents in practice expect a simpler "run and return" shape (`bash(command) -> { stdout, stderr, exit_code }`); the **Tools layer** (see §6) provides that shape on top of the streaming primitives.

SAP adds capabilities for richer shell behaviour:

| Capability | Adds |
|---|---|
| `terminal.pty` | Terminals are full PTYs (resize, raw mode). |
| `exec.in_process` | The shell runs in the host's address space rather than as a subprocess. Inspired by just-bash. Useful for tests, CI smoke runs, and environments without a usable subprocess primitive. |
| `exec.languages` | The shell can execute additional in-process interpreters (`python`, `javascript`, `sqlite`). Inspired by just-bash's pluggable interpreters. |
| `exec.network` | Outbound network is allowed (default off in some Runtimes). |
| `exec.limits` | The Runtime enforces structured execution limits (max commands, max loop iterations, wall-clock timeout, call depth). |

### 3.3 Memory substrate

A Workspace may include a key-value store for agent-accumulated state. Inspired by AgentFS's `kv_*` operations.

| Capability | Adds |
|---|---|
| `memory.kv` | `memory_get(key)`, `memory_set(key, value, ttl?)`, `memory_delete(key)`, `memory_list(prefix?)`. |
| `memory.namespace` | Logical namespaces (e.g., per-skill, per-tool, per-conversation). |
| `memory.export` | Dump memory contents as a portable structure. |

The default Local Workspace does not advertise memory — agents fall back to scratch files. Workspaces that take memory seriously advertise `memory.kv` and design their semantics (consistency, durability, TTL) explicitly.

### 3.4 Audit substrate

A Workspace can be queryable about its own history. Inspired by AgentFS's insert-only `tool_calls` log.

| Capability | Adds |
|---|---|
| `audit.tool_calls` | Append-only record of `{tool_call_id, name, args, started_at, finished_at, status, result_size}`. |
| `audit.fs_changes` | Append-only record of files touched per tool call. |
| `audit.query` | Structured query API over the audit store (`since`, `tool_name`, `status`, …). |

The audit substrate matters because:

- It lives in the Workspace, not the Host. A Host can lose its event stream and still reconstruct what happened by querying the Workspace's audit log.
- It travels with the Workspace if the Workspace is exported. A `.db` or `.tar` dump is a complete reproducible record.
- It enables Workspace-level introspection ("what tool calls touched this file?") that no single tool can answer.

---

## 4. Agent interface

A language-agnostic sketch. The Ruby reference implementation will look broadly similar.

```
interface Agent {
  // identity and capabilities
  agent_kind() -> string
  capabilities() -> { thinking, structured_output, image_input, tools_external, ... }

  // lifecycle
  initialize(host_capabilities) -> negotiated_capabilities
  open_session(options, manifest, workspace, tool_set) -> session_id
  close_session(session_id)

  // turn handling
  prompt(session_id, content_blocks) -> emits session_update notifications
  cancel(session_id)
  respond_permission(session_id, request_id, decision)
}
```

An Agent is bound to a Workspace at session creation. The Agent never reaches around the Workspace contract; all reads, writes, executions, memory accesses, and audit queries go through ACP / SAP methods that the Workspace fulfils.

Agents fall into two implementation styles:

- **ACP-native agents** — they speak ACP `session/update.tool_call` and call back into the Workspace with `fs/read_text_file`, `terminal/create`, etc. Claude Code (via its Ruby SDK) and any agent built directly on ACP fall here.
- **Function-calling agents** — they speak a vendor-specific tool schema (`bash(command)`, `read_file(path)`, …). The Agent adapter renders the bound `ToolSet` into that schema, intercepts tool calls, and translates them into Workspace operations. Codex, Vercel AI SDK agents, OpenAI Agents SDK agents, and most "general function-calling" agents fall here.

### 4.1 The `ACPDirect` agent

`ACPDirect` is a special Agent kind that launches a child process that already speaks ACP and pipes JSON-RPC through. Any agent that adopts ACP — Claude Code, Codex, AmpCode, Cursor, Pi, future entrants — is automatically supported with zero custom code on Solid Agent's side.

This is the design's "long tail" strategy: Solid Agent ships specialised adapters for agents that need translation today, and trusts ACP adoption to handle the rest.

### 4.2 Reusing existing SDK transports

Several agent SDKs already expose internal transport seams. [`claude-agent-sdk-ruby`](https://github.com/ya-luotao/claude-agent-sdk-ruby), for example, defines an abstract `Transport` class (`connect`, `write`, `read_messages`, `close`, `ready?`, `end_input`) and a `Client.new(transport_class:, transport_args:)` constructor; its [`docs/client.md`](https://github.com/ya-luotao/claude-agent-sdk-ruby/blob/main/docs/client.md) ships a worked example that swaps the default subprocess transport for an `E2BCliTransport` that streams stdio through an E2B microVM. The Codex SDK uses a similar JSON-RPC stdio seam.

This is good news for Solid Agent: the `Agents::ClaudeCode` and `Agents::Codex` adapters do **not** need to re-implement an Agent's wire protocol. They reuse the SDK's existing transport hook and supply a Solid-Agent-shaped transport whose `read_messages` / `write` calls are translated into ACP `session/update` events and Workspace method invocations.

Concretely, the reference implementation's `Agents::ClaudeCode` will hold a `claude-agent-sdk-ruby` `Client`, configured with a `transport_class:` whose implementation routes stdio through the bound Workspace's shell substrate (via `terminal/create` + `terminal/output`). Where today a user manually wires `transport_class: E2BCliTransport, transport_args: { sandbox: sandbox }` for one specific runtime, Solid Agent supplies the same wiring uniformly across every Runtime the Workspace is provisioned on.

The single-SDK Transport pattern is the existence proof that Agent↔Runtime decoupling works in production. Solid Agent's contribution is to lift that pattern from "custom code per (agent, runtime) pair" to "implement an Agent adapter once, implement a Runtime adapter once, and they compose."

---

## 5. Runtime interface

```
interface Runtime {
  // identity and capabilities
  runtime_kind() -> string
  capabilities() -> { workspace_substrates: [...], snapshot, pause, port_forward, gpu, ... }

  // lifecycle
  provision(manifest) -> Workspace handle
  pause()                          // capability: pause
  resume(timeout) -> ready          // capability: pause
  snapshot(label) -> snapshot_id    // capability: snapshot
  finalize()
  host_for_port(port) -> { host, url }  // capability: port_forward
}
```

A Runtime is a **Workspace provisioner**. Its job is to materialise a Workspace that the Agent can operate on, hold it for the duration of the session, and dispose of it cleanly. Different Runtimes ship different Workspace shapes:

- A `Local` Runtime provisions a Workspace whose FS is your laptop's filesystem and whose Shell is a subprocess `bash`. No memory or audit by default.
- An `E2B` Runtime provisions a Workspace inside a Firecracker microVM. Same default substrates as Local, plus snapshot, pause, and port-forward at the Runtime level.
- An `AgentFS-backed` Runtime (could be Local or E2B-or-anywhere) provisions a Workspace whose FS is AgentFS — overlay, tool-isolation, audit, export all light up.
- A `Mirage-backed` Runtime provisions a Workspace whose FS includes mounted external backends.
- A `just-bash`-backed Runtime provisions a Workspace whose Shell runs in-process — no isolation, no subprocess, but extremely fast. Suitable for tests and CI smoke runs.

The decoupling is the point: any reasonable combination of Runtime + Workspace substrates should compose.

### 5.1 The `Local` runtime

Reference Local runtime executes child processes on the host machine. Simplest implementation, most useful for testing.

### 5.2 The `Mock` runtime

For testing. Holds an in-memory filesystem and a deterministic terminal that the test can prime with canned output. Bundled with the reference implementation.

---

## 6. Tools — the bridge for non-ACP agents

Because the Workspace is the source of truth, **Tools are a Host-side convenience, not a protocol primitive.** They exist purely to bridge agents whose Reasoning layer expects specific function shapes (`bash(command)`, `read_file(path)`) into the Workspace's ACP-shaped methods.

```
interface Tool {
  name() -> string
  json_schema() -> json                         // function-calling signature
  acp_mapping() -> { kind: read|edit|execute|… } // hint for tool_call_update
  execute(args, workspace) -> result             // routes through Workspace methods
}
```

### 6.1 Built-in tools

The reference implementation ships a baseline ToolSet that any function-calling agent can be wired with:

| Tool | Schema | Backed by |
|---|---|---|
| `Tools::Bash` | `bash(command, cwd?, timeout?, env?)` | Workspace shell substrate (`terminal/*`) |
| `Tools::ReadFile` | `read_file(path, line?, limit?)` | Workspace FS substrate (`fs/read_text_file`) |
| `Tools::WriteFile` | `write_file(path, content)` | Workspace FS substrate (`fs/write_text_file`) |
| `Tools::Edit` | `edit(path, old_text, new_text)` | Workspace FS substrate (read + write) |
| `Tools::Search` | `search(pattern, path?, glob?)` | Workspace shell (`grep`/`rg`) or FS extension |
| `Tools::Memory` | `memory_get/set/list(...)` | Workspace memory substrate (capability: `memory.kv`) |
| `Tools::Plan` | `plan(items)` | Synthetic — emits ACP `plan` update |

### 6.2 Tool surface adapters

For tool surfaces defined by other projects:

- `Tools::MCPBridge` — exposes an external MCP server's tools through the same interface, so an MCP-only agent gets the same routing as a function-calling one.
- `Tools::Mirage` — exposes Mirage's bash-style commands over mounted heterogeneous backends; the Workspace's `fs.mount` capability provides the backing.

### 6.3 What the Tools layer is *not*

It is not a way to add abilities the Workspace does not have. If an agent wants to read S3, the Workspace must expose `mount.s3`; a `Tools::ReadS3` shim that reaches around the Workspace would break the isolation, audit, and portability contracts. Tools are translators between agent-facing schemas and the Workspace's ACP methods — nothing more.

---

## 7. Capabilities are the negotiation primitive

There is no "Solid Agent compliance level 1.0". Implementations advertise capabilities; hosts require them; the session creation either succeeds or fails with `capability_unsatisfied`. This pattern:

- Lets Runtimes adopt SAP without implementing the entire surface (a fast-spinning serverless Runtime with no pause/resume is still useful).
- Lets Workspaces advertise rich substrates (`fs.overlay`, `audit.tool_calls`) without burdening simpler Workspaces with stubbed methods.
- Lets new capabilities ship without a protocol version bump.
- Gives Hosts a precise reason when a pairing won't work, instead of a runtime crash deep into a tool call.

The trade-off: capability sprawl. The roadmap proposes a small **capability registry** maintained in this repository, with RFC review before new capabilities are added.

---

## 8. Manifest — the Workspace seed

The Manifest declares the *initial Workspace state* a session should start with. Everything else — prompts, tool calls, responses — is per-turn.

```
- repos:      git repositories to clone
- files:      file contents to seed (inline / URL / local / secret)
- env:        environment variables (literal or secret-store references)
- mounts:     pluggable backends to attach (kind + options)
- skills:     bundled instructional content (AGENTS.md, skill packs)
- memory:     key/value seeds for the Memory substrate
- capabilities_required:   forward-declared minimum capability set
```

The manifest is declarative, wire-portable JSON. Loading from YAML is a Host convenience. The same manifest works whether the Host is Ruby, Python, TypeScript, or Go.

The manifest is influenced by the OpenAI Agents SDK's `Manifest`, by Mirage's mount-as-path model, and by AgentFS's portable-workspace model. SAP's contribution is to make the format wire-portable and to define `runtime/manifest_apply` so manifests can be re-applied mid-session.

---

## 9. Composition with workflow engines

SAP itself is per-session. A workflow engine — Ruby DAG library, Temporal workflow, Inngest function, Airflow DAG — composes sessions by:

1. Creating a session for each workflow node.
2. Subscribing to the session's update stream and persisting it.
3. Pinning multiple nodes to the **same Workspace handle** (via reference) so they share state, while each runs with its own Agent.
4. Calling `runtime/snapshot` or `fs.snapshot` between nodes for rollback points.
5. Wrapping each node's tool calls in `fs.overlay` for tool-call-grained rollback if the Workspace advertises it.
6. Closing the session when the node is done.

A multi-agent pipeline falls out naturally: Codex implements a feature on an E2B-hosted AgentFS Workspace; Claude Code reviews the same Workspace; a final mock-agent smoke-tests it. Each agent sees the same files, the same audit log, the same memory. Solid Agent does not bundle a workflow engine; the protocol is the meeting point.

---

## 10. Reference implementation plan

A Ruby reference implementation is planned in parallel with this specification, but **the spec is the authoritative artifact**. Concretely:

- `lib/solid_agent/agents/` — built-in Agent adapters (Claude Code, Codex, ACPDirect, Mock).
- `lib/solid_agent/runtimes/` — built-in Runtime adapters (Local, E2B, Mock initially).
- `lib/solid_agent/workspaces/` — built-in Workspace substrate adapters (POSIX, AgentFS-backed, Mirage-backed, just-bash-backed).
- `lib/solid_agent/tools/` — built-in Tool definitions and surface adapters.
- `lib/solid_agent/wire/` — ACP and SAP message types, serializers, transports.
- `lib/solid_agent/manifest.rb` — Manifest type, validation, application orchestration.
- `lib/solid_agent/server.rb` — HTTP/SSE server exposing the surface in [PROTOCOL §2.2](./PROTOCOL.md#22-http-transport-remote-used-for-cloud-runtimes).

Independent implementations in TypeScript, Python, Rust, and Go are very welcome. They should be wire-compatible with the Ruby reference, not API-compatible.

---

## 11. Known limitations and explicit non-goals

- **No standardisation of Reasoning.** SAP does not specify how an Agent reaches an LLM, what model is used, or how cognition is structured.
- **No multi-tenant auth model.** A Host running Solid Agent on behalf of multiple end-users handles its own auth boundary.
- **No built-in persistence at the Host level.** Sessions are stateful in memory while running; persisting the event stream is the Host's concern. The *Workspace's* audit substrate is in scope; the Host's event store is not.
- **No "skills registry."** Skill bundles live inside manifests; how a skill is authored or distributed is out of scope.
- **No editor/UI rendering rules.** SAP says nothing about how a Host should display diffs, terminals, or thinking blocks.
- **Tools are a convenience, not a primitive.** A Host that prefers per-agent translation may bypass the Tools layer entirely.

---

## 12. Influences and prior art

- **[Agent Client Protocol](https://agentclientprotocol.com/)** — wire vocabulary, capability handshake, permission-request model.
- **[SandboxAgent](https://sandboxagent.dev/) (Rivet)** — ACP-over-HTTP/SSE, two-namespace HTTP surfaces, proof that the pattern works in practice.
- **OpenAI Agents SDK `agents.sandbox`** — the `Manifest` concept and the canonical seven-provider Runtime list.
- **[Model Context Protocol](https://modelcontextprotocol.io/) (MCP)** — content-block shapes and JSON-RPC stdio framing; informs the Tools layer's MCP bridge.
- **[Language Server Protocol](https://microsoft.github.io/language-server-protocol/)** — capability-negotiation pattern over JSON-RPC.
- **[AgentFS](https://github.com/tursodatabase/agentfs) (Turso)** — the argument that an agent's filesystem differs in kind from a human's. Directly motivates the `fs.overlay`, `fs.tool_isolation`, `memory.kv`, `audit.tool_calls`, and `fs.export` capabilities; the canonical reference for an AgentFS-backed Workspace.
- **[Mirage](https://github.com/strukto-ai/mirage) (Strukto)** — the argument that heterogeneous external data sources belong in the Workspace as mounted paths, not as N separate tools. Directly motivates the `fs.mount` capability family and the manifest's `mounts:` shape.
- **[just-bash](https://github.com/vercel-labs/just-bash) (Vercel Labs)** — the argument that the shell is the agent's primary verb and its environment is constitutive of agent behaviour. Directly motivates the `exec.in_process`, `exec.languages`, `exec.network`, and `exec.limits` capabilities; the canonical reference for an in-process Shell substrate.

If you have a related effort that should be acknowledged or cross-referenced, please open a PR against this section.
