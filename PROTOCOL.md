# Solid Agent Protocol — Draft

> **Status:** Draft 0.1 — non-normative, for discussion only. Subject to breaking change.
> **Scope:** v0.1 covers the **Agent ↔ Runtime** contract only. Richer Workspace substrates (overlay filesystems, mounted heterogeneous backends, in-process shells, memory and audit stores) are explicitly out of scope for v0.1 and parked in [FUTURE.md](./FUTURE.md).
> Open questions are flagged inline with **[OQ-n]** and indexed at the end.

## 1. Scope and goals

The Solid Agent Protocol (SAP) defines the wire contract between three roles:

- **Host** — the application orchestrating a coding-agent run. Typically a developer tool, IDE, web app, CI job, or autonomous workflow engine.
- **Agent** — a coding-agent backend (Claude Code, Codex, AmpCode, Cursor, Pi, or any other process that takes a prompt and produces a structured response). The Agent's reasoning layer (the LLM, prompt, planner) is outside SAP's scope; SAP describes only how the Agent talks.
- **Runtime** — the execution boundary that provisions and hosts the working directory the Agent operates in. Local, E2B, Daytona, Modal, Cloudflare Containers, Vercel Sandboxes, Runloop, Blaxel, OpenAI built-in, and so on.

The protocol's job is to make any Host × Agent × Runtime combination interoperable as long as each side implements its role. Composition is **not** orthogonal — Agents and Runtimes are intentionally coupled through capability negotiation (see §4) — but the coupling is explicit, declared, and version-stable.

### 1.1 Non-goals

- SAP is **not** a model serving protocol. It does not specify how an Agent reaches an LLM.
- SAP is **not** a workflow engine. Per-session primitives are defined; orchestration is left to Hosts.
- SAP is **not** an editor protocol. It says nothing about rendering.
- SAP v0.1 **does not** specify rich Workspace substrates (filesystem overlays, mounted heterogeneous backends, in-process shells, memory or audit stores). Those ideas live in [FUTURE.md](./FUTURE.md) and may return as RFCs once v0.1 has a reference implementation and usage data.

### 1.2 Design principles

1. **Reuse before reinvent.** The Agent↔Host contract is the [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) verbatim where ACP is defined. SAP only extends ACP where ACP is silent — primarily around Runtime lifecycle (snapshot, pause, resume) and an HTTP/SSE transport.
2. **Composable, capability-negotiated.** Agents and Runtimes declare what they support; sessions are accepted or rejected up front based on the intersection. No "compliance level."
3. **Wire-portable manifests.** Workspace seeding is a first-class JSON document, not a host-language object graph.
4. **Stable JSON.** All messages are JSON. Field names are `snake_case`. Time fields are RFC 3339. IDs are opaque strings (clients MUST NOT parse them).

## 2. Transport

SAP defines two interchangeable transports.

### 2.1 stdio transport

JSON-RPC 2.0 framed by newline-delimited JSON over stdin/stdout. Identical to ACP's stdio transport. A Host that spawns an Agent or Runtime as a child process MUST use this transport unless it has out-of-band knowledge that the peer supports HTTP.

### 2.2 HTTP transport

REST + Server-Sent Events (SSE). The wire payloads inside HTTP requests and SSE events are the same JSON-RPC 2.0 envelopes as the stdio transport.

```
POST   /v1/sessions                          # create a session
GET    /v1/sessions/{id}/stream              # SSE — session/update notifications
POST   /v1/sessions/{id}/rpc                 # JSON-RPC requests to the agent
POST   /v1/sessions/{id}/permission/{req_id} # respond to session/request_permission
DELETE /v1/sessions/{id}                     # close the session

GET    /v1/sessions/{id}/fs/file?path=…      # runtime — read_text_file
POST   /v1/sessions/{id}/fs/file             # runtime — write_text_file
GET    /v1/sessions/{id}/fs/entries?path=…   # runtime — list (capability: fs.list)

POST   /v1/sessions/{id}/terminals           # runtime — terminal/create
GET    /v1/sessions/{id}/terminals/{tid}/stream  # runtime — terminal/output (SSE)
POST   /v1/sessions/{id}/terminals/{tid}/wait    # runtime — terminal/wait_for_exit
POST   /v1/sessions/{id}/terminals/{tid}/kill    # runtime — terminal/kill
POST   /v1/sessions/{id}/terminals/{tid}/release # runtime — terminal/release

POST   /v1/sessions/{id}/snapshot            # runtime/snapshot (capability: snapshot)
POST   /v1/sessions/{id}/pause               # runtime/pause (capability: pause)
POST   /v1/sessions/{id}/resume              # runtime/resume (capability: pause)
POST   /v1/sessions/{id}/manifest            # apply manifest mid-session
```

The HTTP surface is intentionally compatible with the namespacing used by [SandboxAgent](https://sandboxagent.dev/).

**[OQ-1]** Should the HTTP transport be REST+SSE, JSON-RPC over WebSocket, or both? SandboxAgent uses REST+SSE; we tentatively do the same.

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
  "runtime_capabilities": { "fs": true, "fs.list": true, "terminal": true, "snapshot": true, "pause": true, "port_forward": true },
  "negotiated": { "fs": true, "terminal": true, "diff": true, "snapshot": true, "pause": true }
}
```

`negotiated` is the intersection of what the Agent requires, what the Runtime offers, and what the Host advertised. A missing required capability fails creation with `capability_unsatisfied` (see §8).

### 3.2 Prompting

After session creation, the Host calls ACP's `session/prompt`. The Agent emits `session/update` notifications. The payload follows ACP's `SessionUpdate` union plus a small set of SAP additions:

| Update variant | Source | Purpose |
|---|---|---|
| `agent_message_chunk` | ACP | Streamed assistant text. |
| `agent_thought_chunk` | ACP | Streamed thinking (capability: `thinking`). |
| `user_message_chunk` | ACP | Echo of user input. |
| `tool_call` | ACP | About to invoke a tool. |
| `tool_call_update` | ACP | Progress / completion (`pending | in_progress | completed | failed`). |
| `plan` | ACP | Updated plan. |
| `available_commands_update` | ACP | Slash-command list changed. |
| `current_mode_update` | ACP | Agent mode changed. |
| `runtime_status` | SAP | `{ status: "provisioning" | "ready" | "paused" | "resumed" | "lost" }`. |
| `usage_update` | SAP | Periodic `{ input_tokens, output_tokens, cache_read_tokens, cost_usd }`. Optional. |

**[OQ-2]** Should `usage_update` be in SAP core or a separate observability extension?

### 3.3 Permission requests

When an Agent needs the Host's permission to invoke a tool, it sends an ACP `session/request_permission` request. The Host responds with one of `{allow_once, allow_always, reject_once, reject_always, cancelled}`. SAP does not alter this contract.

### 3.4 Cancellation

ACP's `session/cancel` notification. The Agent SHOULD stop streaming and emit a final `session/prompt` response with `stop_reason: cancelled`.

### 3.5 Closing

`DELETE /v1/sessions/{id}` or `solid_agent/session/close`. The Runtime finalises (kill or release back to a pool). After close, the `session_id` MUST NOT be reused.

## 4. Capability registry

Capabilities are string names identifying discrete behaviours. The v0.1 registry is intentionally small.

### 4.1 Agent-side capabilities

| Capability | Meaning |
|---|---|
| `thinking` | Emits `agent_thought_chunk` updates. |
| `structured_output` | Honours a JSON schema in the prompt request. |
| `image_input` | Accepts `image` content blocks in prompts. |
| `tools_external` | Accepts an externally-defined tool list. |
| `session_load` | Can resume a previous session by id. |
| `interrupt` | Honours `session/cancel`. |

### 4.2 Runtime-side capabilities

| Capability | Meaning |
|---|---|
| `fs` | Implements `fs/read_text_file` and `fs/write_text_file`. |
| `fs.list` | Directory listing. |
| `fs.watch` | Directory watching. |
| `terminal` | Implements `terminal/create`, `terminal/output`, `terminal/wait_for_exit`, `terminal/kill`, `terminal/release`. |
| `terminal.pty` | Full PTY (resize, raw mode). |
| `snapshot` | Runtime-level snapshot of the entire workspace + processes. |
| `pause` | Pause / resume the underlying compute. |
| `port_forward` | Expose a TCP port reachable from the public internet. |
| `gpu` | At least one GPU available. |
| `region` | Region selection at provision time. |

### 4.3 Host-side capabilities

| Capability | Meaning |
|---|---|
| `diff` | Renders `diff` content in `tool_call_update`. |
| `terminal_view` | Renders live terminal output. |
| `permission_ui` | Surfaces `request_permission` interactively. |

Richer capabilities (filesystem overlays, mounted heterogeneous backends, in-process shells, memory or audit stores) are out of scope for v0.1. See [FUTURE.md](./FUTURE.md).

**[OQ-3]** Should capability names be hierarchical (`fs.list`) or flat (`fs_list`)? SandboxAgent uses flat. Hierarchical reads better but requires a parser.

## 5. Agent side — required methods

An Agent implementation MUST implement:

| Method | Direction | Source |
|---|---|---|
| `initialize` | Host → Agent | ACP |
| `session/new` | Host → Agent | ACP |
| `session/prompt` | Host → Agent | ACP |
| `session/cancel` | Host → Agent (notification) | ACP |
| `session/update` | Agent → Host (notification) | ACP |
| `session/request_permission` | Agent → Host | ACP |

An Agent MAY also implement: `authenticate`, `session/load`, `session/set_mode` (all ACP).

The Agent calls into the Runtime for filesystem and terminal operations. The Host routes those calls to the Runtime bound at session creation.

## 6. Runtime side — required methods

A Runtime implementation MUST implement at least the `fs` and `terminal` capabilities.

### 6.1 Filesystem (capability `fs`)

| Method | Source |
|---|---|
| `fs/read_text_file` | ACP |
| `fs/write_text_file` | ACP |
| `fs/list` (cap. `fs.list`) | SAP |
| `fs/watch` (cap. `fs.watch`) | SAP |

These are deliberately minimal. ACP's `fs/read_text_file` and `fs/write_text_file` are the baseline for any agent to do useful work.

### 6.2 Terminal (capability `terminal`)

| Method | Source |
|---|---|
| `terminal/create` | ACP |
| `terminal/output` | ACP |
| `terminal/wait_for_exit` | ACP |
| `terminal/kill` | ACP |
| `terminal/release` | ACP |
| `terminal/resize` (cap. `terminal.pty`) | SAP |

### 6.3 Lifecycle (always required)

| Method | Capability |
|---|---|
| `runtime/provision` | (always) |
| `runtime/finalize` | (always) |
| `runtime/manifest_apply` | (always when manifest support advertised) |
| `runtime/pause` | `pause` |
| `runtime/resume` | `pause` |
| `runtime/snapshot` | `snapshot` |
| `runtime/host_for_port` | `port_forward` |

A `runtime/snapshot` captures the entire workspace + process state at the Runtime level — typically a microVM snapshot. The resulting `snapshot_id` may be passed as `runtime.options.snapshot_id` at a later `session/new` call to bootstrap from the snapshot's state.

```json
// runtime/snapshot request
{ "label": "post-test-pass" }
// response
{ "snapshot_id": "snap_01HXYZ…", "created_at": "2026-05-15T10:00:00Z", "size_bytes": 1234567 }
```

## 7. Manifest

A manifest declares the initial workspace state a session should start with. JSON document; Hosts may load it from YAML.

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

Notes:

- `files.source.kind` MAY be `inline`, `url`, `local_path` (Host-resolved), or `secret`.
- `env` values may be plain strings or `{ source: "secret", ref: "<name>" }`. Secret resolution is out of scope.
- `capabilities_required` is a forward-compatible hint; session creation fails capability negotiation if any are missing.

Manifests MAY be applied at session creation or re-applied mid-session via `runtime/manifest_apply`.

**[OQ-4]** Manifest fields for richer use cases (mounts, skills, memory seeds) are deferred to [FUTURE.md](./FUTURE.md). The current shape is intentionally minimal.

## 8. Errors

JSON-RPC errors use the standard envelope. SAP defines the following `code` values:

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
| `-32099` | `internal` | Catch-all. The `data` field SHOULD include a human-readable message. |

## 9. Identifiers and time

- All IDs are opaque strings. ULID-like or UUIDv7 recommended. Clients MUST NOT parse them or assume ordering.
- All timestamps are RFC 3339 strings with millisecond precision and a `Z` or numeric offset.

## 10. Reconnection

- **Stdio** — no reconnection unless the Agent supports `session/load`.
- **HTTP** — the Host MAY reopen the SSE `stream` endpoint with a `Last-Event-ID` header to resume from the last delivered event. Implementations SHOULD buffer `session/update` events for at least 60 seconds.

**[OQ-5]** Standardised reconnect-buffer window. Implementations in the wild use anywhere from 30 seconds to several minutes.

## 11. Versioning

Single integer version field on `initialize`. Current draft: `protocol_version: 1`. Breaking changes bump; additive changes do not. Implementations MUST reject sessions whose `protocol_version` is higher than the highest they understand.

## 12. Security model — outline

The full security model is out of scope for this draft. The boundaries are:

- The Host is **trusted** by definition.
- The Agent is **partially trusted** — it can read and write within the Runtime but cannot escape it.
- The Runtime is the **isolation boundary**. Workspace contents inherit the Runtime's threat model.
- Tool calls that require permission MUST be gated by `session/request_permission`; a Host MAY install an auto-allow policy, but the protocol still requires the request to be visible.

**[OQ-6]** Should SAP specify a sandbox-egress policy hint in the manifest? Useful but couples to specific Runtime implementations.

## 13. Conformance profiles

Named profiles let an implementation declare a starting point. Hosts compose by capability requirements; profiles are conveniences.

- **Minimal Agent** — `initialize`, `session/new`, `session/prompt`, `session/cancel`, `session/update`.
- **Minimal Runtime** — `fs` and `terminal` capabilities.
- **Persistent Runtime** — Minimal Runtime plus `pause` and `snapshot`.

## 14. What's deferred

[FUTURE.md](./FUTURE.md) records ideas intentionally out of scope for v0.1:

- Filesystem overlays, snapshots-at-file-level, tool-call isolation, export.
- Mounted heterogeneous backends (S3, Drive, Slack, GitHub, …) as filesystem paths.
- In-process shell substrates with execution limits and pluggable interpreters.
- Memory and audit stores as Runtime-served substrates.
- A host-side Tools layer for bridging function-calling agents to ACP-shaped Runtimes.
- `session/steer` as a wire method for mid-turn input injection.
- Approval-guardian as a structured capability.

Each item has a sketch of what an RFC would add and why deferral was the right call for v0.1.

## 15. Comparison with adjacent specs

| Concern | Solid Agent v0.1 | ACP | SandboxAgent | OpenAI Sandbox |
|---|---|---|---|---|
| Agent↔Host wire | ACP (re-used) | Native | ACP-based | SDK-only |
| Runtime↔Host wire | ACP client side + SAP lifecycle | Partial | Bespoke REST | Library binding |
| Manifest | First class | None | None | First class |
| Runtime snapshot / pause / resume | First class | None | Partial | None |
| HTTP transport | Optional | Not standardised | Yes | N/A |
| Workspace substrate (overlay/mount/memory/audit) | Deferred to FUTURE | None | None | None |

SAP v0.1's net additions over ACP: a **Runtime lifecycle contract** (provision, finalize, pause, resume, snapshot, port_forward, manifest_apply), a **manifest** ACP does not have, and an **HTTP/SSE transport** with explicit reconnection semantics.

---

## Open questions

- **[OQ-1]** HTTP transport: REST+SSE vs. WebSocket vs. both. (§2.2)
- **[OQ-2]** `usage_update` in core or as an observability extension. (§3.2)
- **[OQ-3]** Hierarchical vs. flat capability names. (§4)
- **[OQ-4]** Manifest fields beyond v0.1's minimum (mounts, skills, memory). Tracking in [FUTURE.md](./FUTURE.md). (§7)
- **[OQ-5]** Standardised reconnect-buffer window. (§10)
- **[OQ-6]** Egress-policy hints in manifest. (§12)

## Change log

- **0.1 (this draft)** — Initial public draft. Scope: Agent SDK wrap + Runtime support. Workspace substrate concerns parked in [FUTURE.md](./FUTURE.md).
