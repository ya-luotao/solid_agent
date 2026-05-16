# Solid Agent

A Ruby gem that runs a coding agent in a sandbox with one API.

```ruby
SolidAgent.run('add tests for the auth module', agent: :claude, sandbox: :e2b)
```

That's the pitch. Solid Agent composes three existing Ruby gems so you don't have to:

- [`claude-agent-sdk`](https://github.com/ya-luotao/claude-agent-sdk-ruby) — Claude Code SDK for Ruby
- [`codex-rb`](https://github.com/ya-luotao/codex-rb) — OpenAI Codex SDK for Ruby
- [`e2b`](https://github.com/ya-luotao/e2b-ruby) — E2B Firecracker sandbox SDK for Ruby

Pick an agent. Pick a sandbox. Solid Agent handles the wiring.

## Status

Pre-alpha. The README is ahead of the code. Issues and PRs welcome.

## Install

```ruby
# Gemfile
gem 'solid_agent'
```

```sh
bundle install
```

## Examples

### One-shot

```ruby
require 'solid_agent'

result = SolidAgent.run('Summarize this repo')
puts result.text
puts result.cost_usd
```

### Pick an agent

```ruby
SolidAgent.run('Write a Fibonacci function', agent: :codex)
SolidAgent.run('Audit auth.rb for issues',   agent: :claude)
```

### Pick a sandbox

```ruby
SolidAgent.run('Run the tests', sandbox: :local)
SolidAgent.run('Run the tests', sandbox: :e2b, sandbox_opts: { template: 'ubuntu-22-04' })
```

### Stream events

```ruby
SolidAgent.run('Refactor billing.rb') do |event|
  case event
  when SolidAgent::TextEvent   then print event.text
  when SolidAgent::ToolEvent   then puts "\n[#{event.tool_name}]"
  when SolidAgent::ResultEvent then puts "\nDone (#{event.duration_ms}ms, $#{event.cost_usd})"
  end
end
```

### Interactive session

```ruby
session = SolidAgent.session(agent: :claude, sandbox: :e2b)
session.start('Read the codebase')
session.prompt('Now add tests for the auth flow')
session.prompt('Now run them')
session.close
```

### Manifest (seed the sandbox)

```ruby
SolidAgent.run(
  'Fix the failing test in app/models/user.rb',
  agent: :claude,
  sandbox: :e2b,
  manifest: {
    repos: [{ url: 'git@github.com:org/app.git', ref: 'main', dest: 'app' }],
    env:   { 'GITHUB_TOKEN' => ENV['GH_TOKEN'] }
  }
)
```

## Supported

**Agents**

| Symbol | Backed by |
|---|---|
| `:claude` | [`claude-agent-sdk`](https://github.com/ya-luotao/claude-agent-sdk-ruby) gem |
| `:codex`  | [`codex-rb`](https://github.com/ya-luotao/codex-rb) gem |

**Sandboxes**

| Symbol | Backed by |
|---|---|
| `:local` | local subprocess |
| `:e2b`   | [`e2b`](https://github.com/ya-luotao/e2b-ruby) gem (E2B Firecracker microVMs) |

More on the [roadmap](./ROADMAP.md).

## Why this gem

Each of the three underlying SDKs is good on its own. Composing them by hand means re-writing the same glue for every project:

- Translating between Claude's stream-JSON messages and Codex's JSON-RPC notifications.
- Bridging the agent's CLI stdio through E2B's command RPC.
- Normalising events, tool calls, costs, and errors across agents.
- Handling sandbox lifecycle (provision, pause, resume, snapshot).

Solid Agent puts that glue in one place. The public surface is Ruby objects — there's no protocol to learn beyond the gem's documented classes.

## Design notes

- **Agent and Sandbox are separate.** Each is a small Ruby class with a known method set. Pair any agent with any sandbox; capability negotiation refuses incompatible pairs at session creation.
- **Events are normalised.** Both Claude's and Codex's native event vocabularies are mapped into one Ruby event hierarchy (`TextEvent`, `ToolEvent`, `ToolResultEvent`, `ResultEvent`, `ErrorEvent`).
- **Transports are reused, not re-implemented.** `claude-agent-sdk` already exposes a pluggable `Transport`; the `:e2b` sandbox uses it to bridge stdio through E2B's command RPC. Same idea for `codex-rb`'s `AppServerClient`.
- **No protocol invention.** Solid Agent doesn't define a wire format — it's a Ruby gem with Ruby idioms. The underlying SDKs handle each agent's actual wire protocol.

## Adding your own agent or sandbox

```ruby
class SolidAgent::Agents::MyAgent < SolidAgent::Agent
  def start(prompt, sandbox:, options: {}, &on_event)
    # call your backend; yield SolidAgent::Event objects
  end
end

class SolidAgent::Sandboxes::MySandbox < SolidAgent::Sandbox
  def provision(manifest); end
  def run_command(command, **opts); end
  def read_file(path); end
  def write_file(path, content); end
  def finalize; end
end
```

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the full contract.

## License

MIT — see [LICENSE](./LICENSE).
