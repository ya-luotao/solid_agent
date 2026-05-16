# Future SAP Extensions

> **Status:** Draft 0.1 — design notes for capabilities intentionally **out of scope for SAP v0.1**.
> Nothing in this document is normative. It records ideas that are likely to return as RFCs once v0.1 has a reference implementation and real-world usage data.

## Why a separate document

SAP v0.1 deliberately scopes itself to **two roles** — Agent and Runtime — and the minimum capability surface needed to wrap a coding-agent SDK and have it run on a chosen Runtime. The ecosystem is moving quickly enough that a small, defensible spec ships faster than a large, ambitious one.

That said, several richer ideas surfaced in the research that led to v0.1 — particularly around the agent's filesystem and shell substrate — that are interesting and worth keeping visible. This document parks them, with provenance, so they can return as RFCs without re-litigating the discovery work.

## Deferred theme: rich Workspace substrates

The v0.1 spec treats "the directory the agent reads/writes/executes in" as opaque — a POSIX-equivalent filesystem and a shell. The Runtime provides it, the Agent operates on it, and SAP says nothing more. Three families of ideas argue that this view is incomplete.

### Filesystem with overlays, snapshots, and per-tool-call isolation

**Provenance:** [AgentFS](https://github.com/tursodatabase/agentfs) (Turso).

**The idea.** An agent's filesystem differs in kind from a human's filesystem because the agent operates speculatively and on the basis of incomplete information. Every tool call is a candidate edit that the agent (or its host) may want to retain, discard, or compare against alternatives. A POSIX filesystem makes this expensive; a copy-on-write filesystem with named overlays makes it cheap. Beyond overlays:

- A **tool-call audit log** turns the workspace into its own observability surface — every action recorded, queryable, reproducible.
- **Snapshots and exports** turn the workspace into a portable artefact — one file, moved between machines.
- A **co-located key-value store** lets the agent accumulate structured memory without going through the filesystem.

**What a future SAP RFC might add.**

- Capabilities: `fs.overlay`, `fs.tool_isolation`, `fs.snapshot`, `fs.export`, `memory.kv`, `audit.tool_calls`, `audit.fs_changes`, `audit.query`.
- Methods: `fs/begin_overlay`, `fs/commit_overlay`, `fs/discard_overlay`, `fs/snapshot`, `fs/restore_snapshot`, `fs/export`, `memory/get/set/delete/list`, `audit/query_tool_calls`, `audit/query_fs_changes`.
- Manifest extensions: `memory.seeds`, `capabilities_required` extended to include these.

**Why deferred.** These capabilities deeply couple to the Agent — an agent has to know it has overlays to use them well, has to know it has memory to write into it. SAP v0.1 wants to ship a defensible Agent↔Runtime contract before adding workspace capabilities that change Agent prompts and tool schemas. Once we have data on how N agents are wrapped, we can design overlay semantics that don't force every Agent to either understand or stub them.

### Filesystem with mounted heterogeneous backends

**Provenance:** [Mirage](https://github.com/strukto-ai/mirage) (Strukto).

**The idea.** Many of the integrations an agent needs — S3 buckets, Google Drive folders, Slack histories, GitHub PRs, Redis caches, SSH targets — fit a "directory tree of resources" abstraction better than they fit "wire up N MCP servers." Mounting these as paths under `/s3/...`, `/slack/...`, `/gdocs/...` turns "configure N tools" into "the workspace has more paths" and lets the agent use familiar Unix-shaped composition (pipes, grep, find) over heterogeneous data.

**What a future SAP RFC might add.**

- Capability: `fs.mount` with a sub-capability per backend kind (`mount.s3`, `mount.gdocs`, `mount.slack`, `mount.github`, …).
- Methods: `fs/mount`, `fs/unmount`, `fs/list_mounts`.
- Manifest field: richer `mounts:` (already in v0.1 but currently anonymous).

**Why deferred.** The MCP boundary question (see [DESIGN.md "Workspace vs MCP"](./DESIGN.md)) is sharpest here. MCP servers already provide structured access to many of these data sources. Before SAP defines a competing mount surface, we need to be clear about when "mount as path" is genuinely better than "expose as MCP tool" — and that requires actual usage data, not just speculation.

### Shell with in-process and pluggable interpreters

**Provenance:** [just-bash](https://github.com/vercel-labs/just-bash) (Vercel Labs).

**The idea.** The shell is the agent's primary verb. Its environment — what filesystem it sees, what network it has, what execution limits apply, whether `python` and `sqlite` are in-process — is constitutive of agent behaviour, not a runtime detail. Some hosts (tests, CI smoke runs, sandboxed product features) want a shell that is fast and in-process; others want a subprocess; others want a microVM. Today every coding-agent SDK assumes a subprocess and the choice is implicit.

**What a future SAP RFC might add.**

- Capabilities: `exec.in_process`, `exec.languages.python`, `exec.languages.javascript`, `exec.languages.sqlite`, `exec.network`, `exec.limits`.
- Methods: extended `terminal/create` parameters for execution limits; `terminal/eval` for in-process language execution.

**Why deferred.** v0.1's `terminal/*` methods cover the subprocess case, which is what every shipped agent SDK already uses. Adding a richer execution model before we have data on which agents benefit from it would be premature.

## Deferred theme: a host-side Tools layer

**Provenance:** the gap between ACP-native agents (which speak `session/update.tool_call` directly) and function-calling agents (which expect specific JSON schemas like `bash(command, cwd?)`).

**The idea.** A Tools layer is a reusable registry of named tool definitions — `Bash`, `ReadFile`, `WriteFile`, `Edit`, `Search` — that adapt non-ACP agents to a Runtime-shaped backend. Each tool has a JSON schema (the function-calling signature) and an executor that routes the call through Runtime methods. Hosts wiring a vanilla function-calling agent to a Solid Agent Runtime would pull in a `Tools::Bash` rather than write the translation by hand.

**Why deferred.** This is a Host-side convenience and not a wire concept. v0.1 leaves it to Hosts to write their own translation. A future RFC can standardise the tool definitions once we see what the reference implementation actually needs.

## Deferred theme: mid-turn steering as a wire method

**Provenance:** Codex's `turn.steer` (inject new input mid-turn) and Pi's `steer` command (queue input during streaming). Both are pre-ACP behaviours that ACP has not yet specified.

**The idea.** Long-running coding-agent turns benefit from the ability to inject new instructions without cancelling and re-prompting. SAP could promote this to a standard wire method (`session/steer`) once two or more agents and a host UI use it.

**Why deferred.** Only two agents in [COMPATIBILITY.md](./COMPATIBILITY.md) ship steer today, and the semantics differ between them (Codex injects between tool calls; Pi queues during streaming). v0.1 acknowledges the divergence and waits for convergence.

## Deferred theme: approval guardian

**Provenance:** Codex's `ItemGuardianApprovalReview*` notifications — a separate LLM acts as a judge before risky tool calls are approved.

**The idea.** Once SAP has audit log infrastructure, a "guardian review" capability could be a clean optional bundle: a second agent observes a session's tool calls and gates the risky ones with structured rationale. Useful for high-trust autonomous workflows.

**Why deferred.** This depends on the audit substrate, which is itself deferred. Naturally moves into a later spec cycle.

## Convention for graduating ideas from this file

Items in this document graduate to the main spec via the RFC process described in [ROADMAP.md](./ROADMAP.md). Concretely:

1. An RFC issue cites the relevant section here and proposes the concrete protocol additions.
2. At least one reference implementation lands the capability behind a feature flag.
3. At least two independent Agents or Runtimes adopt it.
4. The corresponding section moves from `FUTURE.md` into `PROTOCOL.md` and `DESIGN.md`, with a `CHANGELOG` entry noting the version that introduced it.

This document is also where to add **new** deferred ideas. Open a PR with a section describing the idea, the provenance, and why deferring is right for now.

---

## Change log

- **0.1 (this draft)** — Initial parking lot, created as SAP v0.1 trimmed its scope to Agent + Runtime. Captures workspace substrate ideas (overlay/audit/memory, mounts, in-process shell), Tools layer, mid-turn steering, and guardian review.
