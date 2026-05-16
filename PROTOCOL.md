# Solid Agent Protocol

> **Status:** Draft 0.1 — non-normative, for discussion.
> **Foundation:** [Agent Client Protocol (ACP)](https://agentclientprotocol.com/). Solid Agent does not redefine the wire format; it adopts ACP and adds a small extension for runtime lifecycle.

## 1. What this protocol is

The Solid Agent protocol specifies how to **wrap existing agent SDKs and sandbox runtimes** so that any wrapped agent can run on any wrapped runtime. It is a thin layer on top of ACP, not a replacement.

Concretely, the protocol defines:

1. How a coding-agent SDK (e.g., Claude Code, Codex, AmpCode, Cursor, Pi) exposes itself as an **ACP-speaking Agent**, including translation rules from the SDK's native event vocabulary to ACP `session/update` variants.
2. How a sandbox runtime (e.g., Local subprocess, E2B, Daytona, Modal) exposes itself as an **ACP-speaking client side** that fulfils `fs/*` and `terminal/*` methods.
3. A small set of **runtime lifecycle methods** ACP does not specify: `runtime/provision`, `runtime/finalize`, `runtime/snapshot`, `runtime/pause`, `runtime/resume`, `runtime/host_for_port`, `runtime/manifest_apply`.
4. A wire-portable **Manifest** for seeding the runtime at session creation.
5. A **capability registry** so an Agent and a Runtime negotiate compatibility up front.

A reference Ruby implementation lives alongside this specification. The protocol is wire-level and language-agnostic; other implementations are welcome.

### 1.1 What this protocol is not

- **Not a replacement for ACP.** Sessions, prompts, tool calls, permission flows, and event streaming are ACP verbatim.
- **Not a tool/resource registry.** That is [MCP](https://modelcontextprotocol.io/)'s job. MCP servers compose with Solid Agent — an Agent can use MCP tools while running inside a Solid Agent runtime.
- **Not a workflow engine.** Per-session primitives only. DAGs, retries, and human approval gates compose above.
- **Not a workspace substrate spec.** The runtime provides a working directory and a shell. Filesystem overlays, mounted heterogeneous backends, in-process shells, memory and audit stores are intentionally out of scope.
- **Not an editor protocol.** Rendering is the Host's job.

## 2. Roles

- **Host** — the application orchestrating an agent run. Typically a developer tool, IDE, CLI, web app, or CI job.
- **Agent** — a coding-agent backend, exposed via a Solid Agent adapter that translates the backend's native protocol into ACP-shaped messages.
- **Runtime** — a sandbox or execution environment, exposed via a Solid Agent adapter that fulfils ACP's client-side methods plus the lifecycle extensions defined here.

The Host pairs one Agent with one Runtime at session creation. Composition is **composable, not orthogonal** — Agents and Runtimes couple through capability declarations, and incompatible pairs are rejected at the handshake (see ADR-0001).

## 3. Transport

Two interchangeable transports.

### 3.1 stdio

JSON-RPC 2.0 framed by newline-delimited JSON over stdin/stdout. Identical to ACP's stdio transport.

### 3.2 HTTP + SSE

REST + Server-Sent Events. Wire payloads are the same JSON-RPC envelopes as stdio. Routing:

```
POST   /v1/sessions                          # create a session
GET    /v1/sessions/{id}/stream              # SSE — session/update notifications
POST   /v1/sessions/{id}/rpc                 # JSON-RPC requests to the agent
POST   /v1/sessions/{id}/permission/{req_id} # respond to session/request_permission
DELETE /v1/sessions/{id}                     # close the session

GET    /v1/sessions/{id}/fs/file?path=…      # runtime — read_text_file
POST   /v1/sessions/{id}/fs/file             # runtime — write_text_file
POST   /v1/sessions/{id}/terminals           # runtime — terminal/create
GET    /v1/sessions/{id}/terminals/{tid}/stream  # runtime — terminal/output (SSE)
POST   /v1/sessions/{id}/terminals/{tid}/wait    # runtime — terminal/wait_for_exit
POST   /v1/sessions/{id}/terminals/{tid}/kill    # runtime — terminal/kill
POST   /v1/sessions/{id}/terminals/{tid}/release # runtime — terminal/release

POST   /v1/sessions/{id}/snapshot            # runtime/snapshot (cap: snapshot)
POST   /v1/sessions/{id}/pause               # runtime/pause   (cap: pause)
POST   /v1/sessions/{id}/resume              # runtime/resume  (cap: pause)
POST   /v1/sessions/{id}/manifest            # apply manifest mid-session
```

The HTTP surface is intentionally compatible with [SandboxAgent](https://sandboxagent.dev/)'s `/v1/acp/*` namespacing.

## 4. Sessions

A session has:

- `session_id` — opaque string.
- `agent` selector + agent-specific options.
- `runtime` selector + runtime-specific options.
- Optional `manifest` (§7).
- Negotiated capabilities (§5).

### 4.1 Session creation

`POST /v1/sessions` (HTTP) or `solid_agent/session/new` (stdio):

```json
{
  "agent": {
    "kind": "claude_code",
    "options": { "model": "sonnet", "permission_mode": "ask" }
  },
  "runtime": {
    "kind": "e2b",
    "options": { "template": "ubuntu-22-04", "auto_pause": true }
  },
  "manifest": { "...": "see §7" },
  "client_capabilities": { "fs": true, "terminal": true, "diff": true }
}
```

Response:

```json
{
  "session_id": "sess_01HXYZ…",
  "agent_capabilities": { "thinking": true, "structured_output": true, "image_input": false, "interrupt": true },
  "runtime_capabilities": { "fs": true, "terminal": true, "snapshot": true, "pause": true, "port_forward": true },
  "negotiated": { "fs": true, "terminal": true, "diff": true, "snapshot": true, "pause": true }
}
```

If a required capability is missing, session creation fails with `capability_unsatisfied` (§8).

### 4.2 Prompting

After creation, the Host calls ACP's `session/prompt`. The Agent emits `session/update` notifications.

`session/update` variants follow ACP's `SessionUpdate` union. Solid Agent adds two small variants:

| Variant | Source | Purpose |
|---|---|---|
| `runtime_status` | Solid Agent | `{ status: "provisioning" | "ready" | "paused" | "resumed" | "lost" }`. |
| `usage_update` | Solid Agent | Periodic `{ input_tokens, output_tokens, cache_read_tokens, cost_usd }`. Optional. |

### 4.3 Permission, cancellation, close

ACP verbatim — `session/request_permission`, `session/cancel`, `solid_agent/session/close`.

## 5. Capability registry

### 5.1 Agent capabilities

| Capability | Meaning |
|---|---|
| `thinking` | Emits `agent_thought_chunk` updates. |
| `structured_output` | Honours a JSON schema in the prompt request. |
| `image_input` | Accepts `image` content blocks in prompts. |
| `tools_external` | Accepts an externally-defined tool list. |
| `session_load` | Can resume a previous session by id. |
| `interrupt` | Honours `session/cancel`. |

### 5.2 Runtime capabilities

| Capability | Meaning |
|---|---|
| `fs` | Implements `fs/read_text_file` and `fs/write_text_file`. |
| `fs.list` | Directory listing. |
| `fs.watch` | Directory watching. |
| `terminal` | Implements `terminal/*`. |
| `terminal.pty` | Full PTY (resize, raw mode). |
| `snapshot` | Runtime-level snapshot of the entire workspace + processes. |
| `pause` | Pause / resume the underlying compute. |
| `port_forward` | Expose a TCP port reachable from the public internet. |
| `gpu` | At least one GPU available. |

### 5.3 Host capabilities

| Capability | Meaning |
|---|---|
| `diff` | Renders `diff` content in `tool_call_update`. |
| `terminal_view` | Renders live terminal output. |
| `permission_ui` | Surfaces `request_permission` interactively. |

Capabilities are advertised at `initialize` and intersected at `session/new`. Mismatches fail fast.

## 6. Agent contract — wrapping an SDK

An Agent adapter wraps an existing coding-agent SDK. The adapter's responsibilities:

1. **Translate session lifecycle** — map the SDK's session/thread/run primitives onto ACP's `initialize`, `session/new`, `session/prompt`, `session/cancel`, `session/close`.
2. **Translate the event stream** — map the SDK's native messages (Claude's `AssistantMessage`/`UserMessage`/`ResultMessage`, Codex's `Notification`s, etc.) onto ACP's `session/update` variants. See [COMPATIBILITY.md](./COMPATIBILITY.md) for the mapping per SDK.
3. **Translate permission requests** — bridge the SDK's permission callback (e.g., `can_use_tool` in claude-agent-sdk-ruby, `approval_handler` in codex-rb) to ACP's `session/request_permission`.
4. **Route filesystem and terminal calls through the Runtime** — when the wrapped SDK wants to read a file or run a command, the adapter MUST route through the bound Runtime's ACP client-side methods rather than touching the host filesystem directly. This is what makes Agent + Runtime composable.

ACP-native agents (any agent that ships ACP support directly) require no translation; the `acp_direct` adapter pipes JSON-RPC through unchanged.

### 6.1 Required methods (ACP)

| Method | Direction |
|---|---|
| `initialize` | Host → Agent |
| `session/new` | Host → Agent |
| `session/prompt` | Host → Agent |
| `session/cancel` | Host → Agent (notification) |
| `session/update` | Agent → Host (notification) |
| `session/request_permission` | Agent → Host |

Optional: `authenticate`, `session/load`, `session/set_mode` (all ACP).

## 7. Runtime contract — wrapping a sandbox

A Runtime adapter wraps a sandbox SDK or local subprocess. It implements:

1. **ACP client-side methods.** `fs/read_text_file`, `fs/write_text_file`, `terminal/create`, `terminal/output`, `terminal/wait_for_exit`, `terminal/kill`, `terminal/release`. Plus optional `fs/list`, `fs/watch`, `terminal/resize`.
2. **Solid Agent lifecycle methods.** Listed below.
3. **The wire bridge for the wrapped Agent.** Most agent SDKs ship a CLI subprocess; the Runtime is responsible for running that subprocess inside its boundary and streaming its stdio back to the adapter. Reference: `claude-agent-sdk`'s pluggable `Transport` seam.

### 7.1 Lifecycle methods

| Method | Capability |
|---|---|
| `runtime/provision` | (always) — apply the manifest, prepare the workspace |
| `runtime/finalize` | (always) — tear down, release resources |
| `runtime/manifest_apply` | (always when manifest support advertised) |
| `runtime/snapshot` | `snapshot` — return a `snapshot_id` for later resume |
| `runtime/pause` | `pause` — preserve state, stop billing for compute |
| `runtime/resume` | `pause` — restore state from a paused runtime |
| `runtime/host_for_port` | `port_forward` — expose an internal TCP port |

```json
// runtime/snapshot
{ "label": "post-test-pass" }
// →
{ "snapshot_id": "snap_01HXYZ…", "created_at": "2026-05-15T10:00:00Z", "size_bytes": 1234567 }
```

A `snapshot_id` may be passed as `runtime.options.snapshot_id` at a later `session/new` call to bootstrap from the snapshot.

## 8. Manifest

The Manifest declares initial workspace state at session creation. JSON document; Hosts may load it from YAML.

```json
{
  "version": 1,
  "repos": [
    { "name": "app", "url": "git@github.com:org/app.git", "ref": "main", "dest": "workspace/app", "depth": 1 }
  ],
  "files": [
    { "path": "AGENTS.md", "source": { "kind": "inline", "content": "# Project rules\n\nUse Ruby 3.2+." } },
    { "path": ".env",      "source": { "kind": "url", "url": "https://example.com/secrets/env" }, "mode": "0600" }
  ],
  "env": {
    "GITHUB_TOKEN":   { "source": "secret", "ref": "github_token" },
    "FEATURE_FLAG_X": "true"
  },
  "capabilities_required": ["fs", "terminal"]
}
```

- `files.source.kind` MAY be `inline`, `url`, `local_path` (Host-resolved), or `secret`.
- `env` values may be plain strings or `{ source: "secret", ref: "<name>" }`. Secret resolution is out of scope.
- `capabilities_required` is a forward-compatible hint; session creation fails capability negotiation if any are missing.

Manifests MAY be applied at session creation or re-applied mid-session via `runtime/manifest_apply`.

## 9. Errors

JSON-RPC errors use the standard envelope. Solid Agent defines:

| Code | Name | Meaning |
|---|---|---|
| `-32000` | `capability_unsatisfied` | A required capability is not advertised. |
| `-32001` | `manifest_invalid` | The manifest failed schema validation or application. |
| `-32002` | `runtime_unavailable` | The Runtime provider is unreachable. |
| `-32003` | `runtime_lost` | The Runtime disappeared mid-session. |
| `-32004` | `agent_unavailable` | The Agent backend is unreachable or unauthenticated. |
| `-32005` | `session_closed` | The session has already been closed. |
| `-32006` | `permission_denied` | A permission request was rejected. |
| `-32007` | `quota_exceeded` | The Host's quota is exhausted. |
| `-32099` | `internal` | Catch-all. `data` SHOULD include a human-readable message. |

## 10. Identifiers, time, versioning

- All IDs are opaque strings. ULID-like or UUIDv7 recommended.
- Timestamps are RFC 3339 with millisecond precision.
- Single integer version field on `initialize`: `protocol_version: 1`. Additive changes do not bump; breaking changes do.

## 11. Reconnection

- **Stdio** — no reconnection unless the Agent supports `session/load`.
- **HTTP** — the Host MAY reopen the SSE `stream` endpoint with a `Last-Event-ID` header. Implementations SHOULD buffer `session/update` events for at least 60 seconds.

## 12. Security boundary

- The Host is **trusted** by definition.
- The Agent is **partially trusted** — it can read and write within the Runtime, but cannot escape it.
- The Runtime is the **isolation boundary**.
- Tool calls that require permission MUST be gated by `session/request_permission`. A Host MAY install an auto-allow policy, but the request MUST still be visible on the wire.

## 13. Conformance profiles

- **Minimal Agent** — `initialize`, `session/new`, `session/prompt`, `session/cancel`, `session/update`.
- **Minimal Runtime** — `fs` and `terminal` capabilities; `runtime/provision`, `runtime/finalize`, `runtime/manifest_apply`.
- **Persistent Runtime** — Minimal Runtime plus `pause` and `snapshot`.

Each named profile is a starting point; capability declarations (§5) are the source of truth.

## 14. Comparison with adjacent specs

| Concern | Solid Agent | ACP | SandboxAgent | OpenAI Sandbox |
|---|---|---|---|---|
| Agent↔Host wire | ACP (re-used) | Native | ACP-based | SDK-only |
| Runtime↔Host wire | ACP client side + Solid Agent lifecycle | Partial | Bespoke REST | Library binding |
| Manifest | First class | None | None | First class |
| Runtime snapshot / pause / resume | First class | None | Partial | None |
| HTTP/SSE transport | Optional | Not standardised | Yes | N/A |
| Multi-vendor agent wrapping | Specified | Implicit | Implicit | OpenAI-centric |

Solid Agent's net additions over ACP: a **Runtime lifecycle contract** (provision, finalize, pause, resume, snapshot, port_forward, manifest_apply), a **Manifest** ACP does not have, an **HTTP/SSE transport** with reconnection semantics, and **explicit wrapping rules** for non-ACP-native agent SDKs.

## 15. What's not in v0.1

These are intentional omissions, not bugs:

- Filesystem overlays, snapshots-at-file-level, tool-call isolation, portable workspace export.
- Mounted heterogeneous backends (S3, Drive, Slack, GitHub, …).
- In-process shell substrates and pluggable interpreters.
- Memory and audit stores as runtime-served substrates.
- A host-side Tools layer for adapting function-calling agents to ACP-shaped runtimes (Host-side concern; not on the wire).
- `session/steer` for mid-turn input injection.
- Approval-guardian capabilities.

If a future version adopts any of these, it will be via an RFC that adds explicit wire methods and capabilities. See [docs/adr/](./docs/adr/) for the discipline.

## Open questions

1. HTTP transport: REST+SSE only, JSON-RPC over WebSocket, or both?
2. `usage_update` in core or as an observability extension?
3. Hierarchical (`fs.list`) vs flat (`fs_list`) capability names?
4. Standardised reconnect-buffer window (current floor: 60s)?
5. Should `runtime/snapshot` semantics specify guarantees about open file descriptors and in-flight terminal sessions?
6. Egress-policy hints in the manifest?

## Change log

- **0.1** (this draft) — Initial public draft. ACP-foundation, runtime lifecycle extension, manifest, capability registry. Scope: wrapping coding-agent SDKs and sandbox runtimes.
