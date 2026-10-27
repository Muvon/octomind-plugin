---
name: cli
description: Install, configure and operate the open-source Octomind CLI coding agent. Use when the user asks to install or sign in to Octomind, edit its config.toml, pick a model or provider, add MCP servers, roles or taps, run Octomind agents headless (piped, JSONL, named or daemon sessions, sandbox), or troubleshoot a failing Octomind setup. Does not cover Octomind Cloud beyond `octomind login`.
license: Apache-2.0
compatibility: Needs the octomind binary (Linux, macOS, Windows), network access, and model access via `octomind login` or a provider API key. Commands are written for Claude Code's Bash tool.
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
- **Merge:** every `*.toml` in the config directory loads: `config.toml` first, others alphabetically, `mcp-*.toml` last.
  Tables merge recursively, later scalars win, arrays of tables concatenate with the last same-`name` entry winning
  (whole entry replaced, not patched), other arrays are replaced. Put overrides in a sibling file, e.g. `settings.toml`:

  ```toml
  log_level = "debug"
  [model]
  reasoning_effort = "high"
  ```
- Existing configs are strict, not sparse overlays: required keys must exist somewhere in the merged files.
- One-off override, nothing saved: `printf '%s\n' 'Reply with OK.' | octomind run --format plain -c log_level=debug -c supervisor.enabled=false`.
- Do not touch `version`. Root keys must sit before the first `[table]` header, or they land inside that table.
- Setters (`octomind config --model provider:model`, `--log-level none|info|debug`, `--markdown-theme`,
  `--mcp-server "name,type=stdio,command=octofs"`) rewrite `config.toml` from the whole merged config: sibling files' values
  get folded in, comments are lost, and a sibling file setting the same key still wins on the next load. Prefer editing a
  sibling TOML file. `--mcp-providers` REPLACES the whole server list.

### Keys edited most

| Where | Keys (default values) |
|---|---|
| root | `default = "assistant:concierge"` (tag for bare `run`), `sandbox = false`, `telemetry = true`, `log_level = "info"`, `auto_capabilities = true`, `max_session_tokens_threshold = 200000`, `max_session_spending_threshold = 0.0`, `max_request_spending_threshold = 0.0` (0 = no limit) |
| `[model]` | `name`, `reasoning_effort = "medium"` (none, low, medium, high, xhigh, max), `max_tokens = 32768`, `temperature = 0.3`, `top_p = 0.7`, `top_k = 20`, `max_retries = 1`, `request_timeout_seconds = 300` |
| `[supervisor]` | `enabled = true`; `[supervisor.model]` shared by gate, planning, learning; `[supervisor.gate]`, `[supervisor.plan]`, `[supervisor.learning]` each `enabled` |
| `[compression]` | `threshold = 70000` (0 disables auto); `[compression.model]` |
| `[taps]` | empty by default; each entry maps a tap tag to a model name, e.g. `"developer:general" = "ollama:glm-5.3"` (example, not a default) |
| `[registry]` | `cache_ttl_hours = 24` (tap manifest cache; built-in default, absent from the shipped file) |
| `[skills]` | `auto_activation = true`, `auto_validation = false` |

Disable telemetry with `telemetry = false`, `OCTOMIND_TELEMETRY=0` or `DO_NOT_TRACK=1`.

## Roles, tap agents, tags

The optional argument of `octomind run` is a tag, not a message. A bare name (`assistant`) selects a local `[[roles]]` entry.
A `category:variant` tag (`developer:general`) is resolved from taps. No tag: the `default` key.
The shipped config has roles `assistant`, `task_refiner`, `task_researcher`, `reduce`. `octomind complete run` lists installed/cached
tap tags and configured roles, not every tap agent.

Custom role: save as `roles.toml` beside `config.toml`, then `octomind config --validate`; try it with
`printf '%s\n' 'Reply with OK.' | octomind run helper --format plain`.
`system` and `welcome` are required; if `[roles.mcp]` exists it needs both lists.

```toml
[[roles]]
name = "helper"
system = "You are helpful and knowledgeable assistant. Working directory: {{CWD}}"
welcome = "Hello! Working in {{CWD}} (Role: {{ROLE}})"
[roles.model]
temperature = 0.3
[roles.mcp]
server_refs = ["core"]
allowed_tools = ["core:recall"]
```
`core` has no filesystem tools (they come from an external server such as `octofs`). For file access, register that server
(see MCP servers) and add its name to `server_refs` and its tools to `allowed_tools`.

Taps: `octomind tap` (list), `octomind tap myorg/my-agents` (clones `github.com/myorg/octomind-my-agents`),
`octomind tap myorg/my-agents ./local-dir` (symlink), `octomind untap myorg/my-agents`,
`octomind tap init acme/team --agent team:assistant` (scaffold, validate, register locally). The built-in `muvon/tap` is
always present, cannot be added or removed, and is searched last. The first tap listing a tag wins.

## MCP servers

Four builtin servers (`core`, `orchestration`, `runtime`, `agent`) are declared by default. `filesystem` is not builtin; it is
an external server such as `octofs`. Add external ones as `http` or `stdio` in a file named `mcp-<name>.toml`:

```toml
[[mcp.servers]]
name = "project_tools"
type = "stdio"
command = "{{ENV:PROJECT_MCP_COMMAND}}"
args = []
timeout_seconds = 30
tools = []
auto_bind = ["developer:general"]

[[mcp.servers]]
name = "custom_api"
type = "http"
url = "https://api.example.com/mcp"
headers = { Authorization = "Bearer {{ENV:CUSTOM_API_TOKEN}}" }
timeout_seconds = 30
tools = []
```

Registering a server does not enable it: use `auto_bind` (exact role string; `developer` does not match
`developer:general`) or list it in a role's `server_refs` and `allowed_tools`. `tools = []` means all tools. `timeout_seconds` is
1-3600. A server whose `{{ENV:KEY}}` is unset cannot connect. In an interactive session, check `/mcp list`, `/mcp full`, `/mcp health`.
Project scripts in `<workdir>/.agents/tools/` become tools automatically (`local` server).

## Running

```bash
printf '%s\n' 'Explain the auth module' | octomind run developer:general -m 'openai:gpt-5.6-sol' --format plain
printf '%s\n' 'Add retry logic and update tests' | octomind run developer:general --format plain
printf '%s\n' 'List TODOs in src/' | octomind run developer:general --format jsonl | jq -r 'select(.type=="assistant").content'
printf '%s\n' 'Continue the auth work' | octomind run --name feature-auth --format plain   # create, or resume if it exists
printf '%s\n' 'Carry on' | octomind run --resume-recent --format plain                      # latest session for this directory
```

- `run` has no message argument. Pipe the prompt on stdin. Piped stdin or any `--format` makes the run non-interactive
  (`plain` or `jsonl` only); empty stdin fails with `No input provided via stdin`. Claude's Bash tool has no TTY, so every run
  must be piped. Bare `octomind`, `octomind run [tag]` without a pipe and `--resume` are for the user's own terminal.
- JSONL is an event stream, one object per line with a `type`. Types include `assistant`, `thinking`, `tool_use`, `tool_result`,
  `cost`, `status`, `error`, `mcp_notification`, `skill`, `evolution`, `injected`. Take the final answer from the last
  `assistant` event; also check `error` events and the exit code.
- Exit codes: `0` ok, `1` failed, `2` bad arguments, `75` stopped at a spending threshold (session saved, resumable).
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

**Workflows.** `octomind workflow my.toml` (or a tap workflow name; no argument lists them) runs multi-step TOML flows: sequential,
`parallel`, `loop` with `exit_when`, `conditional`, and graph routing. Top-level `name` is required. Only sequential steps carry
`role` and `prompt` (`{{input}}`, `{{<step name>}}`); `parallel = true`, `loop = true` and `conditional = true` steps hold a `name`
plus nested `[[steps.run]]` sequential entries (complete example: `usage/09-workflows`).
Input comes from stdin; `--dry-run` prints the plan; stdout is empty unless `--format jsonl`.

## Pitfalls

- `octomind run 'fix it'` treats the text as a tag. Always pipe.
- `--format` on a terminal with nothing piped errors unless you also pass `--daemon`.
- `.env` files override exported variables, and the project `./.env` beats the user one. An empty assignment still masks
  an earlier value. `OCTOMIND_CONFIG_PATH` does not move the user `.env`.
- A `[[roles]]` or `[[mcp.servers]]` with an existing `name` replaces the earlier entry entirely; repeat every field.
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
