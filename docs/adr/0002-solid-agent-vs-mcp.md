# ADR-0002: Solid Agent vs MCP — where the boundary is

- **Status:** Accepted
- **Date:** 2026-05-15
- **Driven by:** [Issue #1, P0-2](https://github.com/ya-luotao/solid_agent/issues/1)

## Context

Early drafts of the spec did not preempt an obvious challenge: *"My MCP server can already expose filesystem reads, writes, shell, and a key-value store. Why isn't a coding-agent runtime just a fleet of MCP servers?"*

Failure to answer this question in the spec itself has two consequences:

1. The first reasonable reader (HN, lobste.rs, MCP-ecosystem-adjacent developer) hits this challenge with no answer, and the spec loses credibility.
2. The project loses scope clarity — if SAP and MCP overlap without delineation, contributors can't tell which problems belong where.

The reviewer flagged this as a P0 issue without which the spec lacks "existence legitimacy."

## Decision

Add a dedicated section to `DESIGN.md` (now `§3 Solid Agent vs MCP — where the boundary is`) and a non-goal entry to `ROADMAP.md` explicitly stating SAP does not try to replace MCP.

The boundary is articulated as:

- **MCP** answers *"where do the tools come from?"* It defines a discoverable surface of tools, resources, and prompts an agent's reasoning loop can call.
- **SAP** answers *"where does the agent run?"* It defines the lifecycle and boundary of the agent's session — session creation, prompting, cancellation, runtime provisioning, snapshots, pause/resume, manifest seeding, HTTP transport.

These concerns are different shapes:

- MCP does not specify how an Agent is started, paused, snapshotted, resumed, or moved between runtimes.
- MCP does not specify a Manifest for seeding workspace state before the Agent boots.
- MCP does not specify how to recover an Agent's event stream after a network drop.
- MCP does not specify session-level permission flow (it has `elicit` for per-tool only).

They compose. A typical Solid Agent session may include several MCP servers as part of the Agent's tool catalogue, with the Runtime hosting those MCP servers in its process tree.

## Consequences

**Intended:**

- `DESIGN.md §3` documents the boundary explicitly, with concrete examples of composition.
- The "universal operations live in ACP client-side; domain-specific ones live in MCP" rule of thumb is documented.
- `ROADMAP.md` non-goals list includes "Replacement for MCP."
- Reviewers and contributors have a clear answer when the question comes up; we don't have to re-derive it each time.

**Accepted downsides:**

- The boundary has a fuzzy region — e.g., a host can expose `fs.read` either through ACP's client-side method or through an MCP tool. We document a heuristic ("universal operations live in the runtime, domain-specific live in MCP") but accept that some implementations may straddle the line.
- We commit to the position that ACP's fs/terminal as first-class client-side methods is the right design, even though MCP could technically express the same operations as tools.

## Alternatives considered

- **Stay silent and let the reader infer.** Rejected — this is the question that decides whether SAP earns shelf space alongside MCP.
- **Position SAP as a layer above MCP**, e.g., "SAP is the runtime; MCP is the toolbox." Rejected — too imprecise. The fact is they're peers in the agent stack with different concerns, and the boundary deserves explicit articulation.
- **Position SAP as MCP-only**, i.e., expose runtime methods as MCP tools and not duplicate ACP's client surface. Rejected — fs and terminal are universal across coding agents; making them discoverable per-server adds per-session overhead with no benefit, and ACP has already made the opposite choice.
