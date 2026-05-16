# Architecture Decision Records

This directory holds the project's Architectural Decision Records (ADRs). Each ADR captures one design decision, its context, the alternatives considered, and the consequences.

ADRs are numbered, never rewritten, and superseded only by a newer ADR that cites the predecessor. The format is loosely based on [Michael Nygard's template](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions).

## Index

| # | Title | Status |
|---|---|---|
| [0001](./0001-composable-not-orthogonal.md) | Composable, not orthogonal | Accepted |
| [0002](./0002-solid-agent-vs-mcp.md) | Solid Agent vs MCP — where the boundary is | Accepted |
| [0003](./0003-defer-workspace-substrates.md) | Defer workspace substrate capabilities to future SAP versions | Accepted |

## How to add an ADR

See [ROADMAP.md → RFC process](../../ROADMAP.md#rfc-process). In short: open an issue, get one supporter and one reviewer, draft an ADR PR with the template, land alongside the corresponding spec change.

## Template

```markdown
# ADR-NNNN: <Title>

- **Status:** Proposed | Accepted | Superseded by ADR-MMMM
- **Date:** YYYY-MM-DD
- **Driven by:** <issue link, RFC, person, etc.>

## Context

<What's the situation? Why is a decision needed?>

## Decision

<What was decided?>

## Consequences

**Intended:**
- ...

**Accepted downsides:**
- ...

## Alternatives considered

- **<alt 1>** — <why rejected>
- **<alt 2>** — <why rejected>
```
