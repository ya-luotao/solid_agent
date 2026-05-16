# Agent SDK Compatibility Matrix

> **Status:** Draft 0.1 — open for correction. Vendor surfaces change quickly; please file issues or PRs when you spot drift.
> **Purpose:** for adapter writers. Each row tells you what a coding-agent SDK supports natively, so you know what the Solid Agent adapter has to translate vs reuse.
> **Last verified:** May 2026.

This document takes ACP as the canonical baseline and shows whether each major coding-agent SDK has a native, partial, or no equivalent.

## Legend

| Symbol | Meaning |
|:-:|---|
| **✓** | Native equivalent. The SDK exposes this directly. |
| **◐** | Partial / non-standard equivalent. Available but with caveats. |
| **⌬** | Available only through a third-party bridge or community shim. |
| **✗** | No equivalent. Solid Agent adapter would have to synthesise or advertise as unsupported. |
| **?** | Undocumented in publicly available sources. Help wanted. |

## Agents covered

| Agent | Identifier | Source | License | Wire format |
|---|---|---|---|---|
| **Claude Code** | `claude_code` | [`claude-agent-sdk-ruby`](https://github.com/ya-luotao/claude-agent-sdk-ruby) and the [Python equivalent](https://github.com/anthropics/claude-agent-sdk-python) | MIT (SDK), proprietary CLI | newline-delimited JSON over stdio |
| **Codex** | `codex` | [`codex-rb`](https://github.com/ya-luotao/codex-rb) | Apache-2.0 (SDK), proprietary CLI | JSON-RPC 2.0 over stdio |
| **Pi** | `pi` | [`@earendil-works/pi-coding-agent`](https://github.com/badlogic/pi-mono) | MIT | LF-delimited JSONL over stdio (custom 3-message protocol) |
| **AmpCode** | `ampcode` | `@ampcode/cli` (Sourcegraph) | Closed source | "Claude Code compatible" stream-JSON |
| **Cursor** | `cursor` | `@cursor/sdk` | Closed source | HTTP + SSE (Cloud Agents API) |
| **ACPDirect** | `acp_direct` | reference adapter | MIT | Native ACP |

## 1. Wire & transport

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| JSON-RPC 2.0 framing | ✓ | ✗ (stream-JSON) | ✓ | ✗ (custom) | ✗ (stream-JSON) | ✗ (HTTP+SSE) |
| Stdio transport | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| Pluggable transport hook | n/a | ✓ (`Client.new(transport_class: …)`) | ◐ (low-level `AppServerClient`) | ✗ | ✗ | ✗ |

**Implication.** Claude Code, Codex, Pi, and AmpCode are stdio-newline-JSON families. Wrapping them inside an E2B microVM is a matter of replacing the transport. Cursor's HTTP+SSE shape needs an SSE-to-ACP bridge instead.

## 2. Lifecycle methods

| ACP method | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| `initialize` | ✗ (init via first message) | ◐ (at `thread_start`) | ◐ (mode flag) | ✗ (subprocess args) | ◐ (`Agent.create`) |
| `session/new` | ◐ (`Client.connect`) | ✓ (`thread_start`) | ✓ (RPC `new_session`) | ✓ (`amp threads new`) | ✓ (`Agent.create`) |
| `session/load` | ✓ (resume support) | ✓ (`thread_resume`) | ✓ (`switch_session`) | ✓ (`amp threads continue`) | ✓ (cloud runs persist) |
| `session/prompt` | ✓ (`query`) | ✓ (`thread.run`) | ✓ (RPC `prompt`) | ✓ (stdin pipe) | ✓ (`.send` / `.stream`) |
| `session/cancel` | ✓ (`interrupt`) | ✓ (`turn.interrupt`) | ✓ (RPC `abort`) | ◐ (Ctrl+C only) | ◐ (lifecycle API) |
| `session/close` | ✓ (`disconnect`) | ✓ | ✓ | ✓ | ✓ |

## 3. SessionUpdate variants the agent emits

| ACP variant | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| `agent_message_chunk` | ✓ (`TextBlock` in `AssistantMessage`) | ✓ (`AgentMessageDelta`) | ✓ (`text_delta`) | ✓ | ✓ |
| `agent_thought_chunk` | ✓ (`ThinkingBlock`) | ◐ | ✓ (`thinking_delta`) | ? | ? |
| `tool_call` | ✓ (`ToolUseBlock`) | ✓ (`ItemStarted`) | ✓ (`tool_execution_start`) | ✓ | ✓ |
| `tool_call_update` | ✓ (`ToolProgressMessage` + `UserMessage`) | ✓ (`ItemCompleted`) | ✓ (`tool_execution_end`) | ✓ | ◐ |
| `plan` | ✓ (via TodoWrite tool) | ◐ | ◐ | ◐ | ◐ |
| `available_commands_update` | ✓ (`PromptSuggestionMessage`) | ✗ | ✗ | ✗ | ✗ |
| `current_mode_update` | ✓ | ◐ | ✗ | ✗ | ✗ |

## 4. Permission & approval

| Capability | ACP | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Per-tool permission callback | ✓ | ✓ (`can_use_tool`) | ✓ (approval handler) | ✗ (delegated) | ◐ (interactive `[y/n/!]`) | ✗ (hooks only) |
| Mode-based bypass | ◐ | ✓ (`permission_mode`) | ✓ (`AUTO_REVIEW`/`DENY_ALL`) | ◐ (`--tools` allowlist) | ◐ | ◐ (hooks) |

## 5. Multimodal & structured I/O

| Capability | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| Text input | ✓ | ✓ | ✓ | ✓ | ✓ |
| Image input | ✓ | ✓ | ✓ (base64 + paste) | ? | ? |
| Audio input | ✗ | ✗ | ✗ | ✗ | ✗ |
| Structured output (JSON schema) | ✓ (`output_schema`) | ◐ (StructuredOutput tool) | ✗ | ✗ | ? |

## 6. Mid-turn control

| Capability | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| Cancel turn | ✓ | ✓ | ✓ | ◐ (Ctrl+C only) | ◐ |
| Steer / inject input mid-turn | ✗ | ✓ (`turn.steer`) | ✓ | ✗ | ✗ |

## 7. Persistence & resume

| Capability | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| Local file-backed transcripts | ✓ (`~/.claude/projects/`) | ✓ (`~/.codex/sessions/`) | ✓ (`~/.pi/agent/sessions/`) | ◐ (synced to ampcode.com) | ◐ (cloud-only) |
| Cross-device resume | ◐ (file sync) | ◐ (file sync) | ◐ (file sync) | ✓ (native) | ✓ (native) |

## 8. Pluggable transport (the key adapter question)

| Capability | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| Public `Transport` interface | ✓ (`ClaudeAgentSDK::Transport`) | ◐ (`AppServerClient`) | ✗ (in-process SDK only) | ✗ | ✗ |
| Documented example with alternative transport | ✓ ([`docs/client.md` E2B example](https://github.com/ya-luotao/claude-agent-sdk-ruby/blob/main/docs/client.md)) | ✗ | ✗ | ✗ | ✗ |

**Implication.** This is the row that matters most for adapter writers. Wrapping Claude Code is straightforward — drop in an alternative transport, point it at a runtime, done. Codex requires reaching into the lower-level `AppServerClient`. Pi and AmpCode need subprocess wrappers or custom adapters that re-implement their wire formats. Cursor needs an SSE-to-ACP bridge on the Host side.

## 9. Expected `agent_capabilities` per adapter

When wrapped by a Solid Agent adapter:

| Capability | Claude Code | Codex | Pi | AmpCode | Cursor |
|---|:-:|:-:|:-:|:-:|:-:|
| `thinking` | ✓ | ◐ | ✓ | ? | ? |
| `structured_output` | ✓ | ◐ | ✗ | ✗ | ? |
| `image_input` | ✓ | ✓ | ✓ | ? | ? |
| `tools_external` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `session_load` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `interrupt` | ✓ | ✓ | ✓ | ◐ | ◐ |

## How to read this for Solid Agent

- **✓** — the adapter routes through cleanly. Minimal translation.
- **◐** — the adapter has work to do: custom mapping, synthesised event, translated parameter shape. Document the divergence.
- **⌬** — the adapter depends on a third-party bridge. Use cautiously and pin a version.
- **✗** — the adapter MUST advertise the capability as `false` in `session/new`. Capability negotiation rejects mismatches.
- **?** — needs research before promising production support.

If a cell is wrong, please open a PR. If your agent is missing, please open an issue with a pointer to its docs.

## Change log

- **0.1** — Initial matrix. Five agents: Claude Code, Codex, Pi, AmpCode, Cursor. Verified May 2026.
