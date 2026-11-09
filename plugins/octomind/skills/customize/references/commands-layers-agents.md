# Commands, agents, and prompt templates

Add slash commands, sub-agent tools, and prompt shortcuts to Octomind sessions by writing small TOML files in the config directory.

All four extensions are TOML tables in a sibling file inside `<data-dir>/config/`. `<data-dir>` is `$OCTOMIND_DATA_DIR`, or `~/.local/share/octomind` by default. Every `*.toml` in that directory is merged into the config, so never rewrite `config.toml`. Entries with the same `name` replace each other as a whole; the later file wins.

## Commands (`[[commands]]`, run with `/run`)

Where it goes: `<data-dir>/config/commands.toml`.

```toml
[[roles]]
name = "analysis"
system = "Analyze the supplied text. List assumptions, risks, and next steps."
welcome = ""

[roles.mcp]
server_refs = []
allowed_tools = []

[[commands]]
name = "analysis"
description = "Analyze the previous answer"
command = "octomind acp analysis"
input_mode = "last"
output_mode = "append"
output_role = "assistant"
```

Required fields: `name`, `description`, `command`, `input_mode`, `output_mode`, `output_role`. Leaving one out is a config load error (`missing field`). `workdir` is optional.
- `command`: `octomind acp <role>`. It is split on spaces and not run through a shell. The role must exist in `[[roles]]` or come from a tap.
- `input_mode`: `last` (last assistant reply, plus any text you typed), `all` (whole transcript), `summary` (a shortened summary of the conversation).
- `output_mode`: `none` (print only), `append` (add to the conversation), `last` (add only the last output), `replace` (keep the first system message, rebuild the welcome message, drop the old conversation, insert the output), `restart` (clear everything, keep only the last output).
- `output_role`: `assistant` or `user`.

How it activates: configs load at session start, so start a new session after editing. `/run` lists the loaded commands; `/run <name> [text]` runs one. Text after the name is the input; without it, the last real user message is used. A session or request spending limit can refuse the run.

How to verify: `octomind config --validate` checks fields and values. Then, in a session, `/run` must list the name. `/run <name> <text>` calls a model and costs money.

Pitfalls:
- `config --validate` does not check that the role in `command` exists. A missing role fails only when you run the command.
- `replace` and `restart` delete the conversation. Use `append` to keep it.
- Naming a command `reduce` replaces the shipped context-compression command.

## Old `[[layers]]` entries

Do not write `[[layers]]`. No release runs it, and the key is removed after 0.70.1. Delete old entries: once the key is gone, `octomind config --validate` reports them as unknown and exits 1. To run a stage, write a `[[commands]]` entry (user-triggered) or an `[[agents]]` entry (model-triggered). Multi-step pipelines belong to `octomind workflow`.

## Agents (`[[agents]]`)

Where it goes: `<data-dir>/config/agents.toml`.

```toml
[[roles]]
name = "analyst"
system = "Analyze the supplied text. Report assumptions and risks."
welcome = ""

[roles.mcp]
server_refs = []
allowed_tools = []

[[agents]]
name = "analyst"
description = "Analyze supplied text and report assumptions and risks."
command = "octomind acp analyst"
workdir = "."

[[roles]]
name = "delegator"
system = "Delegate analysis to agent_analyst and explain its findings."
welcome = "Ready."

[roles.mcp]
server_refs = ["agent"]
allowed_tools = ["agent:*"]
```

Required fields: `name`, `description`, `command`; `workdir` is optional. There are no input or output modes. Each agent becomes a model-callable tool `agent_<name>` with arguments `task` (required text) and `async` (boolean, default false). The agent starts with no memory of the conversation, so `task` must carry all context. An async result comes back later as a user message, `[Async agent '<name>' completed]`. The number of async jobs is capped at the CPU count.

The parent role must list the `agent` server in `server_refs` and allow it in `allowed_tools`, as `delegator` does above.

Dynamic agents: the `runtime` server's `agent` tool creates temporary agents with no config edit. Call it with `add` (`name` and `system` required; `description`, `model`, `server_refs`, `allowed_tools`, `workdir` optional), then `enable`, then call `agent_<name>`. Other actions are `list`, `disable`, `remove`. They run in the same process and are gone when the session ends. The parent role needs `runtime` in `server_refs`.

How it activates: start a new session with a parent role that has the `agent` server (run `octomind run delegator`). Dynamic agents work in the running session.

How to verify: `octomind config --validate`; then in the session `/mcp full` must show `agent_analyst`.

Pitfalls:
- The shipped `context_gatherer` agent runs `octomind acp context_gatherer`, but no role of that name ships. Define the role or change the command before using it.
- A parent role without `agent` in `server_refs` never sees the tools. A name used by both a config agent and a dynamic agent is refused as ambiguous.

## Prompt templates (`[[prompts]]`, sent with `/prompt`)

Where it goes: `<data-dir>/config/prompts.toml`.

```toml
[[prompts]]
name = "changelog"
description = "Draft a changelog entry"
prompt = "Write a changelog entry for the changes discussed above."
```

Required fields: `name` and `prompt`. `description` is optional and only appears in the `/prompt` list.

How it activates: start a new session after editing. `/prompt` lists templates; `/prompt <name>` queues the text as a normal user message, and the model answers it on the next turn.

How to verify: `octomind config --validate`, then `/prompt` in a session. `/prompt <name>` calls a model.

Pitfalls: the text is sent exactly as written, so placeholders such as `{role}` are not replaced. Reusing a shipped name (`review`, `explain`, `optimize`, `test`, `debug`) replaces the shipped prompt.
