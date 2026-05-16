# Roadmap

## v0.1 — Claude + Local + E2B

- [ ] `SolidAgent::Agents::Claude` — wraps [`claude-agent-sdk`](https://github.com/ya-luotao/claude-agent-sdk-ruby).
- [ ] `SolidAgent::Sandboxes::Local` — local subprocess.
- [ ] `SolidAgent::Sandboxes::E2B` — wraps [`e2b`](https://github.com/ya-luotao/e2b-ruby) gem; bridges the agent's stdio through E2B's command RPC.
- [ ] Event types: `TextEvent`, `ToolEvent`, `ToolResultEvent`, `ThinkingEvent`, `ResultEvent`, `ErrorEvent`.
- [ ] `SolidAgent.run` (one-shot) and `SolidAgent.session` (interactive) helpers.
- [ ] Manifest type for sandbox seeding (repos, files, env).
- [ ] Smoke tests: Claude × Local, Claude × E2B.
- [ ] Released as `solid_agent` gem on RubyGems.

## v0.2 — Codex

- [ ] `SolidAgent::Agents::Codex` — wraps [`codex-rb`](https://github.com/ya-luotao/codex-rb).
- [ ] Smoke tests: Codex × Local, Codex × E2B.
- [ ] Document any per-agent feature gaps (e.g., capability flags for `image_input`, `thinking`, `interrupt`).

## v0.3+ — driven by user demand

Concrete additions only after at least one user asks. Likely candidates:

- More sandboxes: Daytona, Modal, OpenAI built-in, Vercel Sandboxes.
- More agents: when AmpCode / Cursor / Pi expose Ruby-callable surfaces, or via an ACP shim for any agent that speaks Agent Client Protocol.
- ActiveRecord persistence helper for the event stream.
- Snapshot/pause/resume semantics for sandboxes that support them.
- HTTP/SSE server mode so non-Ruby hosts can drive a Solid Agent session.
- OpenTelemetry tracing integration.

## Non-goals

- **A multi-language protocol spec.** Solid Agent is a Ruby gem. Cross-language interop, if it ever happens, lives in a separate project.
- **Replacing the underlying SDKs.** `claude-agent-sdk`, `codex-rb`, and `e2b` keep their own roadmaps; this gem only composes them.
- **A workflow / DAG engine.** Use Sidekiq, Temporal, your own — whatever fits. Solid Agent gives you the per-session primitive; orchestration is the host application's job.
- **A standard wire format across vendors.** Each agent SDK already has its own; the gem normalises events in Ruby, not on a wire.

## How to help

- File an issue if your use case isn't covered.
- PRs welcome for new agents / sandboxes — see [CONTRIBUTING.md](./CONTRIBUTING.md).
- Sketches of failure modes ("here's what went wrong when I tried X") are particularly useful while the gem is pre-alpha.
