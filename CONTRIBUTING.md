# Contributing

Thanks for being here. Solid Agent is small on purpose; most contributions are either an adapter or a bug fix.

## Adding an agent

Subclass `SolidAgent::Agent`:

```ruby
class SolidAgent::Agents::MyAgent < SolidAgent::Agent
  def agent_kind
    :my_agent
  end

  def capabilities
    { thinking: false, structured_output: false, image_input: false, interrupt: true }
  end

  # Called once per session. Translate your backend's events into SolidAgent events.
  def start(prompt, sandbox:, options: {}, &on_event)
    # Use sandbox.run_command(...) for shell commands.
    # Use sandbox.read_file(...) / sandbox.write_file(...) for file I/O.
    # Yield SolidAgent event objects to on_event.
  end

  def prompt(text)
    # Send a follow-up message in the same session.
  end

  def interrupt
    # Cancel the current turn, if your backend supports it.
  end

  def close
    # Clean up.
  end
end
```

### Events to emit

| Event | Fields |
|---|---|
| `SolidAgent::TextEvent` | `text` |
| `SolidAgent::ThinkingEvent` | `text` |
| `SolidAgent::ToolEvent` | `tool_name`, `tool_use_id`, `input` |
| `SolidAgent::ToolResultEvent` | `tool_use_id`, `output`, `is_error` |
| `SolidAgent::ResultEvent` | `text`, `duration_ms`, `cost_usd`, `usage`, `stop_reason` |
| `SolidAgent::ErrorEvent` | `error_type`, `message` |

If your backend emits something that doesn't fit, propose a new event type in an issue first.

## Adding a sandbox

Subclass `SolidAgent::Sandbox`:

```ruby
class SolidAgent::Sandboxes::MySandbox < SolidAgent::Sandbox
  def kind
    :my_sandbox
  end

  def capabilities
    { fs: true, run: true, snapshot: false, pause: false, port_forward: false }
  end

  # Set up the sandbox: clone repos, seed files, export env vars.
  def provision(manifest)
  end

  # Run a command. Return { stdout:, stderr:, exit_code: } or raise on failure.
  def run_command(command, cwd: nil, env: {}, timeout: nil)
  end

  def read_file(path)
  end

  def write_file(path, content)
  end

  def cwd
    '/workspace'
  end

  # Optional, gated by capabilities.
  def snapshot(label = nil); end
  def pause; end
  def resume(timeout: 30); end
  def host_for_port(port); end

  def finalize
    # Tear down.
  end
end
```

### Capability negotiation

When a session opens, Solid Agent intersects the Agent's required capabilities with the Sandbox's advertised capabilities. Mismatches fail fast with a clear error message. Advertise honestly; do not stub.

## Code style

- Ruby 3.2+.
- RuboCop config matches the underlying SDKs (`claude-agent-sdk-ruby`'s in particular).
- Tests use RSpec, `disable_monkey_patching!` enabled, `expect` syntax only.
- Public methods have YARD doc comments.

## Filing issues

- **Bugs** — include Ruby version, gem versions, and the smallest reproducer you can manage.
- **Feature requests** — describe the use case first, then the proposed API. Use cases beat API sketches.
- **New agent or sandbox** — link to the upstream SDK or CLI. If a Ruby SDK doesn't exist yet for the agent, say so; we'll discuss whether a shell-out adapter makes sense.

## Code of conduct

Be kind. Assume good faith. Don't take "no" personally; the gem stays small on purpose.
