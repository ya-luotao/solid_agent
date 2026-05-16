# Solid Agent — Design Notes

> Companion to [PROTOCOL.md](./PROTOCOL.md). Explains the why behind the wire spec and the shape of the Ruby reference implementation.

## 1. The problem

The coding-agent ecosystem has split along two independent axes:

- **Agents** — Claude Code, Codex, AmpCode, Cursor, Pi. Each has its own SDK, its own wire format, its own assumptions about tools.
- **Sandbox runtimes** — Local subprocess, E2B, Daytona, Modal, Cloudflare Containers, Vercel Sandboxes, Runloop, Blaxel, OpenAI built-in. Each has a different isolation model, a different feature set (snapshot, pause, port forward), and a different lifecycle.

Today every project that wants to combine N agents with M runtimes pays an `N × M` integration cost. **Solid Agent's premise is that the cost should be `N + M`.** Write one adapter per agent SDK that turns it into an ACP-speaking Agent. Write one adapter per sandbox SDK that turns it into an ACP-speaking client side. They compose.

## 2. Why ACP

The [Agent Client Protocol](https://agentclientprotocol.com/) already specifies most of what an Agent↔Host wire format needs:

- Session lifecycle (`initialize`, `session/new`, `session/prompt`, `session/cancel`).
- Streaming events (`session/update` with `agent_message_chunk`, `tool_call`, `tool_call_update`, `plan`, …).
- Per-tool permission flow (`session/request_permission` with `allow_once`/`allow_always`/`reject_*`).
- Client-side methods every coding agent needs (`fs/read_text_file`, `fs/write_text_file`, `terminal/create`, `terminal/output`, `terminal/wait_for_exit`, `terminal/kill`, `terminal/release`).
- Capability negotiation between Agent and Client.

Solid Agent adopts all of this verbatim. The only things it adds are:

1. **Runtime lifecycle.** ACP does not specify how a runtime is provisioned, snapshotted, paused, resumed. We add `runtime/provision`, `runtime/finalize`, `runtime/snapshot`, `runtime/pause`, `runtime/resume`, `runtime/host_for_port`, `runtime/manifest_apply`.
2. **Manifest.** ACP does not specify a session-creation seed. We add a JSON manifest with `repos`, `files`, `env`, `capabilities_required`.
3. **HTTP/SSE transport.** ACP standardises stdio. We add a REST+SSE transport for remote runtimes.
4. **Wrapping rules.** ACP doesn't say how to wrap a non-ACP agent SDK. We document the translation patterns for the major SDKs.

That's the whole protocol surface above ACP. Everything else is reuse.

## 3. Composable, not orthogonal

Earlier framings called the Agent and Runtime axes "orthogonal." That phrasing is wrong and a careful reader will reject it.

- Claude Code's tool schema is baked into its training. A runtime that drops the shell or replaces the filesystem with something exotic breaks Claude Code, full stop.
- Capabilities like `snapshot` only matter if the Host knows about them and uses them. Coupling is real.

The right framing is **composable with explicit capability negotiation**. Agents declare what they need; Runtimes declare what they offer; the Host configures the pairing; capability intersection at `session/new` either succeeds or fails fast with `capability_unsatisfied`. No silent breakage. See [ADR-0001](./docs/adr/0001-composable-not-orthogonal.md) for the discipline.

## 4. How a wrapped Agent works

Every Agent adapter does four things:

```
┌────────────────────────────────────────────────────────┐
│ Solid Agent Agent adapter                              │
│                                                         │
│  ┌──────────────┐    ACP session/*    ┌─────────────┐  │
│  │  Translate   │ ◄───────────────────► │ Host        │  │
│  │  lifecycle   │                       └─────────────┘  │
│  └──────┬───────┘                                       │
│         │                                                │
│  ┌──────▼───────┐                                       │
│  │  Translate   │  ← consume SDK events from the wrapped │
│  │  events      │    backend, emit ACP session/update    │
│  └──────┬───────┘                                       │
│         │                                                │
│  ┌──────▼───────┐                                       │
│  │  Translate   │  ← bridge can_use_tool / approval      │
│  │  permission  │    handler ↔ session/request_permission│
│  └──────┬───────┘                                       │
│         │                                                │
│  ┌──────▼───────┐    ACP fs/terminal  ┌─────────────┐  │
│  │  Route fs +  │ ◄───────────────────► │ Runtime     │  │
│  │  terminal    │                       └─────────────┘  │
│  └──────────────┘                                       │
└────────────────────────────────────────────────────────┘
```

The last box is the most important. When the wrapped SDK wants to read a file or run a command, the adapter does **not** reach the host filesystem directly. It routes through the bound Runtime's ACP methods. This is what makes Agent + Runtime composable: the Agent's view of the world is mediated by the Runtime, and replacing the Runtime replaces that view without changing the Agent.

For Agents that ship native ACP support, the `acp_direct` adapter pipes JSON-RPC through unchanged — no translation work.

## 5. How a wrapped Runtime works

Every Runtime adapter does three things:

1. **Implement ACP client-side methods.** `fs/read_text_file`, `fs/write_text_file`, `terminal/*`. These are mandatory once the corresponding capabilities are advertised.
2. **Implement Solid Agent lifecycle.** `runtime/provision` (apply the manifest, prepare workspace), `runtime/finalize` (tear down), plus optional `snapshot`, `pause`, `resume`, `host_for_port`, `manifest_apply` gated on capabilities.
3. **Provide a transport bridge for the wrapped Agent.** Most agent SDKs spawn a CLI subprocess. The Runtime is responsible for running that subprocess inside its isolation boundary and streaming its stdio back to the adapter.

The transport bridge is the single most consequential part of the design. The reference implementation pattern is to give each Agent adapter a `transport_factory` argument that returns a transport bound to a specific Runtime. For example, `Agents::ClaudeCode` configures `claude-agent-sdk-ruby`'s `Client.new(transport_class: …)` with a transport that runs the Claude CLI subprocess through E2B's command RPC instead of the local machine's `Open3`. The Agent code is unchanged across runtimes; the transport class changes.

## 6. Ruby reference implementation

The reference implementation is a Ruby gem (`solid_agent`). Sketch:

```
lib/solid_agent/
  agent.rb             # base class: agent_kind, capabilities, lifecycle methods
  sandbox.rb           # base class: kind, capabilities, fs/terminal/lifecycle methods
  session.rb           # Host-side session orchestration
  manifest.rb          # Manifest type + validation
  events/              # SolidAgent::TextEvent, ToolEvent, ResultEvent, …
  wire/                # ACP message types, serialisers, transports (stdio + HTTP/SSE)
  agents/
    claude.rb          # wraps claude-agent-sdk
    codex.rb           # wraps codex-rb
    acp_direct.rb      # pipes JSON-RPC for any ACP-native agent
    mock.rb            # deterministic test double
  sandboxes/
    local.rb           # Open3-based subprocess
    e2b.rb             # wraps the e2b gem
    mock.rb            # in-memory FS + scripted terminal
  server.rb            # Rack app exposing the HTTP/SSE surface
```

The public API stays Ruby-idiomatic:

```ruby
result = SolidAgent.run('Fix the failing test', agent: :claude, sandbox: :e2b)
result.text       # final assistant text
result.cost_usd   # total cost
result.events     # array of SolidAgent::Event objects
```

Internally `SolidAgent.run` constructs a `Session`, calls `runtime/provision`, sends `session/prompt`, collects `session/update` events, and finalises. Each Agent and Sandbox is a Ruby class; pairing them is one line.

Other-language implementations are welcome. The protocol is wire-level so a TypeScript, Python, Rust, or Go implementation should be wire-compatible without being API-compatible.

## 7. Composition with workflow engines

Solid Agent is per-session. A workflow engine — Ruby DAG library, Sidekiq, Temporal, Inngest, Airflow — composes sessions by:

1. Creating a session per workflow node.
2. Subscribing to `session/update`, persisting if desired.
3. Pinning multiple nodes to the same Runtime handle (via reference) so they share workspace state.
4. Calling `runtime/snapshot` between nodes for rollback points.
5. Closing the session when the node is done.

A multi-agent pipeline falls out naturally: Codex implements a feature on an E2B sandbox, Claude Code reviews the same sandbox, a final smoke test runs. All three see the same files because they share the Runtime.

Solid Agent does not bundle a workflow engine; the protocol is the meeting point.

## 8. Solid Agent and MCP

[MCP](https://modelcontextprotocol.io/) answers *"where do tools come from?"* — a discoverable surface of tools, resources, and prompts an agent can call. Solid Agent answers *"where does the agent run?"* — session lifecycle, runtime provisioning, snapshot/pause/resume, manifest seeding, transport.

They compose at different layers. An Agent inside a Solid Agent runtime can register MCP servers as part of its tool catalogue; the Runtime hosts those MCP servers in its process tree; the manifest can carry their configuration. Solid Agent does not duplicate MCP — universal operations (fs, terminal) live in ACP's client-side methods (built into the runtime); domain-specific ones (GitHub, Slack, vector search) live in MCP.

## 9. Out of scope (and proud of it)

The following ideas are intentionally not in v0.1:

- **Filesystem overlays, mount kinds, audit substrates.** Real but premature. The runtime provides a working directory and a shell; richer filesystem semantics can come in a later spec version after usage data accumulates.
- **In-process shell with pluggable interpreters.** Same reasoning.
- **A host-side Tools layer.** Useful but a Host concern; not on the wire.
- **`session/steer`.** Only two agents support it today and the semantics differ; convergence first, standardisation second.
- **Approval guardian patterns.** Worth standardising eventually; needs audit infrastructure first.
- **Workflow engine.** Use the one you already have.

These are not roadmap items hidden in a parking lot; they are decisions to keep v0.1 small.

## 10. Influences

- **[Agent Client Protocol](https://agentclientprotocol.com/)** — the wire format we adopt.
- **[SandboxAgent](https://sandboxagent.dev/) (Rivet)** — proof that ACP works over HTTP/SSE; HTTP namespace inspiration.
- **OpenAI Agents SDK `agents.sandbox`** — Manifest concept; canonical runtime provider list.
- **[Model Context Protocol](https://modelcontextprotocol.io/)** — the peer at the tool-registry layer that Solid Agent doesn't try to replace.
- **Language Server Protocol** — capability negotiation pattern over JSON-RPC.
