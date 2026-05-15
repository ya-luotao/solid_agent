# Solid Agent Protocol — Draft

> **Status:** Draft 0.1 — non-normative, for discussion only. Subject to breaking change.
> Open questions are flagged inline with **[OQ-n]** and indexed at the end.

## 1. Scope and goals

The Solid Agent Protocol (SAP) defines the wire contract between three roles:

- **Host** — the application that wants to orchestrate one or more coding-agent runs. Typically a developer tool, IDE, web app, CI job, or autonomous workflow engine.
- **Agent** — a coding-agent backend (Claude Code, Codex, AmpCode, Cursor, Pi, or any other process that can take a prompt and emit a structured response, possibly with tool calls).
- **Runtime** — the execution environment in which an Agent operates (the local machine, an E2B sandbox, a Daytona workspace, a Modal container, an OpenAI built-in sandbox, etc.).

The protocol's job is to make any Host × Agent × Runtime combination interoperable as long as each side implements the corresponding role.

### 1.1 Non-goals

- SAP is **not** a model serving protocol. It does not specify how an Agent reaches an LLM, what model is used, or how billing works.
- SAP is **not** a workflow engine. It defines the per-session primitives that a workflow engine can build on, but DAGs, retries, and human approvals live above SAP.
- SAP is **not** an editor protocol. It does not specify how a UI renders messages or diffs; that is the Host's job.

### 1.2 Design principles

1. **Reuse before reinvent.** The Agent↔Host contract is the [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) verbatim where ACP is defined. SAP only extends ACP where ACP is silent.
2. **Two orthogonal axes.** Agents speak the ACP "agent side"; Runtimes speak the ACP "client side". A Host is free to pair any compliant Agent with any compliant Runtime; capability negotiation handles the cases where they do not match.
3. **Wire-portable manifests.** Workspace seeding (which repos, files, env vars, and mounts the Runtime should provide before the Agent starts) is a first-class JSON document, not a host-language object graph.
4. **Capabilities, not levels.** A compliant implementation declares which capability bundles it supports. There is no "SAP 1.0 compliance level"; there is a set of named capabilities a host can require.
5. **Stable JSON.** All messages are JSON. Field names are `snake_case`. Time fields are RFC 3339. IDs are opaque strings (clients MUST NOT parse them).

## 2. Transport

SAP defines two interchangeable transports.

### 2.1 stdio transport (local, default for in-process spawning)

JSON-RPC 2.0 framed by newline-delimited JSON over stdin/stdout. This is identical to ACP's stdio transport. A Host that spawns an Agent or Runtime as a child process MUST use this transport unless it has out-of-band knowledge that the peer supports HTTP.

### 2.2 HTTP transport (remote, used for cloud Runtimes)

A REST + Server-Sent Events (SSE) surface. The HTTP transport is required when the Host and the Agent or Runtime are not co-located (e.g., the Runtime is a managed sandbox provider). The wire payloads inside HTTP requests and SSE events are the same JSON-RPC 2.0 envelopes as the stdio transport, with the following routing:

```
POST   /v1/sessions                          # create a session
GET    /v1/sessions/{id}/stream              # SSE — session/update notifications from agent
POST   /v1/sessions/{id}/rpc                 # JSON-RPC requests to the agent (session/prompt, …)
POST   /v1/sessions/{id}/permission/{req_id} # respond to session/request_permission

GET    /v1/sessions/{id}/fs/file?path=…      # runtime — read_text_file
POST   /v1/sessions/{id}/fs/file             # runtime — write_text_file
GET    /v1/sessions/{id}/fs/entries?path=…   # runtime — list (extension; see §6.2)
POST   /v1/sessions/{id}/terminals           # runtime — terminal/create
GET    /v1/sessions/{id}/terminals/{tid}/stream  # runtime — terminal/output (SSE)
POST   /v1/sessions/{id}/terminals/{tid}/wait    # runtime — terminal/wait_for_exit
POST   /v1/sessions/{id}/terminals/{tid}/kill    # runtime — terminal/kill
POST   /v1/sessions/{id}/terminals/{tid}/release # runtime — terminal/release

POST   /v1/sessions/{id}/snapshot            # SAP extension — create snapshot
POST   /v1/sessions/{id}/pause               # SAP extension — pause runtime
POST   /v1/sessions/{id}/resume              # SAP extension — resume runtime
POST   /v1/sessions/{id}/manifest            # SAP extension — apply manifest
```

The HTTP surface is intentionally compatible with the namespacing used by [SandboxAgent](https://sandboxagent.dev/). A Host that already speaks SandboxAgent's `/v1/acp/{server_id}` should be able to address Solid Agent's `/v1/sessions/{id}` with only a path-prefix change.

**[OQ-1]** Should the HTTP transport be REST+SSE, JSON-RPC over WebSocket, or both? ACP itself does not pick. SandboxAgent uses REST+SSE for streams; we tentatively do the same.

## 3. Sessions

A **session** is a single coordinated run of one Agent on one Runtime. A session has:

- A unique opaque `session_id` (string).
- An `agent` selector (e.g., `"claude_code"`, `"codex"`, `"acp_direct"`).
- A `runtime` selector (e.g., `"local"`, `"e2b"`, `"daytona"`).
- An optional `manifest` (see §7).
- Negotiated `capabilities` (see §4).

### 3.1 Creating a session

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
  "client_capabilities": { "fs": true, "terminal": true, "diff": true },
  "host_metadata": { "client_name": "my-app", "client_version": "1.2.3" }
}
```

Response:

```json
{
  "session_id": "sess_01HXYZ…",
  "agent_capabilities": { "thinking": true, "structured_output": true, "image_input": false },
  "runtime_capabilities": { "fs": true, "terminal": true, "snapshot": true, "pause": true, "port_forward": true, "gpu": false },
  "negotiated": { "fs": true, "terminal": true, "diff": true, "snapshot": true, "pause": true }
}
```

`negotiated` is the intersection of what the Agent needs, what the Runtime offers, and what the Host advertised in `client_capabilities`. If the intersection is empty for a required capability, the session creation fails with error code `capability_unsatisfied` (see §9).

### 3.2 Prompting

After session creation, the Host calls `session/prompt` (ACP) with the user's prompt. The Agent emits `session/update` notifications as it works. These notifications are the canonical event stream that the Host should display, persist, or forward to higher-level workflow code.

The `session/update` payload follows ACP's `SessionUpdate` union:

- `agent_message_chunk` — streamed text from the assistant.
- `agent_thought_chunk` — streamed "thinking" / chain-of-thought (if the Agent capability `thinking` is true).
- `user_message_chunk` — text from the user (useful for echoing in transcripts).
- `tool_call` — the Agent is about to invoke a tool.
- `tool_call_update` — progress or completion of a tool call. Carries `status: pending | in_progress | completed | failed`, optional `content` (text / diff / terminal handle), and an optional `kind` (`read | edit | delete | move | search | execute | think | fetch | other`).
- `plan` — the Agent has produced an updated plan.
- `available_commands_update` — the Agent's slash-command list changed.
- `current_mode_update` — the Agent's mode (e.g., "ask" vs "auto") changed.

SAP **adds** the following session-update variants (all gated on the `runtime` capability advertising support):

- `runtime_status` — `{ status: "provisioning" | "ready" | "paused" | "resumed" | "snapshotted" | "lost" }`. Emitted by the Runtime, multiplexed into the same stream.
- `usage_update` — periodic `{ input_tokens, output_tokens, cache_read_tokens, cost_usd }`. Optional; not all Agents report.

**[OQ-2]** Should `usage_update` be in SAP core or a separate observability extension? Arguments for core: every Host wants it. Arguments against: shape varies wildly between providers.

### 3.3 Permission requests

When an Agent needs the Host's permission to invoke a tool, it sends an ACP `session/request_permission` request. SAP does not alter this contract. The Host responds with one of `{allow_once, allow_always, reject_once, reject_always, cancelled}`.

### 3.4 Cancellation

The Host sends ACP's `session/cancel` notification at any time. The Agent SHOULD stop streaming as soon as possible and emit a final `session/prompt` response with `stop_reason: cancelled`.

### 3.5 Closing

`DELETE /v1/sessions/{id}` (HTTP) or `solid_agent/session/close` (stdio). The Runtime is finalized (kill or release back to a pool). After close, the `session_id` MUST NOT be reused.

## 4. Capability negotiation

A capability is a string name identifying a discrete behaviour. The initial registry:

| Capability | Side | Meaning |
|---|---|---|
| `fs` | Runtime | Implements `fs/read_text_file` and `fs/write_text_file`. |
| `fs.list` | Runtime | Implements directory listing (SAP extension). |
| `fs.watch` | Runtime | Implements directory watching (SAP extension). |
| `terminal` | Runtime | Implements `terminal/create`, `terminal/output`, `terminal/wait_for_exit`, `terminal/kill`, `terminal/release`. |
| `terminal.pty` | Runtime | Terminals are PTYs (resize, raw mode). |
| `snapshot` | Runtime | Implements `POST /v1/sessions/{id}/snapshot`. |
| `pause` | Runtime | Implements pause and resume. |
| `port_forward` | Runtime | Can expose a TCP port reachable from the public internet. |
| `gpu` | Runtime | Provides at least one GPU. |
| `thinking` | Agent | Emits `agent_thought_chunk` updates. |
| `structured_output` | Agent | Honours a JSON schema in the prompt request. |
| `image_input` | Agent | Accepts `image` content blocks in prompts. |
| `diff` | Host | Renders `diff` content in `tool_call_update`. |

Hosts MAY require capabilities at session creation. Runtimes and Agents SHOULD advertise every capability they support, even if unused.

**[OQ-3]** Should capability names be hierarchical (`fs.list`) or flat (`fs_list`)? Hierarchical reads better but requires a parser. SandboxAgent uses flat.

## 5. Agent side — required methods

An Agent implementation MUST implement:

| Method | Direction | Source |
|---|---|---|
| `initialize` | Host → Agent | ACP |
| `authenticate` | Host → Agent | ACP (optional; only if agent gates on auth) |
| `session/new` | Host → Agent | ACP |
| `session/prompt` | Host → Agent | ACP |
| `session/cancel` | Host → Agent (notification) | ACP |
| `session/update` | Agent → Host (notification) | ACP |
| `session/request_permission` | Agent → Host | ACP |

An Agent MAY also implement:

| Method | Notes |
|---|---|
| `session/load` | Resume a previous session by id. ACP. |
| `session/set_mode` | Switch between modes (e.g., ask/auto). ACP. |

The Agent calls the **Runtime** for filesystem and terminal operations using ACP's client-side methods (see §6). The Host's job in those exchanges is to route — when the Agent emits `fs/read_text_file`, the Host hands it to the bound Runtime, returns the response, and may also display the activity in its UI.

## 6. Runtime side — required methods

A Runtime implementation MUST implement at least one capability bundle.

### 6.1 ACP client surface (capability `fs` and `terminal`)

| Method | Direction | Source |
|---|---|---|
| `fs/read_text_file` | Agent → Runtime | ACP |
| `fs/write_text_file` | Agent → Runtime | ACP |
| `terminal/create` | Agent → Runtime | ACP |
| `terminal/output` | Agent → Runtime | ACP |
| `terminal/wait_for_exit` | Agent → Runtime | ACP |
| `terminal/kill` | Agent → Runtime | ACP |
| `terminal/release` | Agent → Runtime | ACP |

### 6.2 SAP extensions

These methods are SAP additions because ACP does not yet specify them. They are gated on the corresponding capability.

#### `fs/list` (capability `fs.list`)

```json
// request
{ "path": "/workspace", "depth": 1 }
// response
{ "entries": [
  { "name": "README.md", "kind": "file", "size": 4096, "mtime": "2026-05-15T10:00:00Z" },
  { "name": "src",        "kind": "dir",  "mtime": "2026-05-15T09:00:00Z" }
]}
```

#### `fs/watch` (capability `fs.watch`)

Subscribes the Agent to file-change notifications. Implementation-defined backpressure.

#### `runtime/snapshot` (capability `snapshot`)

```json
// request
{ "label": "post-test-pass" }
// response
{ "snapshot_id": "snap_01HXYZ…", "created_at": "2026-05-15T10:00:00Z", "size_bytes": 1234567 }
```

Snapshots are opaque to the protocol; a snapshot may be passed as `runtime.options.snapshot_id` at session creation to bootstrap from the snapshot's state.

#### `runtime/pause` and `runtime/resume` (capability `pause`)

```json
// pause request — empty body
// pause response
{ "paused_at": "2026-05-15T10:00:00Z" }

// resume request
{ "timeout_seconds": 30 }
// resume response
{ "resumed_at": "2026-05-15T10:00:31Z" }
```

A paused Runtime preserves filesystem and process state but is not billed for CPU. The session remains alive; the Agent will reconnect on resume.

#### `runtime/manifest_apply` (capability `manifest`)

Re-applies a manifest after the Runtime is already running. See §7.

#### `runtime/host_for_port` (capability `port_forward`)

```json
// request
{ "port": 8080 }
// response
{ "host": "8080-sess-01hxyz.example.dev", "url": "https://8080-sess-01hxyz.example.dev" }
```

## 7. Manifest

A manifest describes the workspace state a session should start with. It is a JSON document; Hosts may load it from YAML or any other source.

```json
{
  "version": 1,
  "repos": [
    {
      "name": "app",
      "url": "git@github.com:org/app.git",
      "ref": "main",
      "dest": "workspace/app",
      "depth": 1
    }
  ],
  "files": [
    {
      "path": "AGENTS.md",
      "source": { "kind": "inline", "content": "# Project rules\n\nUse Ruby 3.2+." }
    },
    {
      "path": ".env",
      "source": { "kind": "url", "url": "https://example.com/secrets/env" },
      "mode": "0600"
    }
  ],
  "env": {
    "GITHUB_TOKEN": { "source": "secret", "ref": "github_token" },
    "FEATURE_FLAG_X": "true"
  },
  "mounts": [
    { "kind": "s3", "bucket": "my-bucket", "prefix": "sessions/", "target": "/mnt/sessions", "mode": "rw" }
  ],
  "skills": {
    "source": "bundled",
    "dirs": ["skills/"]
  },
  "capabilities_required": ["fs", "terminal"]
}
```

Notes:

- `files.source.kind` MAY be `inline`, `url`, `local_path` (Host-resolved), or `secret`.
- `env` values may be plain strings or `{ source: "secret", ref: "<name>" }` references; how a Host or Runtime resolves secrets is out of scope.
- `mounts` is intentionally generic; a Runtime advertises which `kind`s it supports as separate capabilities (e.g., `mount.s3`, `mount.efs`).
- `capabilities_required` is a forward-compatible hint; the session creation will fail capability negotiation if any are missing.

**[OQ-4]** Should the manifest be applied as part of `session/new`, as a separate `runtime/manifest_apply` call, or both? Proposed: both — `session/new` accepts a manifest for the common case, and `runtime/manifest_apply` exists for mid-session reconfiguration.

**[OQ-5]** Should the manifest define `output_dirs` to indicate where session artifacts live? OpenAI's manifest does. Useful for archiving sessions.

## 8. Identifiers and time

- All IDs (`session_id`, `snapshot_id`, `tool_call.id`, `terminal_id`, `request_id`) are opaque strings. The protocol does not require a particular format; ULID-like or UUIDv7 are recommended. Clients MUST NOT parse them or assume ordering.
- All timestamps are RFC 3339 strings with millisecond precision and a `Z` or numeric offset.

## 9. Errors

JSON-RPC errors use the standard envelope. SAP defines the following `code` values in the range `-32000` to `-32099`:

| Code | Name | Meaning |
|---|---|---|
| `-32000` | `capability_unsatisfied` | A required capability is not advertised by the chosen Agent or Runtime. |
| `-32001` | `manifest_invalid` | The manifest failed schema validation or could not be applied. |
| `-32002` | `runtime_unavailable` | The Runtime provider is currently unreachable. |
| `-32003` | `runtime_lost` | The Runtime disappeared mid-session (sandbox eviction, node failure). The Host may retry by reconnecting. |
| `-32004` | `agent_unavailable` | The Agent backend is unreachable or unauthenticated. |
| `-32005` | `session_closed` | The session has already been closed. |
| `-32006` | `permission_denied` | A permission request was rejected. |
| `-32007` | `quota_exceeded` | The Host's quota (concurrent sessions, time, cost) is exhausted. |
| `-32099` | `internal` | Catch-all. The `data` field SHOULD include a human-readable message. |

## 10. Reconnection

Either party MAY disconnect at any time. Reconnect semantics:

- **Stdio transport** — no reconnection. A new child process means a new session unless the Host calls `session/load` with the prior `session_id` and the Agent supports it.
- **HTTP transport** — the Host MAY reopen the SSE `stream` endpoint with a `Last-Event-ID` header to resume from the last delivered event. Implementations SHOULD buffer `session/update` events for at least 60 seconds.

**[OQ-6]** Should we standardise the reconnect-buffer window? Implementations in the wild use anywhere from 30 seconds to several minutes; a recommended floor would help interoperability.

## 11. Versioning

SAP uses a single integer version field on the initial `initialize` handshake. The current draft is `protocol_version: 1`. Breaking changes bump the integer; additive changes do not. Implementations MUST reject sessions whose `protocol_version` is higher than the highest they understand.

## 12. Security model — outline

The full security model is out of scope for this draft, but the boundaries are:

- The Host is **trusted** by definition (it chose the Agent and Runtime).
- The Agent is **partially trusted** — it can read and write within the Runtime's filesystem but cannot escape the Runtime.
- The Runtime is **untrusted** with respect to the Host's secret material — secrets reach the Runtime only via the manifest's explicit `env.source: "secret"` mechanism.
- Tool calls that require permission MUST be gated by `session/request_permission`; a Host MAY install an auto-allow policy, but the protocol still requires the request to be visible.

**[OQ-7]** Should SAP specify a sandbox-egress policy hint in the manifest? Useful for high-security Hosts but couples the spec to Runtime implementations that support egress filtering.

## 13. Conformance levels

Two named profiles are proposed:

- **Minimal Agent** — implements `initialize`, `session/new`, `session/prompt`, `session/cancel`, `session/update`. No `request_permission`, no `thinking`.
- **Minimal Runtime** — implements `fs/read_text_file`, `fs/write_text_file`, `terminal/*`. No snapshot, no pause.

These profiles are the smallest implementations a Host can rely on. Capability negotiation (§4) is how anything richer is expressed.

## 14. Comparison with adjacent specs

| Concern | Solid Agent | ACP | SandboxAgent | OpenAI Sandbox |
|---|---|---|---|---|
| Agent↔Host wire | ACP (re-used) | Native | ACP-based | SDK-only |
| Runtime↔Host wire | ACP client side + SAP extensions | Partial (`fs`, `terminal`) | Bespoke REST | Library binding |
| Manifest | First class | None | None | First class |
| Snapshot / pause | First class | None | Partial | None |
| Workflow | Out of scope | Out of scope | Out of scope | Out of scope |
| HTTP transport | Optional | Not standardised | Yes | N/A |

SAP's net additions: a **runtime contract** that goes beyond ACP, a **manifest** ACP does not have, and an **HTTP/SSE transport** with explicit reconnection semantics.

---

## Open questions

- **[OQ-1]** HTTP transport: REST+SSE vs. WebSocket vs. both. (§2.2)
- **[OQ-2]** `usage_update` in core or as an observability extension. (§3.2)
- **[OQ-3]** Hierarchical vs. flat capability names. (§4)
- **[OQ-4]** Manifest application via `session/new`, dedicated method, or both. (§7)
- **[OQ-5]** Manifest `output_dirs` field. (§7)
- **[OQ-6]** Standardised reconnect-buffer window. (§10)
- **[OQ-7]** Egress-policy hints in manifest. (§12)

## Change log

- **0.1 (this draft)** — Initial public draft for discussion.
