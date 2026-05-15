# Solid Agent — Roadmap

> **Status:** Draft 0.1 — direction-of-travel, not a commitment.
> Items are best understood as RFC slots; community proposals can rearrange or replace any of them.

This is a phased plan for the Solid Agent specification and its first reference implementation. Each phase describes **what the specification needs to be considered "done enough" to use at that stage**, and what the reference implementation needs to deliver to validate it.

The phases are sequential because each unlocks the next. Dates are intentionally absent — the project moves at the speed of review and contributions.

The plan exercises three orthogonal axes of progress (see [DESIGN.md](./DESIGN.md)):

- **Agent** axis — backends like Claude Code, Codex, AmpCode, Cursor, Pi.
- **Workspace** axis — substrates: POSIX, AgentFS-backed (overlay / audit / portable), Mirage-backed (mounted heterogeneous backends), just-bash-backed (in-process shell).
- **Runtime** axis — execution boundaries: Local, E2B, Daytona, Modal, Cloudflare Containers, Vercel Sandboxes, Runloop, Blaxel, OpenAI built-in.

The roadmap deliberately advances all three axes in alternating phases so the spec is exercised by genuinely diverse combinations rather than a long single-vendor chain.

---

## Phase 0 — Specification skeleton (current)

**Goal:** Establish enough of the spec that an implementer could start a prototype.

- [x] Repository scaffold.
- [x] [`PROTOCOL.md`](./PROTOCOL.md) draft 0.1 — wire shapes, ACP alignment, Workspace substrates (fs / shell / memory / audit), capability registry, manifest, errors.
- [x] [`DESIGN.md`](./DESIGN.md) draft 0.1 — three-axis architecture (Agent × Workspace × Runtime), Workspace as a first-class pillar, non-goals.
- [x] [`ROADMAP.md`](./ROADMAP.md) — phases, RFC process, open questions.
- [x] [`COMPATIBILITY.md`](./COMPATIBILITY.md) — ACP-rooted matrix across Claude Code, Codex, Pi, AmpCode, Cursor.
- [ ] `LICENSE` — MIT (in repo).
- [ ] `CONTRIBUTING.md` — how to file RFCs and conformance reports.
- [ ] `docs/adr/` directory — Architectural Decision Records as design questions are resolved.
- [ ] First round of community feedback collected as issues.

### Phase 0 exit criteria

- The protocol document is internally consistent (no contradictions between sections).
- At least one independent reviewer has read it end-to-end and either agreed it is buildable or filed specific objections.
- All ten open questions in [`PROTOCOL.md`](./PROTOCOL.md#open-questions) have at least one proposed answer in an issue.

---

## Phase 1 — Reference implementation minimum

**Goal:** Prove the spec is implementable by shipping a working Ruby gem that can do one realistic end-to-end run.

The defining end-to-end test: **"open a session with the Claude Code Agent on a POSIX Workspace hosted by the Local Runtime, send a prompt that requires reading and editing a file, receive `session/update` events, and observe the file change on disk."**

### Spec deliverables

- `PROTOCOL.md` draft 0.2 — incorporate Phase 0 feedback; freeze the message shapes for `session/new`, `session/prompt`, `session/update`, `session/cancel`, `session/request_permission`.
- ADRs for any non-trivial decisions taken during implementation.

### Reference implementation deliverables

- `Agents::ClaudeCode` — adapter over [`claude-agent-sdk-ruby`](https://github.com/ya-luotao/claude-agent-sdk-ruby).
- `Agents::Mock` — deterministic test double.
- `Workspaces::Posix` — the default Workspace: real filesystem + subprocess shell, no memory or audit.
- `Workspaces::Mock` — in-memory FS and scripted terminal.
- `Runtimes::Local` — provisions a POSIX Workspace on the host machine.
- `Runtimes::Mock` — provisions a Mock Workspace.
- Stdio transport, end to end.
- Test suite: 2 Agents × 2 Workspaces × 2 Runtimes smoke matrix.

### Phase 1 exit criteria

- A user can run the published quick-start example from a fresh `bundle install`.
- The protocol document still matches the implementation.
- Independent contributors can write a new Runtime or Workspace by following the documentation alone.

---

## Phase 2 — Multi-agent, multi-runtime

**Goal:** Validate the Agent and Runtime axes by shipping one more Agent backend and one cloud Runtime, confirming any pair composes.

### Spec deliverables

- `PROTOCOL.md` draft 0.3 — finalise HTTP/SSE transport, reconnection semantics, capability registry expansion.
- Capability registry promoted to its own document (`CAPABILITIES.md`) with an RFC process for additions.
- Conformance test fixtures (JSON files) any implementation can run against itself.

### Reference implementation deliverables

- `Agents::Codex` — adapter over [`codex-rb`](https://github.com/openai/codex-rb) or equivalent.
- `Agents::ACPDirect` — generic adapter for any ACP-speaking child process.
- `Runtimes::E2B` — adapter over [`e2b-ruby`](https://github.com/heymoney/e2b-ruby).
- HTTP/SSE server: minimum subset of `/v1/sessions/*` endpoints.
- Manifest application path end to end (clone repo → seed files → run agent → observe edits).

### Phase 2 exit criteria

- All `(Agent, Workspace, Runtime)` pairs in the smoke matrix pass with the default POSIX Workspace: ClaudeCode×POSIX×Local, ClaudeCode×POSIX×E2B, Codex×POSIX×Local, Codex×POSIX×E2B, ACPDirect×POSIX×Local, ACPDirect×POSIX×E2B.
- A separately-authored Host can drive the HTTP server using only the published surface description.
- At least one external implementation has filed a "conformance report" issue.

---

## Phase 3 — Workspace substrates

**Goal:** Validate the Workspace axis by adding richer substrates and proving they compose with any Agent and any Runtime.

This phase is where Solid Agent's first-class Workspace pays off. After it lands, a Host should be able to choose its Workspace shape independently from its Runtime and Agent.

### Spec deliverables

- `PROTOCOL.md` draft 0.4 — promote overlay, snapshot, mount, memory, and audit methods from extension to core where stable. Tighten `fs.tool_isolation` semantics (resolves OQ-9).
- ADRs covering: how overlays interact with terminals, when memory `set` is durable, what `audit.tool_calls` records vs what stays in the Host's event store.

### Reference implementation deliverables

- `Workspaces::AgentFS` — Workspace backed by [AgentFS](https://github.com/tursodatabase/agentfs). Advertises `fs.overlay`, `fs.tool_isolation`, `memory.kv`, `audit.tool_calls`, `audit.fs_changes`, `fs.export`.
- `Workspaces::Mirage` — Workspace backed by [Mirage](https://github.com/strukto-ai/mirage). Advertises `fs.mount` plus a subset of backend kinds (`mount.s3`, `mount.gdocs`, `mount.slack`, `mount.github` — actual list to be decided alongside Mirage's stable surface).
- `Workspaces::JustBash` — Workspace whose Shell substrate is [just-bash](https://github.com/vercel-labs/just-bash), running in-process. Advertises `exec.in_process`, `exec.languages.*`, `exec.limits`. Useful for tests, CI smoke runs, and untrusted execution where subprocess isolation is unwanted.
- Composition tests: AgentFS Workspace inside Local Runtime, AgentFS Workspace inside E2B Runtime, Mirage Workspace + Codex Agent, just-bash Workspace + Claude Code Agent.

### Phase 3 exit criteria

- A Workspace can be added to the reference implementation in under ~500 lines of Ruby plus its substrate's native SDK.
- The audit substrate produces records that can reconstruct a session even after the Host's event stream is lost.
- An AgentFS-backed Workspace can be exported as a single file, transferred to another machine, and reopened as a new Solid Agent session.

---

## Phase 4 — Runtime diversity

**Goal:** Demonstrate that the Runtime contract scales beyond a single cloud provider.

### Spec deliverables

- Snapshot, pause, resume semantics promoted from extension to core spec where stable.
- `host_for_port` and `mount.*` capabilities clarified with worked examples per provider.
- Recommended floor for reconnect-buffer window (resolves OQ-6).

### Reference implementation deliverables

- `Runtimes::Daytona`
- `Runtimes::OpenAIBuiltin` — Unix-local equivalent of the OpenAI Agents SDK's first-party sandbox.
- Capability-driven Workflow examples: pause-between-nodes, snapshot-before-risky-action, port-forward to a long-running dev server.

### Phase 4 exit criteria

- A Runtime can be added in under 500 lines of Ruby plus a manifest provisioning script.
- The capability registry has accommodated at least one community-proposed capability.

---

## Phase 5 — Agent diversity

**Goal:** Demonstrate that the Agent contract scales beyond Claude Code and Codex.

### Spec deliverables

- `session/load` (resume) semantics standardised across Agents that support it.
- Recommendations for how an Agent exposing proprietary event types should map them into ACP's update set.

### Reference implementation deliverables

- `Agents::AmpCode` (via ACPDirect or a specialised adapter, depending on Amp's protocol direction).
- `Agents::Cursor` (likewise).
- `Agents::Pi` (likewise).
- Documentation page on "how to add an Agent" with worked examples.

### Phase 5 exit criteria

- At least three Agents from independent vendors run on at least three Workspaces and three Runtimes, all in independent combinations.
- The "how to add an Agent" page is followable by someone outside the core team.

---

## Phase 6 — Operability

**Goal:** Make the reference implementation production-grade.

### Spec deliverables

- Observability extension — what's in `usage_update`, what an OpenTelemetry mapping looks like, what trace boundaries SAP recommends.
- Security model draft — secret-store reference resolution, egress-policy hints (resolves OQ-7).

### Reference implementation deliverables

- Optional persistence adapters: ActiveRecord, Sequel.
- Optional observability adapter: OpenTelemetry tracing for sessions.
- Health-check and metrics endpoints on the HTTP server.
- A "session inspector" web UI that subscribes to a session's stream and renders it. Where the Workspace advertises `audit.tool_calls`, the inspector reads from the audit substrate as a backup data source.

### Phase 6 exit criteria

- A Host can run Solid Agent in production and answer "what happened in this session?" from persisted state alone — either the Host's event store or the Workspace's audit substrate.
- Observability hooks compose with existing OpenTelemetry pipelines.

---

## Phase 7 — Long-tail runtimes

**Goal:** Close the seven-provider gap to the OpenAI Agents SDK list, plus anything else the community has built.

### Reference implementation deliverables

- `Runtimes::Modal`
- `Runtimes::CloudflareContainers`
- `Runtimes::VercelSandboxes`
- `Runtimes::Runloop`
- `Runtimes::Blaxel`

### Phase 7 exit criteria

- The Runtime list in [`README.md`](./README.md) matches the OpenAI Agents SDK list at parity.
- All Runtimes pass the conformance test suite.

---

## Phase 8 — Beyond coding agents

**Goal:** Confirm or refute the assumption that Solid Agent generalises beyond coding tasks.

Intentionally open-ended. Possible directions:

- Browser-driving agents (operator-style) — does ACP's tool-call vocabulary stretch, or do we need a new capability?
- Long-running autonomous agents — does the session model scale to runs measured in days?
- Multi-agent collaboration within a single session — can a Workspace host two Agents concurrently, sharing FS and memory under different namespaces?

Output of this phase is a decision: keep the spec coding-focused, or evolve toward a broader "Agent Workspace" protocol.

---

## RFC process

Decisions of any consequence are captured as **ADRs** (Architecture Decision Records) in `docs/adr/`. The process:

1. Open an issue describing the proposal.
2. If the issue gathers ≥1 supporter and ≥1 reviewer, draft an ADR PR.
3. ADR template: context, proposed decision, alternatives considered, consequences (both intended and accepted downsides), status (`proposed | accepted | superseded`).
4. ADRs are numbered (`ADR-0001-…`) and never rewritten — superseding ADRs cite their predecessor.
5. The `PROTOCOL.md` change accompanies the ADR PR.

Process scope:

- **Anything in `PROTOCOL.md`** — requires an ADR.
- **Anything in the capability registry** — requires an ADR.
- **`DESIGN.md` shape and section structure** — does not require an ADR, but PRs should explain motivation.
- **Reference-implementation internals** — does not require an ADR.

---

## Conformance

Each major spec milestone ships a **conformance test suite**: a set of JSON-RPC transcripts and expected behaviours that any implementation can run against itself. A new Agent, Workspace, or Runtime claiming SAP compatibility is encouraged to publish a conformance report (pass/fail per scenario) as part of its release notes.

Conformance is per-capability, not per-spec-version. Reports look like:

```
my-workspace v0.3.1
  Role: Workspace
  SAP protocol_version: 1
  Capabilities advertised: fs, fs.list, fs.overlay, fs.tool_isolation, memory.kv, audit.tool_calls
  Conformance:
    fs:                12/12 pass
    fs.list:            4/4  pass
    fs.overlay:         8/8  pass
    fs.tool_isolation:  6/6  pass
    memory.kv:          5/5  pass
    audit.tool_calls:   4/4  pass
```

---

## Non-goals (current and forecasted)

- A standardised model serving API — Solid Agent does not specify how an Agent reaches an LLM.
- A standardised auth model for end-users — Hosts handle their own auth.
- A bundled workflow engine — workflows compose on top.
- A bundled UI — `DESIGN.md` explicitly leaves rendering to Hosts.
- An IDE plugin — could be built on top, but out of scope here.
- A standardised skill format — skill bundles live inside manifests; how they are authored or distributed is out of scope.

---

## How to help

- **Read [`PROTOCOL.md`](./PROTOCOL.md) and file an issue** for any ambiguity, missing case, or design objection.
- **Propose a capability** for the registry if your Agent, Workspace, or Runtime needs something that's not there.
- **Sketch an adapter** for an Agent, Workspace, or Runtime not yet in the roadmap — even a 200-line proof of concept is enough to validate the contract.
- **Build a Host** — the protocol is only useful if real applications drive it. The earliest Hosts will shape the spec the most.

Discussion happens in GitHub Issues for now; a Discord or similar may follow once volume justifies it.
