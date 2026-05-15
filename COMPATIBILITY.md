# Agent Compatibility Matrix

> **Status:** Draft 0.1 — open for correction. Vendor surfaces change quickly; please file issues or PRs when you spot drift.
> **Last verified:** May 2026.

This document is a developer reference. It enumerates the [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) capability surface as the canonical baseline, then shows whether each major coding-agent SDK has a native, partial, or no equivalent.

The matrix exists to answer two practical questions:

1. **"If I write an `Agents::X` adapter, what does the adapter have to translate vs. reuse?"**
2. **"If I want to use feature Y with agent Z today, is it possible — and if so, through what path?"**

## Legend

| Symbol | Meaning |
|:-:|---|
| **✓** | Native equivalent. The agent SDK exposes this directly. |
| **◐** | Partial / non-standard equivalent. Available but with caveats — usually a different shape, a subset, or non-ergonomic. |
| **⌬** | Available only through a third-party bridge or community shim. Not first-party. |
| **✗** | No equivalent. A Solid Agent adapter would have to synthesise this (or declare it unsupported via capability negotiation). |
| **?** | Undocumented in publicly available sources at the time of writing. Help wanted. |

## Agents covered

| Agent | Identifier | Source | License | Wire format |
|---|---|---|---|---|
| **Claude Code** | `claude_code` | [`@anthropic-ai/claude-agent-sdk`](https://github.com/anthropics/claude-agent-sdk-python), [`claude-agent-sdk-ruby`](https://github.com/ya-luotao/claude-agent-sdk-ruby) | MIT (SDK), proprietary CLI | newline-delimited JSON over stdio (stream-JSON) |
| **Codex** | `codex` | [`codex-rb`](https://github.com/openai/codex-rb), `openai-codex` | Apache-2.0 (SDK), proprietary CLI | JSON-RPC 2.0 over stdio |
| **Pi** | `pi` | [`@earendil-works/pi-coding-agent`](https://github.com/badlogic/pi-mono) | MIT | LF-delimited JSONL over stdio (custom 3-message protocol) |
| **AmpCode** | `ampcode` | [`@ampcode/cli`](https://www.npmjs.com/package/@ampcode/cli) by Sourcegraph | Closed source (minified JS bundle) | "Claude Code compatible" stream-JSON over stdio |
| **Cursor** | `cursor` | [`@cursor/sdk`](https://cursor.com/changelog/sdk-release) | Closed source | HTTP + SSE (Cloud Agents API) |
| **ACPDirect** | `acp_direct` | Solid Agent reference adapter | MIT | Native ACP (JSON-RPC 2.0 over stdio) |

`ACPDirect` is included as a baseline — any agent that ships native ACP support gets row-level "✓" across the matrix automatically. As of May 2026, this column is populated only by ACP reference implementations and a handful of editor-side agents; no mainstream commercial coding-agent ships native ACP yet.

---

## 1. Wire protocol & transport

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| JSON-RPC 2.0 framing | ✓ | ✗ (custom stream-JSON) | ✓ | ✗ (custom 3-msg protocol) | ✗ (custom stream-JSON) | ✗ (HTTP+SSE) |
| Stdio transport | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| HTTP / SSE transport | ◐ (under proposal) | ◐ (via custom transport class) | ✗ | ✗ | ✗ | ✓ |
| WebSocket transport | ◐ (under proposal) | ✗ | ✗ | ✗ | ✗ | ✗ |
| Pluggable transport hook (host swaps the transport class) | n/a | ✓ (`Client.new(transport_class: …)`) | ◐ (low-level `AppServerClient`) | ✗ | ✗ | ✗ |
| Newline-delimited JSON | ✓ | ✓ | ✓ | ✓ (strict LF) | ✓ | n/a |

**Implication for adapters.** Claude Code, Codex, Pi, and AmpCode are all stdio-newline-JSON families that fit cleanly inside the existing SDK Transport pattern. Cursor's HTTP+SSE surface requires a different bridge — the Solid Agent adapter would forward Cursor's SSE stream over the Workspace/Host JSON-RPC boundary. Pi's custom 3-message protocol does not map 1:1 to JSON-RPC's request/response/notification trichotomy; the adapter has to maintain a request-ID correlation table.

## 2. Lifecycle methods

| ACP method | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| `initialize` | ✗ (init handshake via first message) | ◐ (initialize at thread_start) | ◐ (config in mode flag) | ✗ (subprocess args) | ◐ (Agent.create) |
| `authenticate` | ◐ (env vars, login subcommand) | ◐ (env vars) | ◐ (env vars) | ◐ (env vars, login subcommand) | ✓ (`apiKey`) |
| `session/new` | ◐ (`Client.connect` / `query`) | ✓ (`thread_start`) | ✓ (RPC `new_session`) | ✓ (`amp threads new`) | ✓ (`Agent.create`) |
| `session/load` | ✓ (resume support) | ✓ (`thread_resume`) | ✓ (`switch_session`, `--session`) | ✓ (`amp threads continue`) | ✓ (cloud runs persist) |
| `session/resume` | ✓ | ✓ | ✓ | ✓ | ✓ (SSE reconnect) |
| `session/prompt` | ✓ (`query`) | ✓ (`thread.run`) | ✓ (RPC `prompt`) | ✓ (stdin pipe) | ✓ (`.send` / `.stream`) |
| `session/cancel` | ✓ (`interrupt`) | ✓ (`turn.interrupt`) | ✓ (RPC `abort`) | ◐ (Ctrl+C only) | ◐ (lifecycle API; surface ?) |
| `session/close` | ✓ (`disconnect`) | ✓ | ✓ | ✓ | ✓ |
| `session/list` | ◐ (browse subcommand) | ◐ (resume picker) | ✓ (`-r` browse) | ✓ (`amp threads list`) | ✓ (Agents window) |
| `session/set_mode` | ◐ (permission_mode option) | ◐ (approval_mode option) | ✗ | ◐ (config file) | ✗ |
| `session/set_config_option` | ◐ (options dict) | ◐ (config_overrides) | ◐ (config file) | ◐ (config file) | ◐ (Agent.create args) |

**Implication for adapters.** Lifecycle is the area of strongest convergence. Every agent supports new/load/prompt/cancel in some form; Solid Agent's `Agents::*` adapters mostly translate vocabulary. `session/set_mode` is where ACP is ahead of every vendor surveyed — only ACP-native agents have a clean wire concept for runtime mode switching (`ask` ↔ `architect` ↔ `code`).

## 3. SessionUpdate variants (what the agent emits)

| ACP `sessionUpdate` variant | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| `agent_message_chunk` | ✓ (`TextBlock` in `AssistantMessage`) | ✓ (`AgentMessageDelta`) | ✓ (`text_delta` event) | ✓ (stream-JSON `message`) | ✓ (SSE `message` event) |
| `agent_thought_chunk` | ✓ (`ThinkingBlock`) | ◐ (in `Item` payload) | ✓ (`thinking_delta` event) | ? | ? |
| `user_message_chunk` | ◐ (`UserMessage` echo) | ◐ (`Item` payload) | ✓ (`message_update` event) | ? | ? |
| `tool_call` | ✓ (`ToolUseBlock`) | ✓ (`ItemStarted` of tool kind) | ✓ (`tool_execution_start`) | ✓ (stream-JSON `tool_use`) | ✓ (SSE `tool_call` event) |
| `tool_call_update` | ✓ (`ToolProgressMessage` + `UserMessage.tool_use_result`) | ✓ (`ItemCompleted`, `McpToolCallProgress`) | ✓ (`tool_execution_update/end`) | ✓ (stream-JSON `tool_result`) | ◐ (in `tool_call` payload) |
| `plan` | ✓ (via TodoWrite tool output) | ◐ (in agent message) | ◐ (custom plan tool) | ◐ (custom plan tool) | ◐ (subagent payload) |
| `available_commands_update` | ✓ (`PromptSuggestionMessage`) | ✗ | ✗ | ✗ | ✗ |
| `current_mode_update` | ✓ (permission mode emits) | ◐ (approval mode change) | ✗ | ✗ | ✗ |
| `config_option_update` | ✗ | ✗ | ✗ | ✗ | ✗ |

**SAP additions (proposed)**

| SAP `sessionUpdate` variant | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| `usage_update` | ✓ (`ResultMessage.usage`) | ✓ (`ThreadTokenUsageUpdated`) | ✓ (in `agent_start` / completion event) | ✓ (`ResultMessage.usage` analog) | ✓ (SSE `usage` event) |
| `workspace_status` | ✗ (host-side) | ✗ (host-side) | ✗ (host-side) | ✗ (host-side) | ✗ (host-side) |
| `runtime_status` | ✗ (host-side) | ✗ (host-side) | ✗ (host-side) | ✗ (host-side) | ✗ (host-side) |

**Implication for adapters.** All vendors stream message content and tool calls; the mappings are mostly straightforward. The "long tail" variants — `available_commands_update`, `current_mode_update`, `config_option_update` — exist mostly because Claude Code's slash-command UI surfaced them. Solid Agent adapters MAY emit these for agents that do not natively support them by inferring from config, but most adapters will simply omit them and advertise the corresponding capability as `false`.

## 4. Permission & approval

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| `session/request_permission` (per-tool callback) | ✓ | ✓ (`can_use_tool` callback) | ✓ (approval handler callback) | ✗ (delegated to extension UI) | ◐ (interactive [y/n/!] only) | ✗ (hooks only) |
| Permission outcomes: `allow_once` / `allow_always` / `reject_once` / `reject_always` | ✓ | ✓ (decisions in callback) | ◐ (allow/deny only, no "always") | n/a | ◐ (`y/n/!`) | n/a |
| Mode-based bypass (`bypassPermissions`, etc.) | ◐ (modes) | ✓ (`permission_mode`) | ✓ (`AUTO_REVIEW / DENY_ALL`) | ◐ (`--tools` allowlist) | ◐ (command allowlist) | ◐ (hooks) |
| Auto-approve policy from Host | ◐ (mode = "code") | ✓ | ✓ | ◐ (`tools` arg) | ◐ (allowlist) | ◐ (hooks) |
| Approval guardian (LLM-as-judge for risky ops) | ✗ | ◐ (via custom hook) | ✓ (`ItemGuardianApprovalReview*`) | ✗ | ✗ | ✗ |

**Implication for adapters.** Claude Code and Codex have the cleanest mappings to ACP `request_permission`. Pi sidesteps the problem entirely (security is delegated to the runtime container). AmpCode requires the Solid Agent adapter to fake permission gates by intercepting tool calls before forwarding to the CLI. Cursor's `hooks` model is one-way — the adapter cannot intercept a tool call interactively, only block it deterministically.

## 5. Tool calls

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Built-in tool set (read/write/edit/bash/search) | n/a | ✓ | ✓ (via `shell` + Codex tools) | ✓ (`read`/`write`/`edit`/`bash`/`grep`/`find`/`ls`) | ✓ | ✓ |
| MCP server hosting | ◐ (`mcpCapabilities`) | ✓ (in-process + subprocess MCP) | ✓ (MCP-based tools) | ◐ (feature add) | ✓ | ✓ |
| Custom tool registration (in-process) | ✗ | ✓ (`@tool` / `create_tool`) | ◐ (via MCP) | ✓ (`defineTool`) | ◐ (via MCP) | ✓ (subagents / hooks) |
| Subagent invocation | ✗ | ✓ (Task tool) | ◐ (multi-thread) | ◐ (via tools) | ◐ (via tools) | ✓ |
| Structured / typed tool inputs | ✓ (JSON schema) | ✓ (Anthropic SDK tools) | ✓ (TypeBox-equiv) | ✓ (TypeBox) | ✓ (MCP) | ✓ (MCP) |
| Streamed tool output to client | ✓ (`tool_call_update`) | ✓ (`ToolProgressMessage`) | ✓ (`McpToolCallProgress`) | ✓ (`tool_execution_update`) | ◐ (stream-JSON) | ◐ (SSE) |
| Diff-shaped tool output | ✓ (`diff` content) | ◐ (synthesised from Edit results) | ✗ | ✗ | ✗ | ◐ (server-rendered) |
| Live terminal reference in tool call | ✓ (`terminal` content) | ◐ (Bash tool returns full buffer) | ◐ (via `terminal_id` in payload) | ◐ (background bash via `tool_execution_update`) | ✗ | ✗ |

**Implication for adapters.** Tool calls are where ACP's content vocabulary is richest. Most vendors only emit the final tool result, not a streamed update with intermediate states. Solid Agent adapters can synthesise `tool_call_update` events from progress messages where available, or fall back to a single `completed` update for vendors that only emit final results.

## 6. Multimodal content & structured I/O

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Text content in prompts | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Image content in prompts (`promptCapabilities.image`) | ✓ | ✓ | ✓ (`ImageInput`) | ✓ (base64 + clipboard paste) | ? | ? |
| Audio content (`promptCapabilities.audio`) | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Embedded resources (`promptCapabilities.embeddedContext`) | ✓ | ◐ (via file refs) | ◐ (via file refs) | ◐ (via file refs) | ◐ (via file refs) | ◐ (via codebase indexing) |
| Resource link content blocks | ✓ | ◐ (Read tool output) | ◐ (via files) | ◐ (via files) | ◐ (via files) | ◐ (via codebase indexing) |
| Structured output (JSON-schema-constrained final answer) | ✗ | ✓ (`output_schema`) | ◐ (StructuredOutput tool) | ✗ | ✗ | ? |
| Stop reasons (`end_turn` / `max_tokens` / `cancelled` / etc.) | ✓ | ✓ (`ResultMessage.stop_reason`) | ✓ (`stop_reason` in `RunResult`) | ◐ (event types) | ◐ (in result) | ◐ (in result) |

**Implication for adapters.** Audio input is the most consistent gap; no surveyed vendor accepts audio. Structured output is the most consistent vendor-specific feature; only Claude Code exposes a clean JSON-schema-constrained final-answer mode. Adapters for vendors without structured output can synthesise it by prompting and parsing, but Solid Agent should advertise `structured_output: false` for those.

## 7. Mid-turn control

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Cancel turn (`session/cancel`) | ✓ | ✓ (`interrupt`) | ✓ (`turn.interrupt`) | ✓ (`abort`) | ◐ (Ctrl+C only) | ◐ (lifecycle API) |
| Steer / inject mid-turn input | ✗ | ✗ | ✓ (`turn.steer`) | ✓ (`steer` RPC queues during stream) | ✗ | ✗ |
| Pause / resume mid-turn | ✗ | ✗ | ✗ | ✗ | ✗ | ◐ (SSE reconnect) |
| Reconnect to running session | ◐ (under proposal) | ✓ (session resume) | ✓ (thread resume) | ✓ (session resume) | ✓ (threads) | ✓ (Agents window) |

**Implication for adapters.** Codex's `steer` is the most interesting outlier — it lets the host inject input *into* a running turn rather than cancelling and re-prompting. Solid Agent could promote this to a SAP-extension method (`session/steer`) if there is community appetite (proposed as a follow-up to OQ-9).

## 8. Persistence & resume

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Session ID stable across restarts | ✓ | ✓ | ✓ | ✓ (file path or id) | ✓ (thread id) | ✓ (run id) |
| Local file-backed transcripts | ✗ (host's job) | ✓ (`~/.claude/projects/`) | ✓ (`~/.codex/sessions/`) | ✓ (`~/.pi/agent/sessions/`) | ◐ (synced to ampcode.com) | ◐ (cloud-only) |
| Cross-device resume | ✗ (host's job) | ◐ (via file sync) | ◐ (via file sync) | ◐ (via file sync) | ✓ (native) | ✓ (native) |
| Tree-structured / forkable sessions | ✗ | ◐ (rewind support) | ◐ (rewind support) | ✓ (parent id, `fork()`, `/tree`) | ✓ (`amp threads fork`) | ✗ |
| Workspace state captured with session | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

**Implication for adapters.** "Workspace state captured with session" is the row where every vendor scores ✗ — and where SAP's `fs.export` capability is most differentiated. An [AgentFS](https://github.com/tursodatabase/agentfs)-backed Workspace can serialise its full state alongside the session transcript; no agent SDK ships this today.

## 9. Pluggable transport (the key dev question)

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Public `Transport` interface | n/a | ✓ (`ClaudeAgentSDK::Transport`) | ◐ (low-level `AppServerClient`) | ✗ (in-process SDK only) | ✗ | ✗ |
| Drop-in alternative transport example | n/a | ✓ ([docs/client.md `E2BCliTransport`](https://github.com/ya-luotao/claude-agent-sdk-ruby/blob/main/docs/client.md)) | ◐ (custom client class) | ✗ | ✗ | ✗ |
| Configurable working directory | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Configurable environment | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Sandbox arg at session creation | ✗ | ◐ (permission_mode) | ✓ (`sandbox:` option) | ✗ | ✗ | ✗ |

**Implication for adapters.** This row is the single biggest predictor of how easy it is to host an agent inside an arbitrary Solid Agent Runtime. Claude Code's public Transport pattern is the easiest — drop in an HTTP-bridged transport, point it at the Workspace, done. Codex requires reaching into the lower-level `AppServerClient`. Pi and AmpCode currently require either subprocess wrapping (with all the brittleness that implies) or a custom adapter that re-implements their wire format. Cursor's HTTP+SSE shape means the adapter has to terminate SSE on the host side and re-emit ACP-shaped notifications.

## 10. Capability summary by Agent

The rollup. Each agent's expected `agent_capabilities` (per [PROTOCOL.md §4.1](./PROTOCOL.md#41-agent-side-capabilities)) when wrapped by a Solid Agent adapter:

| Capability | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| `thinking` | ✓ | ◐ | ✓ | ? | ? |
| `structured_output` | ✓ | ◐ | ✗ | ✗ | ? |
| `image_input` | ✓ | ✓ | ✓ | ? | ? |
| `tools_external` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `session_load` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `interrupt` | ✓ | ✓ | ✓ | ◐ | ◐ |
| `steer` | ✗ | ✓ | ✓ | ✗ | ✗ |

---

## How to read this for Solid Agent

For each agent, the matrix tells you what the `Agents::X` adapter has to do:

- **Cells marked ✓** — the adapter routes through cleanly. Minimal translation.
- **Cells marked ◐** — the adapter has work to do: a custom mapping, a synthesised event, or a translated parameter shape. Document the divergence.
- **Cells marked ⌬** — the adapter depends on a third-party bridge. Use cautiously and pin a version.
- **Cells marked ✗** — the adapter MUST advertise the capability as `false` in `session/new` and let capability negotiation reject mismatches.
- **Cells marked ?** — needs research before promising production support.

If you maintain or work on an agent listed here and a cell is wrong, please open a PR. If your agent is missing, please open an issue with a pointer to its docs and we will add a column.

## Where this matrix points the spec

A few patterns visible across the matrix are worth highlighting as RFC candidates:

1. **`session/steer` as a SAP extension.** Codex and Pi both ship it; the ergonomic story for long-running agents would benefit from a standard wire shape. (Relates to [PROTOCOL OQ-9](./PROTOCOL.md#open-questions).)
2. **`workspace_state` capture-with-session.** Every vendor scores ✗ here, and AgentFS already demonstrates the design. This is the gap Solid Agent's Workspace layer is most positioned to close — see [DESIGN §3](./DESIGN.md#3-the-workspace-contract).
3. **Approval guardian patterns.** Codex's "guardian review" pattern is unusual but valuable for high-risk workflows. Worth considering as an optional capability (`audit.guardian_review`) once SAP has more deployment data.
4. **Transport public interface as a vendor expectation.** Claude Code's `Transport` class is the strongest enabler in the matrix. Encouraging other SDKs to expose a comparable seam would make the entire ecosystem more composable, and is a reasonable upstream ask.

---

## Change log

- **0.1 (this draft)** — Initial matrix. Five agents: Claude Code, Codex, Pi, AmpCode, Cursor. Verified May 2026.
