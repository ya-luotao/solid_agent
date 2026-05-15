# Solid Agent Protocol — Draft

> **Status:** Draft 0.1 — non-normative, for discussion only. Subject to breaking change.
> Open questions are flagged inline with **[OQ-n]** and indexed at the end.

## 1. Scope and goals

The Solid Agent Protocol (SAP) defines the wire contract between four roles:

- **Host** — the application that wants to orchestrate one or more coding-agent runs. Typically a developer tool, IDE, web app, CI job, or autonomous workflow engine.
- **Agent** — a coding-agent backend (Claude Code, Codex, AmpCode, Cursor, Pi, or any other process that takes a prompt and produces a structured response, possibly with tool calls). The Agent's **Reasoning** layer (the LLM) is outside SAP's scope; SAP describes only how the Agent talks.
- **Workspace** — the operating substrate the Agent reads, writes, executes against, and learns from. A Workspace has up to four substrates: filesystem, shell, memory, and audit. See [DESIGN §2–§3](./DESIGN.md#2-why-workspace-is-first-class) for the rationale.
- **Runtime** — the execution boundary that provisions and hosts the Workspace. Local, E2B, Daytona, Modal, Cloudflare Containers, Vercel Sandboxes, Runloop, Blaxel, OpenAI built-in, and so on.

The protocol's job is to make any Host × Agent × Workspace × Runtime combination interoperable as long as each side implements the corresponding role. A given implementation may bundle the Workspace and Runtime into one binary — they remain conceptually distinct in the wire contract.

### 1.1 Non-goals

- SAP is **not** a model serving protocol. It does not specify how an Agent reaches an LLM, what model is used, or how billing works.
- SAP is **not** a workflow engine. It defines per-session primitives that a workflow engine can build on, but DAGs, retries, and human approvals live above SAP.
- SAP is **not** an editor protocol. It does not specify how a UI renders messages or diffs; that is the Host's job.
- SAP does **not** mandate a particular Workspace substrate set. Implementations advertise what they support and Hosts require what they need.

### 1.2 Design principles

1. **Reuse before reinvent.** The Agent↔Host contract is the [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) verbatim where ACP is defined. SAP only extends ACP where ACP is silent — primarily around Workspace substrates beyond plain filesystem and terminal, plus snapshot/pause/resume.
2. **Workspace is first-class.** SAP names filesystem, shell, memory, and audit as distinct substrates of the Workspace. ACP's `fs/*` and `terminal/*` map onto the first two; SAP adds wire methods for the other two and for richer variants of the first two.
3. **Three-axis composability.** Agents speak the ACP "agent side"; Workspaces / Runtimes speak the ACP "client side" plus SAP extensions. A Host is free to pair any compliant Agent with any compliant Workspace+Runtime; capability negotiation handles the cases where they do not match.
4. **Wire-portable manifests.** Workspace seeding is a first-class JSON document, not a host-language object graph.
5. **Capabilities, not levels.** A compliant implementation declares which capability bundles it supports. There is no "SAP 1.0 compliance level"; there is a set of named capabilities a Host can require.
6. **Stable JSON.** All messages are JSON. Field names are `snake_case`. Time fields are RFC 3339. IDs are opaque strings (clients MUST NOT parse them).

## 2. Transport

SAP defines two interchangeable transports.

### 2.1 stdio transport (local, default for in-process spawning)

JSON-RPC 2.0 framed by newline-delimited JSON over stdin/stdout. Identical to ACP's stdio transport. A Host that spawns an Agent or a Workspace/Runtime as a child process MUST use this transport unless it has out-of-band knowledge that the peer supports HTTP.

### 2.2 HTTP transport (remote, used for cloud Runtimes)

A REST + Server-Sent Events (SSE) surface. The wire payloads inside HTTP requests and SSE events are the same JSON-RPC 2.0 envelopes as the stdio transport, with the following routing:

```
POST   /v1/sessions                          # create a session
GET    /v1/sessions/{id}/stream              # SSE — session/update notifications from agent
POST   /v1/sessions/{id}/rpc                 # JSON-RPC requests to the agent
POST   /v1/sessions/{id}/permission/{req_id} # respond to session/request_permission

GET    /v1/sessions/{id}/fs/file?path=…      # workspace — read_text_file
POST   /v1/sessions/{id}/fs/file             # workspace — write_text_file
GET    /v1/sessions/{id}/fs/entries?path=…   # workspace — list  (capability: fs.list)
POST   /v1/sessions/{id}/fs/overlay          # workspace — begin overlay (capability: fs.overlay)
POST   /v1/sessions/{id}/fs/overlay/{oid}/commit
POST   /v1/sessions/{id}/fs/overlay/{oid}/discard

POST   /v1/sessions/{id}/terminals           # workspace — terminal/create
GET    /v1/sessions/{id}/terminals/{tid}/stream  # workspace — terminal/output (SSE)
POST   /v1/sessions/{id}/terminals/{tid}/wait    # workspace — terminal/wait_for_exit
POST   /v1/sessions/{id}/terminals/{tid}/kill    # workspace — terminal/kill
POST   /v1/sessions/{id}/terminals/{tid}/release # workspace — terminal/release

GET    /v1/sessions/{id}/memory/{key}        # workspace — memory_get (capability: memory.kv)
PUT    /v1/sessions/{id}/memory/{key}
DELETE /v1/sessions/{id}/memory/{key}
GET    /v1/sessions/{id}/memory?prefix=…     # memory_list

GET    /v1/sessions/{id}/audit/tool_calls    # workspace — audit query (capability: audit.tool_calls)
GET    /v1/sessions/{id}/audit/fs_changes

GET    /v1/sessions/{id}/mounts              # workspace — list mounts (capability: fs.mount)
POST   /v1/sessions/{id}/mounts              # attach a mount
DELETE /v1/sessions/{id}/mounts/{mid}

POST   /v1/sessions/{id}/snapshot            # runtime/workspace snapshot (capability: snapshot)
POST   /v1/sessions/{id}/pause               # runtime pause
POST   /v1/sessions/{id}/resume              # runtime resume
POST   /v1/sessions/{id}/manifest            # apply manifest mid-session
GET    /v1/sessions/{id}/export?format=…     # export workspace (capability: fs.export)
```

The HTTP surface is intentionally compatible with the namespacing used by [SandboxAgent](https://sandboxagent.dev/). A Host already speaking SandboxAgent's `/v1/acp/{server_id}` should be able to address Solid Agent's `/v1/sessions/{id}` with only a path-prefix change.

**[OQ-1]** Should the HTTP transport be REST+SSE, JSON-RPC over WebSocket, or both? ACP does not pick. SandboxAgent uses REST+SSE; we tentatively do the same.

## 3. Sessions

A **session** is a single coordinated run of one Agent against one Workspace, hosted by one Runtime. A session has:

- A unique opaque `session_id` (string).
- An `agent` selector (e.g., `"claude_code"`, `"codex"`, `"acp_direct"`).
- A `workspace` selector — either implicit (provided by the chosen Runtime) or explicit (e.g., `"agentfs"`, `"mirage"`, composing with the Runtime).
- A `runtime` selector (e.g., `"local"`, `"e2b"`, `"daytona"`).
- An optional `tool_set` reference for function-calling Agents.
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
  "workspace": {
    "kind": "agentfs",
    "options": { "tool_isolation": "per_call", "audit": true }
  },
  "runtime": {
    "kind": "e2b",
    "options": { "template": "ubuntu-22-04", "auto_pause": true }
  },
  "tool_set": "default",
  "manifest": { "...": "see §7" },
  "client_capabilities": { "fs": true, "terminal": true, "diff": true },
  "host_metadata": { "client_name": "my-app", "client_version": "1.2.3" }
}
```

Response:

```json
{
  "session_id": "sess_01HXYZ…",
  "agent_capabilities": { "thinking": true, "structured_output": true },
  "workspace_capabilities": {
    "fs": true, "fs.list": true, "fs.overlay": true, "fs.tool_isolation": true,
    "terminal": true, "memory.kv": true, "audit.tool_calls": true
  },
  "runtime_capabilities": { "snapshot": true, "pause": true, "port_forward": true, "gpu": false },
  "negotiated": { "fs": true, "fs.overlay": true, "terminal": true, "memory.kv": true, "snapshot": true, "pause": true }
}
```

`negotiated` is the intersection of what the Agent needs, what the Workspace offers, what the Runtime offers, and what the Host advertised. If the intersection lacks a required capability, session creation fails with `capability_unsatisfied` (see §9).

If `workspace` is omitted, the Runtime provides its default Workspace shape (typically `fs` + `terminal`, nothing more).

### 3.2 Prompting

After session creation, the Host calls `session/prompt` (ACP). The Agent emits `session/update` notifications. The payload follows ACP's `SessionUpdate` union plus SAP additions:

| Update variant | Source | Purpose |
|---|---|---|
| `agent_message_chunk` | ACP | Streamed assistant text. |
| `agent_thought_chunk` | ACP | Streamed thinking (capability: `thinking`). |
| `user_message_chunk` | ACP | Echo of user input. |
| `tool_call` | ACP | About to invoke a tool. |
| `tool_call_update` | ACP | Progress / completion (status: `pending | in_progress | completed | failed`). |
| `plan` | ACP | Updated plan. |
| `available_commands_update` | ACP | Slash-command list changed. |
| `current_mode_update` | ACP | Agent mode changed. |
| `workspace_status` | SAP | `{ status: "ready" | "lost" | "exported" | "snapshotted" | "overlay_started" | "overlay_committed" | "overlay_discarded" }`. Multiplexed Workspace lifecycle events. |
| `runtime_status` | SAP | `{ status: "provisioning" | "ready" | "paused" | "resumed" | "lost" }`. |
| `usage_update` | SAP | Periodic `{ input_tokens, output_tokens, cache_read_tokens, cost_usd }`. Optional. |

**[OQ-2]** Should `usage_update` be in SAP core or a separate observability extension? Arguments for core: every Host wants it. Arguments against: shape varies wildly between providers.

### 3.3 Permission requests

When an Agent needs the Host's permission to invoke a tool, it sends an ACP `session/request_permission` request. The Host responds with one of `{allow_once, allow_always, reject_once, reject_always, cancelled}`. SAP does not alter this contract.

### 3.4 Cancellation

ACP's `session/cancel` notification. The Agent SHOULD stop streaming as soon as possible and emit a final `session/prompt` response with `stop_reason: cancelled`.

### 3.5 Closing

`DELETE /v1/sessions/{id}` or `solid_agent/session/close`. The Runtime finalises the Workspace (kill, release back to a pool, or persist for later resume). After close, the `session_id` MUST NOT be reused.

## 4. Capability registry

A capability is a string name identifying a discrete behaviour. The initial registry is grouped by which role advertises it.

### 4.1 Agent-side capabilities

| Capability | Meaning |
|---|---|
| `thinking` | Emits `agent_thought_chunk` updates. |
| `structured_output` | Honours a JSON schema in the prompt request. |
| `image_input` | Accepts `image` content blocks in prompts. |
| `tools_external` | Accepts an externally-defined ToolSet (see §10). |
| `session_load` | Can resume a previous session by id. |

### 4.2 Workspace-side capabilities

**Filesystem substrate**

| Capability | Meaning |
|---|---|
| `fs` | Implements `fs/read_text_file` and `fs/write_text_file`. |
| `fs.list` | Directory listing. |
| `fs.watch` | Directory watching. |
| `fs.overlay` | Copy-on-write overlays (`begin_overlay` / `commit_overlay` / `discard_overlay`). |
| `fs.tool_isolation` | Each tool call gets an implicit overlay; auto-commit on success, auto-discard on failure. |
| `fs.snapshot` | Named filesystem checkpoints with restore. |
| `fs.mount` | Pluggable backends mounted as paths. Each backend kind is a sub-capability (e.g., `mount.s3`, `mount.gdocs`). |
| `fs.export` | Portable workspace export. |

**Shell substrate**

| Capability | Meaning |
|---|---|
| `terminal` | Implements `terminal/create`, `terminal/output`, `terminal/wait_for_exit`, `terminal/kill`, `terminal/release`. |
| `terminal.pty` | Terminals are full PTYs (resize, raw mode). |
| `exec.in_process` | The shell runs in the host's address space (no subprocess). |
| `exec.languages.python` | In-process Python execution. |
| `exec.languages.javascript` | In-process JavaScript execution. |
| `exec.languages.sqlite` | In-process SQLite execution. |
| `exec.network` | Outbound network from the shell is allowed. |
| `exec.limits` | The Workspace enforces structured execution limits. |

**Memory substrate**

| Capability | Meaning |
|---|---|
| `memory.kv` | Key-value memory (`memory_get/set/delete/list`). |
| `memory.namespace` | Logical namespaces. |
| `memory.ttl` | TTL on entries. |
| `memory.export` | Memory export. |

**Audit substrate**

| Capability | Meaning |
|---|---|
| `audit.tool_calls` | Append-only tool-call log. |
| `audit.fs_changes` | Append-only filesystem-change log per tool call. |
| `audit.query` | Structured query API over audit records. |

### 4.3 Runtime-side capabilities

| Capability | Meaning |
|---|---|
| `snapshot` | Runtime-level snapshot of the entire Workspace + processes. |
| `pause` | Pause / resume the underlying compute. |
| `port_forward` | Expose a TCP port reachable from the public internet. |
| `gpu` | At least one GPU available. |
| `region` | Region selection at provision time. |

### 4.4 Host-side capabilities

| Capability | Meaning |
|---|---|
| `diff` | Renders `diff` content in `tool_call_update`. |
| `terminal_view` | Renders live terminal output. |
| `permission_ui` | Surfaces `request_permission` interactively. |

Hosts MAY require capabilities at session creation. Workspaces, Runtimes, and Agents SHOULD advertise every capability they support, even if unused.

**[OQ-3]** Should capability names be hierarchical (`fs.list`) or flat (`fs_list`)? Hierarchical reads better but requires a parser. SandboxAgent uses flat.

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

The Agent calls into the Workspace for filesystem, shell, memory, and audit operations. The Host routes those calls to the Workspace bound at session creation.

## 6. Workspace side — required and optional methods

A Workspace implementation MUST implement at least one substrate.

### 6.1 Filesystem substrate (capability `fs`)

| Method | Source |
|---|---|
| `fs/read_text_file` | ACP |
| `fs/write_text_file` | ACP |
| `fs/list` (cap. `fs.list`) | SAP |
| `fs/watch` (cap. `fs.watch`) | SAP |
| `fs/begin_overlay` (cap. `fs.overlay`) | SAP |
| `fs/commit_overlay` | SAP |
| `fs/discard_overlay` | SAP |
| `fs/snapshot` (cap. `fs.snapshot`) | SAP |
| `fs/restore_snapshot` | SAP |
| `fs/mount` (cap. `fs.mount`) | SAP |
| `fs/unmount` | SAP |
| `fs/export` (cap. `fs.export`) | SAP |

#### `fs/begin_overlay`

```json
// request
{ "label": "tool-call-edit-1" }
// response
{ "overlay_id": "ovl_01HXYZ…", "created_at": "2026-05-15T10:00:00Z" }
```

Reads see overlay-or-base; writes go into the overlay. `commit_overlay` merges into the base; `discard_overlay` drops the overlay.

#### `fs/snapshot`

```json
// request
{ "label": "post-test-pass" }
// response
{ "snapshot_id": "snap_01HXYZ…", "created_at": "2026-05-15T10:00:00Z", "size_bytes": 1234567 }
```

#### `fs/mount`

```json
// request
{ "kind": "s3", "options": { "bucket": "my-bucket", "prefix": "data/", "mode": "ro" }, "target": "/s3/data" }
// response
{ "mount_id": "mnt_01HXYZ…", "target": "/s3/data" }
```

A Workspace advertises each backend kind it supports as a separate `mount.*` capability so Hosts can require exactly what they need.

### 6.2 Shell substrate (capability `terminal`)

| Method | Source |
|---|---|
| `terminal/create` | ACP |
| `terminal/output` | ACP |
| `terminal/wait_for_exit` | ACP |
| `terminal/kill` | ACP |
| `terminal/release` | ACP |
| `terminal/resize` (cap. `terminal.pty`) | SAP |

When the Workspace advertises `exec.in_process`, terminals run in the host process's address space; lifetime, signals, and isolation guarantees differ from subprocess terminals. The capability tells the Host what to expect.

### 6.3 Memory substrate (capability `memory.kv`)

| Method | Description |
|---|---|
| `memory/get` | `{ key, namespace? }` → `{ value, found, ttl? }` |
| `memory/set` | `{ key, value, namespace?, ttl? }` → `{ replaced }` |
| `memory/delete` | `{ key, namespace? }` → `{ deleted }` |
| `memory/list` | `{ prefix?, namespace? }` → `{ keys: [...] }` |
| `memory/export` (cap. `memory.export`) | `{}` → stream of namespaced entries |

### 6.4 Audit substrate (capability `audit.tool_calls`)

| Method | Description |
|---|---|
| `audit/query_tool_calls` | `{ since?, until?, tool_name?, status?, limit?, cursor? }` → `{ records: [...], cursor? }` |
| `audit/query_fs_changes` (cap. `audit.fs_changes`) | `{ since?, path_prefix?, tool_call_id? }` → `{ records: [...], cursor? }` |

Audit records have stable shapes:

```json
{
  "tool_call_id": "tc_01H…",
  "name": "Bash",
  "args_digest": "sha256:…",
  "started_at": "2026-05-15T10:00:00Z",
  "finished_at": "2026-05-15T10:00:02Z",
  "status": "completed",
  "result_digest": "sha256:…",
  "fs_changes_count": 3
}
```

Args and results are not stored inline; SAP stores digests and lets the Workspace expose blobs through a separate retrieval method if it wants to (`audit/fetch_blob(digest)` — capability `audit.blobs`).

## 7. Runtime side — required and optional methods

A Runtime is a Workspace **provisioner**. Beyond hosting a Workspace, it implements lifecycle and infrastructure methods.

| Method | Capability |
|---|---|
| `runtime/provision` | (always) |
| `runtime/finalize` | (always) |
| `runtime/pause` | `pause` |
| `runtime/resume` | `pause` |
| `runtime/snapshot` | `snapshot` |
| `runtime/host_for_port` | `port_forward` |
| `runtime/manifest_apply` | (always when manifest support advertised) |

A `runtime/snapshot` captures the entire Workspace + process state at the Runtime level (typically a microVM snapshot). It is coarser than `fs/snapshot` (which captures only filesystem state) and depends on Runtime support.

## 8. Manifest

A manifest describes the Workspace state a session should start with. JSON document; Hosts may load from YAML.

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
  "mounts": [
    { "kind": "s3", "options": { "bucket": "data", "prefix": "logs/", "mode": "ro" }, "target": "/s3/logs" }
  ],
  "memory": {
    "seeds": [
      { "namespace": "facts", "key": "primary_language", "value": "ruby" }
    ]
  },
  "skills":   { "source": "bundled", "dirs": ["skills/"] },
  "capabilities_required": ["fs", "terminal", "memory.kv"]
}
```

Notes:

- `files.source.kind` MAY be `inline`, `url`, `local_path` (Host-resolved), or `secret`.
- `env` values may be plain strings or `{ source: "secret", ref: "<name>" }`. Secret resolution is out of scope.
- `mounts` is generic; the Workspace advertises which backend kinds it supports.
- `memory.seeds` populates the Memory substrate at provision time if it is available.
- `capabilities_required` is a forward-compatible hint; session creation will fail capability negotiation if any are missing.

**[OQ-4]** Should the manifest be applied as part of `session/new`, as a separate `runtime/manifest_apply` call, or both? Proposed: both.

**[OQ-5]** Should the manifest define `output_dirs` to indicate where session artefacts live? Useful for archiving sessions.

## 9. Errors

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
| `-32008` | `workspace_corrupt` | The Workspace's substrate is in an inconsistent state and cannot be recovered. |
| `-32009` | `overlay_conflict` | An overlay commit conflicts with concurrent base changes. |
| `-32099` | `internal` | Catch-all. The `data` field SHOULD include a human-readable message. |

## 10. ToolSets (informative)

ToolSets are a Host-side concept; they do not travel on the wire between Agent and Workspace. A ToolSet is a registry of named tool definitions (e.g., `bash`, `read_file`, `write_file`, `edit`, `search`, `memory_get`). Each tool has a JSON schema (the function-calling signature) and an executor that routes the call into Workspace methods.

For ACP-native Agents, the Agent adapter passes the ToolSet through as ACP `tool_call` definitions. For function-calling Agents, the Agent adapter renders the ToolSet into the backend's tool format (Anthropic tools, OpenAI tools, etc.) and translates tool invocations into Workspace method calls.

This section is informative: a Host that wants to wire each Agent to the Workspace by hand is free to do so. See [DESIGN §6](./DESIGN.md#6-tools--the-bridge-for-non-acp-agents) for the architectural framing.

## 11. Identifiers and time

- All IDs are opaque strings. ULID-like or UUIDv7 recommended. Clients MUST NOT parse them or assume ordering.
- All timestamps are RFC 3339 strings with millisecond precision and a `Z` or numeric offset.

## 12. Reconnection

Either party MAY disconnect at any time.

- **Stdio** — no reconnection unless the Agent supports `session/load`.
- **HTTP** — the Host MAY reopen the SSE `stream` endpoint with a `Last-Event-ID` header to resume from the last delivered event. Implementations SHOULD buffer `session/update` events for at least 60 seconds.

**[OQ-6]** Standardised reconnect-buffer window. Implementations in the wild use anywhere from 30 seconds to several minutes.

## 13. Versioning

Single integer version field on `initialize`. Current draft: `protocol_version: 1`. Breaking changes bump; additive changes do not. Implementations MUST reject sessions whose `protocol_version` is higher than the highest they understand.

## 14. Security model — outline

Full security model is out of scope for this draft; the boundaries are:

- The Host is **trusted** by definition.
- The Agent is **partially trusted** — it can read and write within the Workspace but cannot escape the Runtime.
- The Workspace is **untrusted with respect to Host secrets** — secrets reach the Workspace only via explicit manifest `env.source: "secret"` references.
- The Runtime is the **isolation boundary**. The Workspace inherits the Runtime's threat model.
- Tool calls that require permission MUST be gated by `session/request_permission`; a Host MAY install an auto-allow policy, but the protocol still requires the request to be visible.

**[OQ-7]** Should SAP specify a sandbox-egress policy hint in the manifest? Useful for high-security Hosts but couples the spec to Runtime implementations that support egress filtering.

## 15. Conformance profiles

Named profiles let an implementation declare a starting point. Hosts compose these by capability requirements; the profiles are conveniences.

- **Minimal Agent** — `initialize`, `session/new`, `session/prompt`, `session/cancel`, `session/update`.
- **Minimal Workspace** — `fs` and `terminal` capabilities.
- **Auditable Workspace** — Minimal Workspace plus `audit.tool_calls`.
- **Stateful Workspace** — Minimal Workspace plus `memory.kv` and `fs.overlay`.
- **Portable Workspace** — Stateful Workspace plus `fs.export`.
- **Minimal Runtime** — provision/finalize only.
- **Persistent Runtime** — Minimal Runtime plus `pause` and `snapshot`.

## 16. Comparison with adjacent specs

| Concern | Solid Agent | ACP | SandboxAgent | OpenAI Sandbox |
|---|---|---|---|---|
| Agent↔Host wire | ACP (re-used) | Native | ACP-based | SDK-only |
| Workspace as first-class | Yes | No (just `fs/terminal`) | No | Partial (manifest only) |
| Runtime↔Host wire | ACP client side + SAP extensions | Partial | Bespoke REST | Library binding |
| FS overlays / tool isolation | First class (`fs.overlay`) | None | None | None |
| Memory substrate | First class (`memory.kv`) | None | None | None |
| Audit substrate | First class (`audit.*`) | None | None | None |
| Mounted heterogeneous FS | First class (`fs.mount`) | None | None | None |
| Manifest | First class | None | None | First class |
| Snapshot / pause | First class | None | Partial | None |
| HTTP transport | Optional | Not standardised | Yes | N/A |

SAP's net additions: a **first-class Workspace** with overlay/memory/audit/mount substrates, a **manifest** ACP does not have, and an **HTTP/SSE transport** with explicit reconnection semantics.

---

## Open questions

- **[OQ-1]** HTTP transport: REST+SSE vs. WebSocket vs. both. (§2.2)
- **[OQ-2]** `usage_update` in core or as an observability extension. (§3.2)
- **[OQ-3]** Hierarchical vs. flat capability names. (§4)
- **[OQ-4]** Manifest application via `session/new`, dedicated method, or both. (§8)
- **[OQ-5]** Manifest `output_dirs` field. (§8)
- **[OQ-6]** Standardised reconnect-buffer window. (§12)
- **[OQ-7]** Egress-policy hints in manifest. (§14)
- **[OQ-8]** Should the Workspace be addressable independently of the Runtime (separate `workspace_id`), or always anonymous-inside-session?
- **[OQ-9]** Should `fs.tool_isolation` overlays be triggered explicitly by the Host, implicitly by the Workspace, or by an Agent-level signal?
- **[OQ-10]** Should `memory.kv` define a consistency model (strong, eventual, session-scoped) or leave it implementation-defined?

## Change log

- **0.1 (this draft)** — Initial public draft for discussion. Introduces Workspace as a first-class concern with filesystem, shell, memory, and audit substrates.
