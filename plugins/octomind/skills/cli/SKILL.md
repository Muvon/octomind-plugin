---
name: cli
description: Install, sign in to, set up and run the open-source Octomind CLI coding agent, including `octomind login` and `octomind configure`, choosing a model or provider, how its config files merge and get rewritten, headless runs (piped, JSONL, named or daemon sessions, sandbox), `octomind evaluate`, and diagnosing a failing Octomind setup. Use only when the request names Octomind or the `octomind` command, or the conversation is already about Octomind. Not for adding Octomind commands, roles, MCP servers, skills, workflows or guardrails (use octomind:customize), not for Octomind Cloud beyond login (use octomind:cloud), and not for other coding agents' own settings.
license: Apache-2.0
compatibility: Needs the octomind binary (Linux, macOS, Windows), network access, and model access via `octomind login` or a provider API key. Shell examples use Bash syntax unless marked as PowerShell.
---

# Octomind CLI

Octomind is an open-source AI coding agent and agent runtime for the terminal: one Rust binary, any model, MCP-native.
Behaviour comes from a TOML config, local `[[roles]]`, and agents fetched from git-based registries called taps.

## Install and verify

```bash
curl -fsSL https://octomind.run/install.sh | bash   # Linux/macOS, installs to ~/.local/bin
brew install muvon/tap/octomind                     # macOS alternative
cargo install octomind                              # from source, needs Rust 1.95+
irm https://raw.githubusercontent.com/muvon/octomind/master/install.ps1 | iex   # Windows PowerShell
octomind --version && octomind config --show
```

- Ask the user before running a piped installer. Scoop also works: `scoop bucket add muvon https://github.com/Muvon/scoop-bucket`, then `scoop install muvon/octomind`.
- Not found after install: `export PATH="$HOME/.local/bin:$PATH"`.
- Pin a release: `curl -fsSL https://octomind.run/install.sh | bash -s -- --version "<tag>"`. Another directory: set `OCTOMIND_INSTALL_DIR`.

## Model access

- **Octomind account (default).** `octomind login` runs a browser device flow and stores `OCTOHUB_API_KEY` in
  `<data-dir>/config/.env`. The shipped config uses `octohub:auto` for the main, supervisor and compression models.
  Headless machine: `octomind login --no-browser`. Replace a stored login: `octomind login --force`. Both block until the user
  confirms in a browser and `--force` mints new credentials: tell the user to run them in their own terminal, never run them yourself.
- **Guided setup.** `octomind configure` lists the hub and every provider, saves any key to `<data-dir>/config/.env`, and changes only the three model names
  in `config.toml`, keeping comments. It needs a terminal on stdin and stdout, so ask the user to run it. Without one it fails: edit the three `name` keys instead.
- **Own provider key.** Export the key (or put it in `.env`), then change `name` in all three profiles: `[model]`,
  `[supervisor.model]`, `[compression.model]`. Changing only `[model]` leaves the other two on OctoHub.
- Model names are always `provider:model`. Common keys: `OPENAI_API_KEY` (`openai`), `ANTHROPIC_API_KEY` (`anthropic`),
  `OPENROUTER_API_KEY` (`openrouter`), `GOOGLE_STUDIO_API_KEY` (`google-studio`), `DEEPSEEK_API_KEY` (`deepseek`).
  `chatgpt:` uses `octomind login chatgpt`. Local servers: `ollama:` (set `OLLAMA_API_URL`) and `local:` (`LOCAL_API_URL`),
  both full endpoint paths. Full prefix table: providers doc.
- `octomind config --api-key` is rejected: keys never go in config files. Load order, later wins: process environment,
  `<data-dir>/config/.env`, project `./.env`.
- One run on another model: `printf '%s\n' 'Reply with OK.' | octomind run -m 'anthropic:claude-sonnet-4-6' --format plain`.
  Precedence: runtime override > role profile > `[taps]` mapping > `[model]`.

## Config files

- Default: `~/.local/share/octomind/config/config.toml` (macOS/Linux), `%LOCALAPPDATA%/octomind/config/config.toml` (Windows).
  Created on first run. Relocate state with `OCTOMIND_DATA_DIR`; pick another main file with `OCTOMIND_CONFIG_PATH`.
- `octomind config --show` (summary, not full TOML), `--validate` (config and model names, not auth), `--upgrade`.
- There is no per-project config file. Octomind reads settings only from the config directory; `<workdir>/.agents/` holds guardrails, tools and skills.
- **Merge:** every `*.toml` in the config directory loads: `config.toml` first, others alphabetically, `mcp-*.toml` last.
  Tables merge recursively, later scalars win, arrays of tables concatenate with the last same-`name` entry winning
  (whole entry replaced, not patched), other arrays are replaced. Put overrides in a sibling file, e.g. `settings.toml`:

  ```toml
  log_level = "debug"
  [model]
  reasoning_effort = "high"
  ```

What happens on load, for every command including `--show` and `--validate`:

- No config directory, or no `*.toml` in it: Octomind writes the full default template.
- `config.toml` has an older `version` than the binary's template: Octomind migrates that file, writes a backup and prints its path to stderr. Runs with
  `-c` skip the migration. Sibling files are never migrated.
- A missing required key is never filled in from the template. Loading fails with `All required fields must be present`. Don't delete keys from `config.toml`;
  override them in a sibling file. Files must collectively supply every required key, so a directory holding only sibling files is not upgraded and can stop loading after a release.
- An unknown or misspelled key is ignored when a session loads. Only `octomind config --validate` reports it, so run that after every edit.
- One-off override, nothing saved: `printf '%s\n' 'Reply with OK.' | octomind run --format plain -c log_level=debug -c supervisor.enabled=false`. The dotted path must
  already exist in the merged config (`Unknown config override path` otherwise), and `-c` is rejected on `config` and `configure`.
- Do not touch `version`. Root keys must sit before the first `[table]` header, or they land inside that table.

What writes where:

| Action | What it changes |
|---|---|
| `octomind config --model`, `--log-level`, `--markdown-theme`, `--markdown-enable`, `--mcp-server` | Rewrites `config.toml` from the whole merged config |
| `octomind config --mcp-providers` | Same rewrite, and REPLACES the whole server list |
| `octomind configure` | The three model names in `config.toml`; keys in `.env` |
| `octomind login` | `OCTOHUB_API_KEY` in `.env`, account session in `auth.json` |
| `/model` in a session | That session only |
| Runtime `mcp` tool, persist action | `<config-dir>/mcp-<name>.toml` |
| `octomind tap`, `untap` | `<data-dir>/taps.toml` |

The full rewrite folds sibling files' values into `config.toml` and loses comments, and a sibling file setting the same key still wins on the next load. Prefer editing a
sibling TOML file.

### Model settings

In `[model]`: `name`, `reasoning_effort` (none, low, medium, high, xhigh, max), `max_tokens`, `max_retries`, `request_timeout_seconds` (one attempt's total deadline, 0 disables),
`stream_timeout_seconds` (inactivity limit, 0 disables), `request_transport` (`websocket` or `http`, only octohub and chatgpt use it). Root `default` is the tag for bare `run`.

Disable telemetry with `telemetry = false`, `OCTOMIND_TELEMETRY=0` or `DO_NOT_TRACK=1`. For cost limits, compression, the supervisor, learning, or any other behaviour change, use `octomind:customize`.

## Tags and taps

The optional argument of `octomind run` is a tag, not a message. A bare name (`assistant`) selects a local `[[roles]]` entry.
A `category:variant` tag (`developer:general`) is resolved from taps. No tag: the `default` key.
The shipped config has roles `assistant`, `task_refiner`, `task_researcher`, `reduce`. `octomind complete run` lists installed/cached
tap tags and configured roles, not every tap agent.

Taps: `octomind tap` (list), `octomind tap myorg/my-agents` (clones `github.com/myorg/octomind-my-agents`),
`octomind tap myorg/my-agents ./local-dir` (symlink), `octomind untap myorg/my-agents`. The built-in `muvon/tap` is
always present, cannot be added or removed, and is searched last. The first tap listing a tag wins.

To create a role, tap or MCP server, use `octomind:customize`. In an interactive session, inspect tools with `/mcp list`, `/mcp full`, `/mcp health`.

## Running

```bash
printf '%s\n' 'Explain the auth module' | octomind run developer:general -m 'openai:gpt-5.6-sol' --format plain
printf '%s\n' 'Add retry logic and update tests' | octomind run developer:general --format plain
printf '%s\n' 'List TODOs in src/' | octomind run developer:general --format jsonl | jq -r 'select(.type=="assistant").content'
printf '%s\n' 'Continue the auth work' | octomind run --name feature-auth --format plain   # create, or resume if it exists
printf '%s\n' 'Carry on' | octomind run --resume-recent --format plain                      # latest session for this directory
```

- `run` has no message argument. Pipe the prompt on stdin. Piped stdin or any `--format` makes the run non-interactive
  (`plain` or `jsonl` only); empty stdin fails with `No input provided via stdin`. Always pipe prompts when running through
  an agent's shell tool. Bare `octomind`, `octomind run [tag]` without a pipe and `--resume` are for the user's own terminal.
- JSONL is an event stream, one object per line with a `type`. Types include `assistant`, `thinking`, `tool_use`, `tool_result`,
  `cost`, `status`, `error`, `mcp_notification`, `skill`, `evolution`, `injected`. Take the final answer from the last
  `assistant` event; also check `error` events and the exit code.
- Exit codes: `0` ok, `1` failed, `2` bad arguments, `75` stopped at a spending threshold (session saved, resumable), `130` force-stopped (second Ctrl+C or SIGTERM).
- `--sandbox` (or `sandbox = true`) restricts writes to the working directory for `run`, `server`, `acp` (Landlock on Linux,
  Seatbelt on macOS; Windows only warns).
- `--schema todos.schema.json` constrains replies to a JSON Schema object file; the model must support structured output;
  pass it again on resume. Best with `--format jsonl`.
- Resuming without a tag keeps the saved role. Bare `--resume` opens a picker and needs a TTY; scripts pass a name.

**Long or iterative work: daemon + send.** Run the daemon as a background shell command with a non-empty piped prompt:

```bash
printf '%s\n' 'Implement the OAuth refresh flow' | octomind run developer:general --name oauth --daemon --format jsonl >oauth.jsonl 2>oauth.log &
oauth_pid=$!   # keep the pid for stopping it
octomind send --name oauth "Now add error handling for token expiry"   # same user and host
```

`send` only confirms the message was queued; read `oauth.jsonl` for the reply. It fails if the session is not running.
`--daemon` does not detach; stop it with `kill -TERM "$oauth_pid"` from the same shell (exit 130).

**Workflows.** `octomind workflow my.toml` or a tap workflow name; no argument lists them. Input comes from stdin. `--dry-run` prints the plan to stdout without reading
stdin or running steps. A real run writes progress to stderr, and stdout stays empty unless you pass `--format jsonl`. To write one, use `octomind:customize`.

## Other commands

- `octomind evaluate [FILE] [-m provider:model]` asks the evaluation model typed questions about a JSON state and prints the answers as JSON. Input comes from FILE or stdin:
  `{"state": ..., "questions": {"id": {"type": "noul", "instructions": "..."}}}`. It needs `-m` or `[decision].model`.
- `octomind vars [--preview | --expand]` shows the placeholders and their values.
- `octomind server [TAG] [--host H] [-p PORT] [--sandbox] [--allow-origin URL]` starts the WebSocket server, by default on `127.0.0.1:8080`. It has no TLS and no
  authentication unless `OCTOMIND_SERVER_TOKEN` is set.
- `octomind acp [TAG] [-n NAME] [-r SESSION] [--resume-recent] [-m MODEL] [--sandbox] [--hook NAME]` runs an ACP agent over stdio for editors.
- `octomind completion <bash|zsh|fish|powershell|elvish>` prints a shell completion script.

## Pitfalls

- `octomind run 'fix it'` treats the text as a tag. Always pipe.
- `--format` on a terminal with nothing piped errors unless you also pass `--daemon`.
- `.env` files override exported variables, and the project `./.env` beats the user one. An empty assignment still masks
  an earlier value. `OCTOMIND_CONFIG_PATH` does not move the user `.env`.
- A sibling-file entry whose `name` matches an existing `[[roles]]` or `[[mcp.servers]]` replaces that entry entirely; repeat every field.
- A stray backup `*.toml` in the config directory is merged too.
- Bare role names must match a local `[[roles]]` entry; unknown names are rejected. Don't name a local role like a tap tag.
- Tap agents fetch their manifest and dependencies on first use (needs network). Manifests are cached; a tap edit may stay
  hidden until `cache_ttl_hours` passes, or delete `<data-dir>/agents/<category>/<variant>.toml`.
- A tap manifest `{{ENV:KEY}}` with the variable unset prompts, then appends the answer to `./.env` in the current directory.
  Supply keys beforehand. MCP server questions (elicitation) are declined in piped and `--format` runs.
- The `google:` prefix is not accepted; use `google-vertex:` or `google-studio:`. `openai:` uses the Responses API;
  use `local:` for Chat Completions endpoints.
- `/role` can overwrite an earlier `/model` choice in a session.
- Never put API keys in TOML, never commit `.env`.

## Smoke test

```bash
octomind --version && octomind config --validate
printf '%s\n' 'Reply with OK.' | octomind run --format plain
```

Success: the validate step passes and the reply arrives without a provider or credential error. On failure run
`octomind config --show`, then re-check the provider key and the three model names. For account auth, ask the user to run
`octomind login` (or `--force`) in their own terminal; never run it yourself.

## Read more

Pages: `https://octomind.run/docs/<slug>`. Exact TOML keys: `https://raw.githubusercontent.com/muvon/octomind/master/doc/<slug>.md`.

| Question | Slug |
|---|---|
| Install, shell completions | `usage/01-installation` |
| First session, resume, piping | `usage/02-quickstart` |
| Config files, merge, model purposes | `usage/03-configuration` |
| Providers, keys, local models | `usage/04-providers` |
| Sessions, daemon, resume | `usage/05-sessions` |
| Roles, tool permissions | `usage/06-roles` |
| MCP servers, runtime tools | `usage/07-mcp-tools` |
| Workflows | `usage/09-workflows` |
| JSONL events, `--schema` | `usage/11-structured-output` |
| Skills, local tools, guardrails | `usage/15-skills`, `usage/17-local-tools`, `usage/18-guardrails` |
| Taps, capabilities, manifests | `integration/04-tap-system` |
| Daemon, `send`, webhooks | `integration/03-daemon-and-hooks` |
| Every flag and exit code | `reference/01-cli-reference` |
| Every config key | `reference/03-config-reference` |
| Environment variables | `reference/04-environment-variables` |
| Session slash commands | `reference/02-session-commands` |
| Errors, sandbox, logs | `troubleshooting/01-common-issues` |


## Local inference lifecycle

Hindsight uses an automatically started local worker shared by Octomind processes in the
same runtime directory. It may remain alive for up to 15 idle minutes. Use normal Octomind
commands; `hindsight-worker` is an internal process entry point, not an agent command.
CoreML compiled-model cache reuse is macOS-only; Linux and Windows keep their existing
execution-provider selection.
