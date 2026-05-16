# Solid Agent

> **Status:** Early draft — open for discussion. Pre-alpha; no published gem yet.

**Solid Agent** is an open protocol — built on top of the [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) — that wraps coding-agent SDKs and sandbox runtimes so they compose. A Ruby reference implementation ships alongside the spec.

```ruby
SolidAgent.run('add tests for the auth module', agent: :claude, sandbox: :e2b)
```

The protocol specifies how to wrap an agent SDK so it speaks ACP, how to wrap a sandbox SDK so it fulfils ACP's client-side methods (`fs/*`, `terminal/*`), and adds a small set of runtime lifecycle methods (`runtime/snapshot`, `runtime/pause`, `runtime/resume`, `runtime/manifest_apply`) that ACP does not yet specify.

```
              ┌────── Agent ──────┐         ┌────── Runtime ──────┐
              │ Claude Code       │         │ Local                │
              │ Codex             │         │ E2B                  │
              │ AmpCode           │ ─ ACP ─ │ Daytona              │
              │ Cursor            │         │ Modal                │
              │ Pi                │         │ Cloudflare           │
              │ ACPDirect         │         │ Vercel               │
              │ … any ACP agent   │         │ Runloop / Blaxel     │
              └───────────────────┘         │ OpenAI built-in      │
                                             │ … any ACP runtime    │
                                             └──────────────────────┘
```

## Why this exists

Coding-agent SDKs and sandbox runtimes have proliferated. Composing them by hand for every project means re-writing the same glue — translating between each SDK's native event vocabulary and ACP-shaped messages, bridging the agent CLI's stdio through whichever sandbox you picked, handling reconnection, snapshots, and so on.

Solid Agent puts that glue in one place. The protocol is small (ACP + four runtime lifecycle methods + a manifest); the reference implementation is a Ruby gem.

## Three reference SDKs power the Ruby implementation

| Layer | Backed by |
|---|---|
| Claude Code agent | [`claude-agent-sdk`](https://github.com/ya-luotao/claude-agent-sdk-ruby) — already exposes a pluggable `Transport`, which makes wrapping it through any runtime straightforward. |
| Codex agent | [`codex-rb`](https://github.com/ya-luotao/codex-rb) |
| E2B sandbox | [`e2b`](https://github.com/ya-luotao/e2b-ruby) |

Other Solid Agent agents and runtimes are welcome — see [CONTRIBUTING.md](./CONTRIBUTING.md) for the adapter contract.

## Contents

| File | Purpose |
|---|---|
| [`PROTOCOL.md`](./PROTOCOL.md) | Wire spec — ACP foundation, runtime lifecycle extension, manifest, capability registry, errors. |
| [`DESIGN.md`](./DESIGN.md) | Architecture and rationale — two-axis composition, how wrapping works, Ruby reference implementation. |
| [`COMPATIBILITY.md`](./COMPATIBILITY.md) | Adapter writer's reference — what each agent SDK supports natively, with mapping notes. |
| [`ROADMAP.md`](./ROADMAP.md) | Phased plan, RFC process, open questions. |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to write a new Agent or Runtime adapter. |
| [`docs/adr/`](./docs/adr/) | Architectural Decision Records. |

## Ruby quick start

```ruby
require 'solid_agent'

# One-shot
result = SolidAgent.run('Summarize this repo')

# Pick an agent
SolidAgent.run('Write a Fibonacci function', agent: :codex)
SolidAgent.run('Audit auth.rb',              agent: :claude)

# Pick a sandbox
SolidAgent.run('Run the tests', sandbox: :local)
SolidAgent.run('Run the tests', sandbox: :e2b, sandbox_opts: { template: 'ubuntu-22-04' })

# Stream events
SolidAgent.run('Refactor billing.rb') do |event|
  case event
  when SolidAgent::TextEvent   then print event.text
  when SolidAgent::ToolEvent   then puts "\n[#{event.tool_name}]"
  when SolidAgent::ResultEvent then puts "\nDone (#{event.duration_ms}ms, $#{event.cost_usd})"
  end
end

# Interactive session
session = SolidAgent.session(agent: :claude, sandbox: :e2b)
session.start('Read the codebase')
session.prompt('Now add tests for the auth flow')
session.close
```

## Relationship to existing efforts

- **[Agent Client Protocol](https://agentclientprotocol.com/)** — adopted verbatim as the wire vocabulary. Solid Agent adds runtime lifecycle and manifest on top, and documents the rules for wrapping non-ACP agents.
- **[SandboxAgent](https://sandboxagent.dev/)** — the HTTP surface uses the same `/v1/acp/*` namespacing so existing SandboxAgent clients can talk to a Solid Agent server.
- **OpenAI Agents SDK `agents.sandbox`** — the runtime provider list and the `Manifest` concept come from here.
- **[Model Context Protocol](https://modelcontextprotocol.io/)** — MCP defines what tools an agent can call; Solid Agent defines where the agent runs. They compose at different layers. See [DESIGN §8](./DESIGN.md#8-solid-agent-and-mcp).

## Getting involved

The first useful PRs are likely:

- Clarifications, typos, and ambiguity reports on `PROTOCOL.md`.
- Counter-proposals for any design decision in `DESIGN.md`.
- Sketches of how a specific agent or runtime would adapt to the proposed interface.
- Compatibility-matrix corrections.

Architectural decisions are captured as numbered ADRs in [`docs/adr/`](./docs/adr/).

## License

MIT — see [`LICENSE`](./LICENSE).
