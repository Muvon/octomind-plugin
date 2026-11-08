# Roles

Lets the user define their own agent role (system prompt, model settings, tool grants), pin a model to a tap agent, and use prompt placeholders.

A role is a `[[roles]]` entry in config, addressed by a plain name such as `assistant`. A tap agent is a manifest published in a tap, addressed by a `category:variant` tag such as `developer:general`. Any tag containing `:` is resolved through the tap system, not through `[[roles]]`.

Config lives in `<data-dir>/config/`. The default data dir is `~/.local/share/octomind`. Put new snippets in a sibling file such as `<data-dir>/config/roles.toml`. Do not rewrite `config.toml`.

## Custom role (`[[roles]]`)

Where it goes: `<data-dir>/config/roles.toml` (any `*.toml` beside `config.toml` is merged; `config.toml` loads first, the rest alphabetically).

```toml
[[roles]]
name = "helper"
system = "You are a helper. Working directory: {{CWD}}"
welcome = "Ready in {{CWD}} (Role: {{ROLE}})"

[roles.model]
temperature = 0.3

[roles.mcp]
server_refs = ["core"]
allowed_tools = ["core:*"]
```

Required fields and allowed values:
- `name`, `system`, `welcome` are required strings. `welcome = ""` shows no banner. A missing field fails parsing.
- `[roles.model]` is optional. Omitted fields inherit from `[model]`.
- `[roles.mcp]` is optional. If present, give both `server_refs` (server names) and `allowed_tools`. If absent, both lists are empty.
- `allowed_tools` patterns: `"server:*"`, `"server:prefix_*"`, `"server:tool"`. An empty list means no role-level filter.
- A `server_refs` name that is not a registered server is silently dropped.

How it activates: config is read when octomind starts, so start a new session. Then `octomind run helper`. Inside a session, `/role helper` switches to it, rebuilds the system prompt and MCP tools, and saves the role to the session.

How to verify: `octomind config --validate`, then `octomind run helper` and `/mcp list` to see the tools the role really has.

Pitfalls:
- Use a plain name. A local role named like a tap tag (`developer:general`) can block tap resolution.
- Two same-named entries across files: the last whole entry wins, it is not a field patch. Repeat every required field. This is also how to replace a shipped role (`assistant`, `task_refiner`, `task_researcher`, `reduce`): redefine it in a later file.
- An unknown plain role name is rejected with `Role '<name>' not found. Available roles: ...`.
- A tap manifest role with the same name as a config role is skipped; the config role wins.
- Empty `server_refs` does not mean no tools: `auto_bind` servers, interactive `schedule` and `monitor`, automatic capabilities and project `.agents/tools/` can still add tools.

## Role model profile (`[roles.model]`)

Where it goes: inside a `[[roles]]` entry, in the same file.

```toml
[roles.model]
name = "openai:gpt-5.6-luna"
reasoning_effort = "high"
```

Required fields and allowed values: any subset of `name`, `reasoning_effort`, `max_tokens`, `temperature`, `top_p`, `top_k`, `max_retries`, `retry_timeout`, `request_timeout_seconds`. Ranges: `temperature` 0.0 to 2.0, `top_p` 0.0 to 1.0, `top_k` 0 to 1000. Model names are `provider:model`.

Model precedence, highest first: `--model` or runtime override, the role's `[roles.model]`, the `[taps]` name mapping, then `[model]`.

How it activates: new session. `/role <name>` re-applies the role's profile and can replace an earlier `/model` or `/effort` choice.

How to verify: `octomind config --validate` (out-of-range values fail with `roles.<name>.temperature must be between 0.0 and 2.0`).

Pitfalls: a model from another provider needs that provider's credential; validation does not check it.

## Tap model mapping (`[taps]`)

Where it goes: `<data-dir>/config/taps-models.toml`.

```toml
[taps]
"developer:general" = "openai:gpt-5.6-luna"
```

Required fields and allowed values: keys are exact tap tags, values are `provider:model`. It changes only the model name; other parameters come from `[model]`.

How it activates: resolving the tag at `octomind run <tag>` (new session).

How to verify: `octomind config --validate`.

Pitfalls: it applies only to tags with `:`. A plain role ignores it. A model set in the agent role's own `[roles.model]` beats it.

## Placeholders

Where it goes: inside `system` and `welcome` of a role.

```toml
system = "Project: {{CWD}}, role {{ROLE}}, today {{DATE}}"
```

Allowed values: `{{DATE}}`, `{{SHELL}}`, `{{OS}}`, `{{BINARIES}}`, `{{CWD}}`, `{{HOME}}`, `{{SYSTEM}}`, `{{CONTEXT}}`, `{{GIT_STATUS}}`, `{{GIT_TREE}}`, `{{README}}`, `{{ROLE}}`. `{{ROLE}}` is the active role of that session.

How it activates: expanded when the session builds its prompt.

How to verify: `octomind vars` lists them; `octomind vars --preview` or `--expand` shows values for the current directory.

Pitfalls: `{{ROLE}}` is not in the `octomind vars` value dump, only in its list.
