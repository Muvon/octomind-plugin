---
name: customize
description: Add or change how Octomind behaves through its own extension points - slash commands (`[[commands]]`, `/run`), `[[layers]]`, `[[agents]]`, `[[prompts]]`, `[[roles]]`, `[[mcp.servers]]`, webhook `[[hooks]]`, `.agents/guardrails.toml` guards and pipes, `.agents/tools` scripts, Octomind skills with activation rules, workflow TOML, taps and agent manifests, scheduled messages, and supervisor, learning, compression, capability or spending settings. Use only when the request names Octomind or the conversation is already configuring Octomind. Not for Claude Code's or Codex's own commands, hooks, skills, plugins or MCP settings, not for changing Octomind's Rust source, and not for installing, signing in or running it (use octomind:cli).
license: Apache-2.0
compatibility: Needs the octomind binary (Linux, macOS, Windows) to validate changes. Shell examples use Bash syntax.
---

# Customize Octomind

Octomind changes behaviour through three places: TOML files in its config directory, files in a project's `.agents/` directory, and tap repositories that package agents to share.
This skill says which one a request needs. Read the matching reference file before writing anything, because field names and required keys are strict.

## Choose the extension

| The user wants | Use | Where it goes | Read |
|---|---|---|---|
| A `/run <name>` command that hands work to another role | `[[commands]]` plus a role | Config directory | `references/commands-layers-agents.md` |
| A reusable prompt sent with `/prompt <name>` | `[[prompts]]` | Config directory | `references/commands-layers-agents.md` |
| A sub-agent the main agent can call as a tool | `[[agents]]` | Config directory | `references/commands-layers-agents.md` |
| A persona with its own prompt, model or tool access | `[[roles]]` | Config directory | `references/roles.md` |
| Tools from an external MCP server | `[[mcp.servers]]` | `mcp-<name>.toml` in the config directory | `references/mcp-servers.md` |
| A project script the agent can call | Executable script | `<workdir>/.agents/tools/<name>` | `references/local-tools.md` |
| Fixed rules that block, rewrite or check work | Guards, pipes, hooks, validators | `<workdir>/.agents/guardrails.toml` | `references/guardrails.md` |
| Instructions that load when relevant | Octomind skill | `<workdir>/.agents/skills/<name>/SKILL.md` | `references/octomind-skills.md` |
| A repeatable multi-step run | Workflow | Any `.toml` file, or a tap's `workflows/` | `references/workflows.md` |
| A session that reacts to an incoming HTTP request | `[[hooks]]` and `--hook` | Config directory | `references/webhooks-and-scheduling.md` |
| A message sent later or on a timer | `schedule` tool or `/schedule` | No file; bound to the session | `references/webhooks-and-scheduling.md` |
| Agents, skills or workflows shared across projects or people | Tap | A git repository or local directory | `references/taps.md` |
| Different limits or behaviour: cost, compression, supervisor, learning, capabilities | Existing settings | Sibling TOML file in the config directory | `references/tuning.md` |

Names that look alike:

- `[[commands]]` creates a `/run` entry. `[[layers]]` does nothing: no release runs it, and it is removed after 0.70.1. Don't use it, and delete old entries.
- `[[hooks]]` in the config directory are webhook listeners. `[[hook]]` in `.agents/guardrails.toml` runs a script after each matching tool result.
- Octomind skills are separate from Claude Code and Codex skills, even when they share a directory such as `.agents/skills/`.

## Rules for every config change

The config directory is `<data-dir>/config/`. The data directory defaults to `~/.local/share/octomind` on macOS and Linux and `%LOCALAPPDATA%/octomind` on Windows, and `OCTOMIND_DATA_DIR` moves it. If `OCTOMIND_CONFIG_PATH` is set, use that file's parent directory.

- Put changes in a new sibling file, such as `commands.toml` or `settings.toml`. Every `*.toml` there is merged. Leave `config.toml` alone, because Octomind migrates it on upgrade.
- Don't delete keys from `config.toml`. Octomind never fills a missing key from its template, so loading fails.
- A sibling table only needs the keys it changes. An array entry is different: a `[[roles]]` or `[[mcp.servers]]` entry with an existing `name` replaces that whole entry, so repeat every field.
- Keep root keys above the first `[table]` header, or they land inside that table.
- Never edit `version`.
- Never put a secret in TOML. Use `{{ENV:KEY}}` where the reference allows it and keep the value in `.env`.
- Don't use `octomind config --model` or other setters for these edits. They rewrite `config.toml` from the merged result and drop any comments.

## Finish every change

1. Run `octomind config --validate` after any TOML edit in the config directory. It is the only command that reports misspelled or unknown keys; a session ignores them silently, except in `[decision]`, where an unknown key makes loading fail.
2. Run the verify step in the reference file. `.agents/guardrails.toml` is not covered by `config --validate`.
3. Tell the user to start a new session, because a running session keeps the settings it loaded.
4. Give the user the path of each file you changed.

## Read more

A reference file may lag a new release. For exact fields, read the shipped template at `https://raw.githubusercontent.com/muvon/octomind/master/config-templates/default.toml` and the guides at `https://raw.githubusercontent.com/muvon/octomind/master/doc/<slug>.md`.

| Topic | Slug |
|---|---|
| Every config key | `reference/03-config-reference` |
| Commands, layers, agents, prompts | `usage/10-commands-and-layers` |
| Roles | `usage/06-roles` |
| MCP servers and runtime tools | `usage/07-mcp-tools` |
| Skills | `usage/15-skills` |
| Local tools | `usage/17-local-tools` |
| Guardrails | `usage/18-guardrails` |
| Workflows | `usage/09-workflows` |
| Webhooks and daemons | `integration/03-daemon-and-hooks` |
| Taps | `integration/04-tap-system` |
| Supervisor, learning, compression | `usage/14-supervisor`, `usage/13-learning`, `usage/08-compression` |
