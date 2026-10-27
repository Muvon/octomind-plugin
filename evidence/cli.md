# Evidence for skills/cli/SKILL.md

`D` = `doc/` and `S` = `src/` in https://github.com/muvon/octomind (branch `master`); line numbers as of 2026-10-27.
Docs branch: `master` (raw files resolve at https://raw.githubusercontent.com/muvon/octomind/master/doc/..., HTTP 200 on 2026-10-27).

claim | source | verbatim quote

## Frontmatter / framing
Octomind is open-source AI coding agent, one Rust binary, any model, MCP-native | D/README.md:3 | "Octomind is an open-source AI coding agent and agent runtime for the terminal: one Rust binary, any model, MCP-native."
Behaviour from TOML config, roles, taps | D/integration/04-tap-system.md:3 | "Use taps to install and author reusable Octomind agents, skills, capabilities, workflows, and plugins."
Taps are git-based registries | D/integration/04-tap-system.md:86 | "A tap is a Git repository (or local directory) containing:"
License Apache-2.0 (skill license, from brief and sibling skill) | octomind `skills/octomind/SKILL.md:4` | "license: Apache-2.0"
Compatibility Linux, macOS, Windows; needs login or provider key | octomind `skills/octomind/SKILL.md:5` | "Requires the octomind CLI (Linux, macOS, Windows) and model access via `octomind login` or a provider API key"
Frontmatter fields name/description/license/compatibility valid; compatibility <= 500 chars | https://code.claude.com/docs/en/skills | "Accepts a string of up to 500 characters."
Network needed (tap fetch, provider) | D/usage/02-quickstart.md:123 | "Tap resolution fetches agent definitions and may run dependency setup; model"

## Install
Unix installer URL, installs to ~/.local/bin | D/usage/01-installation.md:11,14-15 | "curl -fsSL https://octomind.run/install.sh | bash" ; "installs `octomind` in `~/.local/bin` by default"
Windows installer command | D/usage/01-installation.md:21 | "irm https://raw.githubusercontent.com/muvon/octomind/master/install.ps1 | iex"
Homebrew command | D/usage/01-installation.md:83 | "brew install muvon/tap/octomind"
cargo install, needs Rust 1.95+ | D/usage/01-installation.md:132,135 | "cargo install octomind" ; "requires Rust 1.95 or newer"
Scoop commands | D/usage/01-installation.md:92-93 | "scoop bucket add muvon https://github.com/Muvon/scoop-bucket"
Verify with --version and config --show | D/usage/01-installation.md:204-205 | "octomind --version" / "octomind config --show"
PATH fix | D/usage/01-installation.md:213 | "export PATH=\"$HOME/.local/bin:$PATH\""
Pin a release with --version | D/usage/01-installation.md:159 | "curl -fsSL https://octomind.run/install.sh | bash -s -- --version \"$OCTOMIND_VERSION\""
OCTOMIND_INSTALL_DIR chooses destination | D/usage/01-installation.md:66 | "Set `OCTOMIND_INSTALL_DIR` to choose another destination:"

## Model access
login is browser device flow, stores OCTOHUB_API_KEY in <data-dir>/config/.env | D/usage/04-providers.md:15-16 | "`octomind login` starts a device-authorization flow, displays a short code, and opens a browser approval page."
(same) key location | D/usage/01-installation.md:38-39 | "The completed login stores `OCTOHUB_API_KEY` in `<data-dir>/config/.env`"
Default config uses octohub:auto for main, supervisor, compression | D/usage/01-installation.md:47-48 | "uses `octohub:auto` for its main, supervisor, and compression model purposes."
login --no-browser / --force | D/reference/01-cli-reference.md:79 | "octomind login --force --no-browser"
login blocks until browser confirmation; --force mints new credentials (user runs it, not Claude) | S/commands/login.rs:72-124 | "Confirm the code in your browser to finish signing in." ; "waiting…" ; "if !args.force {"
Replace stored login with --force | D/usage/04-providers.md:205-208 | "force a new login to replace the machine's stored gateway credential:"
BYOK: change name in all three profiles; main-only leaves others on OctoHub | D/usage/04-providers.md:74-75 | "change `name` in all three existing profiles while keeping their other settings. A main-model override alone leaves the shipped internal profiles on OctoHub"
Model names must be provider:model | D/usage/03-configuration.md:121 | "Model names must use `provider:model`."
OPENAI/ANTHROPIC/OPENROUTER/GOOGLE_STUDIO/DEEPSEEK keys and prefixes | D/usage/04-providers.md:218-222,225 | "| OpenAI | `openai` | `OPENAI_API_KEY` |" (also anthropic, openrouter, google-studio `GOOGLE_STUDIO_API_KEY`, deepseek `DEEPSEEK_API_KEY`)
chatgpt prefix uses octomind login chatgpt | D/usage/04-providers.md:217 | "Sign in with `octomind login chatgpt`; uses your ChatGPT plan"
ollama default is ollama.com; set OLLAMA_API_URL | D/usage/04-providers.md:122-123 | "`ollama:` defaults to `https://ollama.com/v1/chat/completions`, so set an explicit endpoint to use a local Ollama server"
local: uses LOCAL_API_URL; full endpoint path required | D/usage/04-providers.md:131 | "Use the full endpoint path for `LOCAL_API_URL` and `OLLAMA_API_URL`"
config --api-key rejected; keys never in config | D/reference/01-cli-reference.md:120-122 | "the mutation path rejects it at runtime — API keys can never be stored in the config file for security reasons."
Credential load order | D/usage/04-providers.md:103-107 | "with later sources overriding earlier ones: 1. Process environment 2. User-scope `<data-dir>/config/.env` 3. Project-local `./.env`"
-m override example | D/usage/04-providers.md:173 | "octomind run -m 'anthropic:claude-sonnet-4-6'"
Model precedence | D/usage/03-configuration.md:125 | "runtime override > active role profile > tap model mapping > main [model]"
google: prefix not accepted | D/usage/04-providers.md:249-250 | "The historical `google:` prefix is not accepted by the current provider factory; use `google-vertex:` or"
openai uses Responses API; local: for Chat Completions | D/usage/04-providers.md:132 | "The `openai:` adapter uses the Responses API; use `local:` for a local Chat Completions-compatible endpoint."

## Config files
Default paths | D/usage/03-configuration.md:25-27 | "| macOS | `~/.local/share/octomind/` | `~/.local/share/octomind/config/config.toml` |"
Created automatically when absent | D/usage/03-configuration.md:8-9 | "Octomind creates the default configuration automatically when no TOML configuration exists."
OCTOMIND_DATA_DIR / OCTOMIND_CONFIG_PATH | D/usage/03-configuration.md:34-35 | "| `OCTOMIND_DATA_DIR` | Replaces the platform data directory for config and other persistent state |"
config --show summary not full dump | D/reference/01-cli-reference.md:115 | "Display a selected summary (model, logging, rendering, roles, credentials, and MCP), not a full TOML dump."
config --validate checks config and model names not auth | D/usage/04-providers.md:197-198 | "Config validation checks configuration and model names, not remote authentication or account access."
config --upgrade | D/usage/03-configuration.md:15 | "octomind config --upgrade   # run the current migration explicitly"
Merge order, array/table rules | D/usage/03-configuration.md:194-199 | "`config.toml` loads first." ... "Files named `mcp-*.toml` load last" ... "Tables merge recursively and later scalar values replace earlier ones."
Arrays of tables concatenated, same name last wins; other arrays replaced | D/usage/03-configuration.md:198-199 | "entries with the same `name` are deduplicated with the last entry kept." / "Other arrays are replaced by the later value."
Same-named entry replaces whole entry | D/usage/03-configuration.md:202 | "The last same-named array entry replaces the whole entry, so repeat all required server or role fields."
Sibling settings.toml partial override example | D/usage/03-configuration.md:205-212 | "put this partial override in `settings.toml` alongside `config.toml`" (log_level = "debug"; [model] reasoning_effort = "high")
Strict, not sparse overlays | D/reference/03-config-reference.md:916-917 | "Existing configs are strict, not sparse overlays on built-in defaults."
Required fields must exist collectively | D/reference/03-config-reference.md:51-52 | "Existing files must collectively supply required fields; they are"
-c KEY=VALUE one-off, not saved | D/reference/01-cli-reference.md:39,41-42 | "Use `-c/--config KEY=VALUE` to override an existing config value for one invocation." ; "Overrides are not saved to the config file"
Do not modify version | D/reference/03-config-reference.md:69 | "Config version. Do not modify. Used for automatic upgrades."
Root keys before first table header | D/usage/03-configuration.md:57-58 | "Edit these existing keys before any TOML table header; do not append them beneath `[model]`."
Setters: --model, --log-level, --markdown-theme, --mcp-server | D/reference/01-cli-reference.md:103-109 | "| `--model MODEL` | Set root-level model (`provider:model` format). |"
Setters save by rewriting config.toml from the whole merged config (siblings folded in, comments lost) | S/config/loading.rs:441-461 ; S/commands/config.rs:364 | "let config_str = toml::to_string_pretty(self)" ; "if let Err(e) = config.save() {"
Sibling file with same key still wins on next load (config.toml loads first, siblings later) | D/usage/03-configuration.md:194-196 | "`config.toml` loads first." ... "Tables merge recursively and later scalar values replace earlier ones."
--mcp-providers replaces list | D/reference/01-cli-reference.md:105 | "**Replace** the MCP server list: clears all configured servers, then adds each named one as a `builtin` server"
--mcp-server example | D/reference/01-cli-reference.md:158 | "octomind config --mcp-server \"files,type=stdio,command=octofs\""
Stray *.toml in config dir merged | D/troubleshooting/01-common-issues.md:147-148 | "A stray `.toml` backup can therefore change the loaded settings."

## Keys table
default = "assistant:concierge" is tag for bare run | D/reference/03-config-reference.md:71 | "Default tag for bare `run`, `acp`, and `server`."
Root defaults: sandbox false, telemetry true, log_level info | D/usage/03-configuration.md:62-65 | "log_level = \"info\"" / "sandbox = false" / "telemetry = true"
auto_capabilities default true | D/reference/03-config-reference.md:123 | "| `auto_capabilities` | bool | `true` | Enable automatic capability activation on user messages."
max_session_tokens_threshold 200000 | D/reference/03-config-reference.md:92 | "| `max_session_tokens_threshold` | usize | `200000` |"
Spending thresholds default 0.0 = no limit | D/reference/03-config-reference.md:109-110 | "`0.0` = no limit." (max_session_spending_threshold, max_request_spending_threshold)
[model] fields and defaults | D/reference/03-config-reference.md:137-145 | "| `reasoning_effort` | enum | `\"medium\"` | `\"none\"`, `\"low\"`, `\"medium\"`, `\"high\"`, `\"xhigh\"`, or `\"max\"`." (max_tokens 32768, temperature 0.3, top_p 0.7, top_k 20, max_retries 1, request_timeout_seconds 300)
supervisor enabled true | D/reference/03-config-reference.md:640 | "| `enabled` | bool | `true` | Master switch for the whole control plane |"
[supervisor.model] shared by gate, plan, etc.; learning uses it | D/reference/03-config-reference.md:645-646 | "Optional partial profile shared by every supervisor mechanic: authorizer, gate, resolve, plan, condense, extraction, recall,"
supervisor.gate/plan/learning enabled flags | D/reference/03-config-reference.md:714,741,686 | "| `enabled` | bool | `true` | Enable the verify-gate |" ; "Enable adaptive external planning" ; "Enable the learning system (lessons + orientation)"
compression threshold 70000, 0 disables | D/reference/03-config-reference.md:528 | "Baseline automatic compression trigger in tokens; `0` disables automatic checks"
[compression.model] exists | D/reference/03-config-reference.md:566 | "### `[compression.model]`"
[taps] is empty in the shipped template; the entry is an example only | config-templates/default.toml:136-142 | "# Example:" / "# [taps]" / "# \"developer:general\" = \"ollama:glm-5\"" / "# Leave empty to use default model for all tap agents" / "[taps]"
[taps] tag -> model name (doc example) | D/usage/03-configuration.md:232-233 | "[taps]\n\"developer:general\" = \"ollama:glm-5.3\""
registry cache_ttl_hours 24 is a code default, not in the shipped template | D/reference/03-config-reference.md:834 ; S/config/registry.rs:17 | "| `cache_ttl_hours` | u64 | `24` | How long a fetched tap manifest is cached before re-checking. |"
Shipped template values re-checked (default, sandbox, telemetry, log_level, auto_capabilities, thresholds, [model], [skills], [compression], [supervisor*]) | config-templates/default.toml:21-124,575-584,641-659,723-776 | "default = \"assistant:concierge\"" ; "threshold = 70000" ; "max_tokens = 32768"
[skills] auto_activation true, auto_validation false | D/reference/03-config-reference.md:475-476 | "| `auto_activation` | bool | `true` |" ; "| `auto_validation` | bool | `false` |"
Telemetry off switches | D/reference/04-environment-variables.md:235 | "turn it off with `DO_NOT_TRACK=1`, `OCTOMIND_TELEMETRY=0`, or `telemetry = false` in the config."

## Roles, tags, taps
Tag semantics (role name / category:variant / omitted) | D/reference/01-cli-reference.md:58-60 | "A **role name** (e.g. `assistant`) — matched against `[[roles]]` in your config." ; "Omitted — uses the `default` tag from config."
`run` positional is tag not message | D/usage/02-quickstart.md:108-109 | "The positional argument is the agent tag, so pipe the prompt through stdin"
Shipped roles | D/usage/06-roles.md:28-31 | "Config role | `assistant`" / "`task_refiner`" / "`task_researcher`" / "`reduce`"
octomind complete run lists installed/cached tap tags and configured roles, not every tap agent | D/reference/01-cli-reference.md:497,502 | "`complete` reads cached tap data" ; "`run` lists agent tags and configured roles"
roles.toml beside config.toml; [[roles]] with name | D/usage/06-roles.md:50 | "Add this complete role entry in `roles.toml` alongside the generated `config.toml`."
Role required fields; mcp needs both lists | D/usage/06-roles.md:82-88 | "| `mcp` | table | no | If present, include both `server_refs` and `allowed_tools`; if absent, both start empty |"
Custom role example (core server, allowed_tools core:recall) | D/usage/06-roles.md:53-70 | "server_refs = [\"core\"]" ; "allowed_tools = [\"core:recall\"]"
core has no filesystem tools; they come from an external server | D/usage/07-mcp-tools.md:35 | "Filesystem tools come from external servers such as `octofs`; they are not implemented by Octomind."
welcome "" allowed | D/reference/03-config-reference.md:200 | "use `\"\"` for no banner"
`core:*` pattern; server:* grants all tools | D/usage/06-roles.md:161 | "`\"server:*\"` — all tools from a server"
{{CWD}} placeholder | D/usage/03-configuration.md:257 | "| `{{CWD}}` | Current working directory |"
Role example validate + run | D/usage/06-roles.md:73-74 | "octomind config --validate" / "octomind run helper"
Unknown plain role names rejected | D/usage/06-roles.md:335 | "> **Unknown plain role names are rejected.**"
Local role must not share tap tag name | D/usage/06-roles.md:331-333 | "Use a unique plain name for a local customization; a local `developer`"
tap list/add/local symlink | D/reference/01-cli-reference.md:302-308 | "octomind tap" / "octomind tap myorg/my-tap" / "octomind tap myorg/my-tap /path/to/local/tap"
tap myorg/x clones octomind-x repo | D/troubleshooting/01-common-issues.md:271 | "octomind tap myorg/my-agents     # clones github.com/myorg/octomind-my-agents"
untap | D/reference/01-cli-reference.md:344 | "octomind untap myorg/my-tap"
tap init scaffolds, validates, git init, registers | D/reference/01-cli-reference.md:313-315 | "Renders the template, validates it, runs `git init`, and registers the directory as a local tap"
tap init example | D/reference/01-cli-reference.md:327 | "octomind tap init acme/team --agent team:assistant"
muvon/tap always present, last, cannot be added/removed | D/integration/04-tap-system.md:69-70 | "always present as the **last-priority fallback**" ; "cannot be added (`tap muvon/tap` is rejected) or removed"
First-listed tap wins | D/integration/04-tap-system.md:79-80 | "the **first-listed tap wins**"
First use fetches manifest and deps | D/usage/02-quickstart.md:22-23 | "The first use of a tap agent fetches its manifest and dependencies, so it requires network access."
Manifest cache can hide edits; cache path | D/integration/04-tap-system.md:551-553,511 | "so an edit can remain hidden until expiry and a successful background" ; "rm -f \"${OCTOMIND_DATA_DIR:-$HOME/.local/share/octomind}/agents/developer/general.toml\""
{{ENV:KEY}} unset prompts, appends to ./.env | D/integration/04-tap-system.md:210-212 | "otherwise the user is prompted and the value is appended to `./.env` in the current directory"
/role can overwrite /model | D/usage/06-roles.md:320-321 | "The profile application can replace an earlier `/model` or `/effort` choice"

## MCP
Four builtin servers declared | D/usage/03-configuration.md:143 | "The default registry declares four built-in MCP servers:"
Names core, orchestration, runtime, agent | D/usage/03-configuration.md:146-149 | "| `core` |" / "| `orchestration` |" / "| `runtime` |" / "| `agent` |"
filesystem is external (octofs), not builtin | D/usage/07-mcp-tools.md:35 | "Filesystem tools come from external servers such as `octofs`; they are not implemented by Octomind."
External servers are http or stdio | D/usage/03-configuration.md:152 | "External servers use `http` or `stdio`."
Save definition as mcp-project.toml in config dir | D/usage/03-configuration.md:160 | "Save this definition as `mcp-project.toml` in the config directory:"
stdio example fields (command {{ENV}}, args, timeout, tools, auto_bind) | D/usage/03-configuration.md:162-171 | "command = \"{{ENV:PROJECT_MCP_COMMAND}}\"" / "auto_bind = [\"developer:general\"]"
http example with Authorization header | D/usage/07-mcp-tools.md:51-58 | "headers = { Authorization = \"Bearer {{ENV:CUSTOM_API_TOKEN}}\" }  # optional"
Registration doesn't enable; use auto_bind or server_refs | D/usage/03-configuration.md:184-185 | "Registration alone does not enable a server; use `auto_bind` or a role's `server_refs`."
auto_bind exact match | D/usage/03-configuration.md:183 | "`auto_bind` matches exact role strings: `developer` does not match `developer:general`."
tools = [] means all | D/reference/03-config-reference.md:277 | "Tool filter. Empty = all tools."
timeout_seconds 1-3600 | D/reference/03-config-reference.md:299 | "Timeouts must be 1–3600 seconds."
Unset {{ENV}} server cannot connect | D/usage/07-mcp-tools.md:60-62 | "a server whose placeholders reference unset or empty env vars cannot connect and is logged as a startup error."
/mcp list, full, health | D/troubleshooting/01-common-issues.md:218-220 | "| `/mcp list` | Tool names |" / "| `/mcp full` | Tool details and schemas |" / "| `/mcp health` | Server health checks |"
.agents/tools scripts auto-discovered as `local` server | D/usage/07-mcp-tools.md:33 | "Project-local shebang-script tools auto-discovered from `<workdir>/.agents/tools/`."
MCP elicitation declined in piped / --format runs | D/usage/07-mcp-tools.md:107-108 | "The question is **declined** without asking anyone when nobody can answer: piped and `--format` runs"

## Running
Bare octomind = run | D/usage/01-installation.md:47 | "Running `octomind` without a subcommand starts the same interactive session as `octomind run`."
Piped one-shot plain / jsonl | D/usage/02-quickstart.md:112-116 | "echo \"Explain the authentication module\" | \\ octomind run developer:general --format plain"
jq filter on assistant events | D/usage/11-structured-output.md:77-78 | "| jq -r 'select(.type == \"assistant\") | .content'"
--name creates or resumes | D/reference/01-cli-reference.md:185 | "Create a named session, or resume it if it already exists."
--resume-recent | D/reference/01-cli-reference.md:187 | "Resume the most recent session for the current directory"
Piped stdin or --format = non-interactive; plain|jsonl only | D/reference/01-cli-reference.md:195-197 | "If `--format` is given (`plain` or `jsonl`) **or** stdin is piped, the session runs **non-interactively**"
No TTY and no pipe: run goes non-interactive and fails on empty stdin (so Claude pipes every run) | S/commands/run.rs:73,77-81,171-182 | "let is_interactive_session = args.format.is_none() && std::io::stdin().is_terminal();" ; "anyhow::bail!(\"No input provided via stdin\");"
Empty stdin error text | D/troubleshooting/01-common-issues.md:330 | "An empty pipe fails with `No input provided via stdin`."
--format on TTY w/o pipe errors unless --daemon | D/troubleshooting/01-common-issues.md:330-331 | "With terminal stdin, `--format` requires piped input unless you also use `--daemon`."
JSONL types include assistant, thinking, tool_use, tool_result, cost, status, error, mcp_notification, skill, evolution, injected | D/usage/11-structured-output.md:47-59 | "| `mcp_notification` | Notification forwarded from an MCP server |" ; "| `injected` | A non-user message injected into the loop"
JSONL is a stream, last assistant = final; check error events/exit status | D/usage/11-structured-output.md:81,88-89 | "collect the events and select the last" ; "Check the producer's exit status as well as any `error` events"
Exit codes 0/1/2/75 | D/reference/01-cli-reference.md:534-537 | "| `75` | A non-interactive `run` stopped at a spending threshold"
Exit 75 session saved/resumable | D/reference/01-cli-reference.md:537 | "The session is saved, so the run can be resumed (`EX_TEMPFAIL`)."
Sandbox scope + Landlock/Seatbelt | D/reference/01-cli-reference.md:521-523 | "restricts writes with Landlock on Linux or Seatbelt on macOS" ; "applies only to `run`, `server`, and `acp`"
Sandbox config key or flag | D/reference/01-cli-reference.md:522 | "active if **either** the config `sandbox` setting **or** the `--sandbox` flag is set"
Windows sandbox only warns | D/troubleshooting/01-common-issues.md:431 | "| Windows | Sandbox requests log a warning and continue without OS write restrictions. |"
--schema semantics (object file, structured output model, not persisted, best with jsonl) | D/reference/01-cli-reference.md:193,208-210 | "The resolved model must support structured output, or the run fails fast." ; "so pass it again when resuming. Most useful with `--format jsonl`."
Resume without tag keeps role | D/reference/01-cli-reference.md:200 | "Resuming without an explicit tag restores the saved role."
Bare --resume needs TTY | D/reference/01-cli-reference.md:546 | "Supply a session name; the picker requires an interactive TTY."
Daemon needs non-empty piped prompt | D/integration/03-daemon-and-hooks.md:25-26 | "provide a non-empty initial prompt. The process reads stdin to EOF before starting listeners"
Daemon example with redirects | D/integration/03-daemon-and-hooks.md:30-31 | "octomind run --name ci-watcher --daemon --format jsonl >ci-watcher.jsonl 2>ci-watcher.log &"
Daemon pid via & then $! | D/integration/03-daemon-and-hooks.md:30-32,38 | "ci_watcher_pid=$!" ; "kill -TERM \"$ci_watcher_pid\""
send command | D/integration/03-daemon-and-hooks.md:68 | "octomind send --name ci-watcher \"Check the build status\""
send acks queueing not completion | D/integration/03-daemon-and-hooks.md:74 | "A successful send acknowledges enqueueing, not AI completion; read the daemon output for the response."
send fails if session not running | D/integration/03-daemon-and-hooks.md:76 | "If no session by that name is running, `send` fails immediately."
send same OS user / host runtime dir | D/integration/03-daemon-and-hooks.md:93 | "Run the sender as the same OS user with the same runtime environment."
--daemon doesn't detach; stop with kill -TERM; exit 130 | D/integration/03-daemon-and-hooks.md:27,38,41 | "does not detach it from your terminal" ; "kill -TERM \"$ci_watcher_pid\"" ; "exits with status 130"
workflow command, list with no arg | D/reference/01-cli-reference.md:434,460 | "Omit to list available tap workflows." ; "octomind workflow"
Workflow modes: sequential/parallel/loop/conditional/graph | D/usage/09-workflows.md:3-4 | "This guide covers sequential, parallel, loop, conditional, and graph execution for CLI automation."
Workflow step fields and {{input}}, {{step}} | D/usage/09-workflows.md:82-106 | "name    = \"spec\"" / "role    = \"developer:general\"" / "{{input}}" / "Security review of:\n{{spec}}"
Top-level workflow name required; only sequential steps have role/prompt; parallel/loop/conditional steps hold name + nested run list | S/workflow/schema.rs:24-25,73-76,155-201 | "pub struct Sequential {" (name, role, prompt) ; "pub run: Vec<Sequential>," in ParallelStep, LoopStep, ConditionalStep
loop exit_when | D/usage/09-workflows.md:116-118 | "loop           = true" / "exit_when      = { output = \"tester\", contains = \"NO ISSUES\" }"
Workflow stdin required; --dry-run prints plan; stdout empty unless jsonl | D/usage/09-workflows.md:24-25 | "For structured stdout, use `--format jsonl`; without it, stdout is empty. `--dry-run` prints a plan without reading stdin or spawning steps."

## Pitfalls (additional)
.env overrides exported variables; project beats user | D/reference/04-environment-variables.md:35-36 | "**`.env` overrides the system environment.**" ; "**Project-local `.env` overrides user-scope `.env`.**"
Empty assignment masks earlier value | D/reference/04-environment-variables.md:458-459 | "Both `.env` files override exports. Check the project file last; even an empty assignment masks the earlier value."
OCTOMIND_CONFIG_PATH doesn't move user .env | D/reference/04-environment-variables.md:460-461 | "User dotenv loading always uses the standard shared config directory under the data root."

## Smoke test
Diagnose command | D/usage/04-providers.md:193 | "printf '%s\\n' 'Reply with OK.' | octomind run --format plain"
config --validate command | D/usage/03-configuration.md:14 | "octomind config --validate  # validate the merged configuration"

## Read more
Public URL form and slug source | https://octomind.run/docs/usage/01-installation | HTTP 200, prerendered page (2026-10-27)
Docs repo branch master | https://raw.githubusercontent.com/muvon/octomind/master/doc/usage/01-installation.md | HTTP 200 (2026-10-27)
Raw GitHub path doc/<path> | https://raw.githubusercontent.com/muvon/octomind/master/doc/reference/03-config-reference.md | HTTP 200 (2026-10-27)
Page https://octomind.run/docs/usage/03-configuration renders the config guide | WebFetch 2026-10-27 | "Configuration: Models, Roles, and MCP Config in TOML."
Slugs used in the Read more table exist | https://octomind.run/docs/<slug> for each slug, e.g. usage/09-workflows, reference/02-session-commands | HTTP 200 (2026-10-27); slug = doc path without .md

## Doc-vs-code conflicts
Config `version`: template and code are 24; the config reference table row `22` (:69) is stale, its own example shows 24 (:79) | config-templates/default.toml:10 ; S/config/mod.rs:77 ; D/reference/03-config-reference.md:69,79 | "version = 24" ; "pub const CURRENT_CONFIG_VERSION: u32 = 24;" (SKILL.md only says "do not touch version")
