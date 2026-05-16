# Architecture Decision Records

This directory holds the project's Architectural Decision Records (ADRs). Each ADR captures one design decision, its context, the alternatives considered, and the consequences.

ADRs are numbered, never rewritten, and superseded only by a newer ADR that cites the predecessor.

## Index

| # | Title | Status |
|---|---|---|
| [0001](./0001-composable-not-orthogonal.md) | Composable, not orthogonal | Accepted |

## How to add an ADR

See [ROADMAP.md → RFC process](../../ROADMAP.md#rfc-process). In short: open an issue, get one supporter and one reviewer, draft an ADR PR using the template below, land alongside the corresponding spec change.

## Template

```markdown
# ADR-NNNN: <Title>

- **Status:** Proposed | Accepted | Superseded by ADR-MMMM
- **Date:** YYYY-MM-DD

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
