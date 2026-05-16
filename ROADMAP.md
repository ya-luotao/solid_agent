# Roadmap

## Phase 0 — Specification skeleton (current)

- [x] Repository scaffold.
- [x] [`PROTOCOL.md`](./PROTOCOL.md) draft 0.1 — ACP foundation + runtime lifecycle + manifest + capability registry.
- [x] [`DESIGN.md`](./DESIGN.md) draft 0.1 — two-axis architecture, wrapping rules, Ruby reference implementation sketch.
- [x] [`COMPATIBILITY.md`](./COMPATIBILITY.md) — agent SDK feature matrix for adapter writers.
- [x] [`CONTRIBUTING.md`](./CONTRIBUTING.md) — adapter contract for new agents and sandboxes.
- [x] [`docs/adr/`](./docs/adr/) — first ADR landed (composable, not orthogonal).
- [x] `LICENSE` — MIT.

### Phase 0 exit criteria

- The protocol document is internally consistent.
- At least one independent reviewer has read it end-to-end.
- All six open questions in [`PROTOCOL.md`](./PROTOCOL.md#open-questions) have at least one proposed answer.

## Phase 1 — Ruby reference implementation minimum

End-to-end demonstration: open a session with Claude Code on the Local runtime, prompt it to read and edit a file, observe the events, see the file change.

### Spec deliverables

- `PROTOCOL.md` draft 0.2 — incorporate Phase 0 feedback; freeze the `session/*` shapes.
- ADRs for non-trivial decisions taken during implementation.

### Ruby gem deliverables

- `SolidAgent::Agents::Claude` (wraps [`claude-agent-sdk`](https://github.com/ya-luotao/claude-agent-sdk-ruby) via its `Transport` seam).
- `SolidAgent::Agents::Mock`.
- `SolidAgent::Sandboxes::Local` (subprocess via `Open3`).
- `SolidAgent::Sandboxes::Mock`.
- Event types: `TextEvent`, `ThinkingEvent`, `ToolEvent`, `ToolResultEvent`, `ResultEvent`, `ErrorEvent`.
- `SolidAgent.run` / `SolidAgent.session` helpers.
- Manifest type for sandbox seeding.
- Smoke tests: Claude × Local, Mock × Local, Claude × Mock.

## Phase 2 — Codex and E2B; HTTP/SSE; snapshot demo

Demonstrate Runtime portability by snapshotting a session on E2B and resuming it later.

### Spec deliverables

- `PROTOCOL.md` draft 0.3 — finalise HTTP/SSE transport, reconnection semantics, snapshot/pause/resume.
- Conformance test fixtures (JSON files) any implementation can run against itself.

### Ruby gem deliverables

- `SolidAgent::Agents::Codex` (wraps [`codex-rb`](https://github.com/ya-luotao/codex-rb)).
- `SolidAgent::Agents::ACPDirect` — generic adapter for any ACP-speaking child process.
- `SolidAgent::Sandboxes::E2B` (wraps [`e2b`](https://github.com/ya-luotao/e2b-ruby)).
- HTTP/SSE server: minimum subset of `/v1/sessions/*` endpoints.
- Manifest application end-to-end (clone repo → seed files → run agent → snapshot).
- `examples/snapshot-resume/` — runnable demo + 90-second screencast.

## Phase 3 — Runtime diversity

- `SolidAgent::Sandboxes::Daytona`
- `SolidAgent::Sandboxes::OpenAIBuiltin`

Driven by user demand for any of: Modal, Cloudflare Containers, Vercel Sandboxes, Runloop, Blaxel.

## Phase 4 — Agent diversity

- `SolidAgent::Agents::AmpCode`
- `SolidAgent::Agents::Cursor`
- `SolidAgent::Agents::Pi`

Likely via the `ACPDirect` adapter where the upstream agent ships ACP support, or via per-agent shims where it doesn't.

## Phase 5 — Operability

- Optional persistence adapters: ActiveRecord, Sequel.
- Optional observability adapter: OpenTelemetry tracing for sessions.
- Health-check and metrics endpoints on the HTTP server.
- A session inspector web UI.

## RFC process

Non-trivial decisions are captured as **ADRs** in [`docs/adr/`](./docs/adr/). Process:

1. Open an issue describing the proposal.
2. If the issue gathers ≥1 supporter and ≥1 reviewer, draft an ADR PR.
3. ADR template: context, decision, alternatives considered, consequences, status (`proposed | accepted | superseded`).
4. ADRs are numbered (`ADR-0001-…`) and never rewritten — superseding ADRs cite their predecessor.
5. The `PROTOCOL.md` change accompanies the ADR PR.

Process scope:

- **Anything in `PROTOCOL.md`** — requires an ADR.
- **Anything in the capability registry** — requires an ADR.
- **`DESIGN.md` shape and section structure** — no ADR required, but PRs should explain motivation.
- **Reference-implementation internals** — no ADR required.

## Conformance

Each major spec milestone ships a conformance test suite: JSON-RPC transcripts and expected behaviours any implementation can run against itself. A new Agent or Runtime claiming Solid Agent compatibility is encouraged to publish a conformance report.

## Non-goals

- A multi-language standard body. The protocol is wire-portable, but governance stays light and contributor-driven.
- Replacement for ACP. Solid Agent uses ACP for the wire format; we contribute back if a SAP extension proves general enough.
- Replacement for MCP. Tools, resources, and prompts stay in MCP's lane.
- A bundled workflow engine.
- A bundled UI.
- A workspace substrate spec (overlay filesystems, mount kinds, in-process shells, memory/audit stores). The runtime provides a working directory and a shell; richer substrates can come later if usage data demands it.

## How to help

- Read [`PROTOCOL.md`](./PROTOCOL.md) and file an issue for any ambiguity.
- Propose a capability for the registry if your agent or runtime needs something that's not there.
- Sketch an adapter for an agent or runtime not yet in the roadmap.
- Build a Host that drives the protocol — the earliest Hosts will shape the spec the most.
