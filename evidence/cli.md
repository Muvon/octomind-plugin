# Evidence for skills/cli/SKILL.md

`D` = `doc/` and `S` = `src/` in https://github.com/muvon/octomind (branch `master`); line numbers re-checked on 2026-11-08 against Octomind 0.70.0. Rows whose quote wraps across lines keep their 2026-10-27 line numbers and may be a few lines off.
Docs branch: `master` (raw files resolve at https://raw.githubusercontent.com/muvon/octomind/master/doc/..., HTTP 200 on 2026-10-27).

claim | source | verbatim quote

## Frontmatter / framing
Octomind is open-source AI coding agent, one Rust binary, any model, MCP-native | D/README.md:3 | "Octomind is an open-source AI coding agent and agent runtime for the terminal: one Rust binary, any model, MCP-native."
Behaviour from TOML config, roles, taps | D/integration/04-tap-system.md:3 | "Install and author reusable Octomind agents, skills, capabilities, workflows, and plugins with taps"
Taps are git-based registries | D/integration/04-tap-system.md:87 | "A tap is a Git repository (or local directory) containing:"
License Apache-2.0 (skill license, from brief and sibling skill) | octomind `skills/octomind/SKILL.md:4` | "license: Apache-2.0"
Compatibility Linux, macOS, Windows; needs login or provider key | octomind `skills/octomind/SKILL.md:5` | "Requires the octomind CLI (Linux, macOS, Windows) and model access via `octomind login` or a provider API key"
Frontmatter fields name/description/license/compatibility valid; compatibility <= 500 chars | https://code.claude.com/docs/en/skills | "Accepts a string of up to 500 characters."
Network needed (tap fetch, provider) | D/usage/02-quickstart.md:133 | "Tap resolution fetches agent definitions and may run dependency setup; model"

## Install
Unix installer URL, installs to ~/.local/bin | D/usage/01-installation.md:11,14-15 | "curl -fsSL https://octomind.run/install.sh | bash" ; "installs `octomind` in `~/.local/bin` by default"
Windows installer command | D/usage/01-installation.md:21 | "irm https://raw.githubusercontent.com/muvon/octomind/master/install.ps1 | iex"
Homebrew command | D/usage/01-installation.md:86 | "brew install muvon/tap/octomind"
cargo install, needs Rust 1.95+ | D/usage/01-installation.md:132,135 | "cargo install octomind" ; "requires Rust 1.95 or newer"
Scoop commands | D/usage/01-installation.md:95 | "scoop bucket add muvon https://github.com/Muvon/scoop-bucket"
Verify with --version and config --show | D/usage/01-installation.md:207 | "octomind --version" / "octomind config --show"
PATH fix | D/usage/01-installation.md:216 | "export PATH=\"$HOME/.local/bin:$PATH\""
Pin a release with --version | D/usage/01-installation.md:162 | "curl -fsSL https://octomind.run/install.sh | bash -s -- --version \"$OCTOMIND_VERSION\""
OCTOMIND_INSTALL_DIR chooses destination | D/usage/01-installation.md:69 | "Set `OCTOMIND_INSTALL_DIR` to choose another destination:"

## Model access
login is browser device flow, stores OCTOHUB_API_KEY in <data-dir>/config/.env | D/usage/04-providers.md:15-16 | "`octomind login` starts a device-authorization flow, displays a short code, and opens a browser approval page."
(same) key location | D/usage/01-installation.md:38-39 | "The completed login stores `OCTOHUB_API_KEY` in `<data-dir>/config/.env`"
Default config uses octohub:auto for main, supervisor, compression | D/usage/01-installation.md:47-48 | "uses `octohub:auto` for its main, supervisor, and compression model purposes."
login --no-browser / --force | D/reference/01-cli-reference.md:106 | "octomind login --force --no-browser"
login blocks until browser confirmation; --force mints new credentials (user runs it, not Claude) | S/commands/login.rs:72-124 | "Confirm the code in your browser to finish signing in." ; "waiting…" ; "if !args.force {"
Replace stored login with --force | D/usage/04-providers.md:217 | "force a new login to replace the machine's stored gateway credential:"
BYOK: change name in all three profiles; main-only leaves others on OctoHub | D/usage/04-providers.md:74-75 | "change `name` in all three existing profiles while keeping their other settings. A main-model override alone leaves the shipped internal profiles on OctoHub"
Model names must be provider:model | D/usage/03-configuration.md:125 | "Model names must use `provider:model`."
OPENAI/ANTHROPIC/OPENROUTER/GOOGLE_STUDIO/DEEPSEEK keys and prefixes | D/usage/04-providers.md:231 | "| OpenAI | `openai` | `OPENAI_API_KEY` |" (also anthropic, openrouter, google-studio `GOOGLE_STUDIO_API_KEY`, deepseek `DEEPSEEK_API_KEY`)
chatgpt prefix uses octomind login chatgpt | D/usage/04-providers.md:229 | "Sign in with `octomind login chatgpt`; uses your ChatGPT plan"
ollama default is ollama.com; set OLLAMA_API_URL | D/usage/04-providers.md:122-123 | "`ollama:` defaults to `https://ollama.com/v1/chat/completions`, so set an explicit endpoint to use a local Ollama server"
local: uses LOCAL_API_URL; full endpoint path required | D/usage/04-providers.md:143 | "Use the full endpoint path for `LOCAL_API_URL` and `OLLAMA_API_URL`"
config --api-key rejected; keys never in config | D/reference/01-cli-reference.md:120-122 | "the mutation path rejects it at runtime — API keys can never be stored in the config file for security reasons."
Credential load order | D/usage/04-providers.md:103-107 | "with later sources overriding earlier ones: 1. Process environment 2. User-scope `<data-dir>/config/.env` 3. Project-local `./.env`"
-m override example | D/usage/04-providers.md:185 | "octomind run -m 'anthropic:claude-sonnet-4-6'"
Model precedence | D/usage/03-configuration.md:128 | "runtime override > active role profile > tap model mapping > main [model]"
google: prefix not accepted | D/usage/04-providers.md:261 | "The historical `google:` prefix is not accepted by the current provider factory; use `google-vertex:` or"
openai uses Responses API; local: for Chat Completions | D/usage/04-providers.md:144 | "The `openai:` adapter uses the Responses API; use `local:` for a local Chat Completions-compatible endpoint."

## Config files
Default paths | D/usage/03-configuration.md:30 | "| macOS | `~/.local/share/octomind/` | `~/.local/share/octomind/config/config.toml` |"
Created automatically when absent | D/usage/03-configuration.md:8-9 | "Octomind creates the default configuration automatically when no TOML configuration exists."
OCTOMIND_DATA_DIR / OCTOMIND_CONFIG_PATH | D/usage/03-configuration.md:39 | "| `OCTOMIND_DATA_DIR` | Replaces the platform data directory for config and other persistent state |"
config --show summary not full dump | D/reference/01-cli-reference.md:142 | "Display a selected summary (model, logging, rendering, roles, credentials, and MCP), not a full TOML dump."
config --validate checks config and model names not auth | D/usage/04-providers.md:197-198 | "Config validation checks configuration and model names, not remote authentication or account access."
config --upgrade | D/usage/03-configuration.md:15 | "octomind config --upgrade   # run the current migration explicitly"
Merge order, array/table rules | D/usage/03-configuration.md:194-199 | "`config.toml` loads first." ... "Files named `mcp-*.toml` load last" ... "Tables merge recursively and later scalar values replace earlier ones."
Arrays of tables concatenated, same name last wins; other arrays replaced | D/usage/03-configuration.md:202 | "entries with the same `name` are deduplicated with the last entry kept." / "Other arrays are replaced by the later value."
Same-named entry replaces whole entry | D/usage/03-configuration.md:202 | "The last same-named array entry replaces the whole entry, so repeat all required server or role fields."
Sibling settings.toml partial override example | D/usage/03-configuration.md:205-212 | "put this partial override in `settings.toml` alongside `config.toml`" (log_level = "debug"; [model] reasoning_effort = "high")
Strict, not sparse overlays | D/reference/03-config-reference.md:916-917 | "Existing configs are strict, not sparse overlays on built-in defaults."
Required fields must exist collectively | D/reference/03-config-reference.md:51-52 | "Existing files must collectively supply required fields; they are"
-c KEY=VALUE one-off, not saved | D/reference/01-cli-reference.md:40 | "Use `-c/--config KEY=VALUE` to override an existing config value for one invocation." ; "Overrides are not saved to the config file"
Do not modify version | D/reference/03-config-reference.md:70 | "Config version. Do not modify. Used for automatic upgrades."
Root keys before first table header | D/usage/03-configuration.md:57-58 | "Edit these existing keys before any TOML table header; do not append them beneath `[model]`."
Setters: --model, --log-level, --markdown-theme, --mcp-server | D/reference/01-cli-reference.md:130 | "| `--model MODEL` | Set root-level model (`provider:model` format). |"
Setters save by rewriting config.toml from the whole merged config (siblings folded in, comments lost) | S/config/loading.rs:443-463 ; S/commands/config.rs:370 | "let config_str = toml::to_string_pretty(self)" ; "if let Err(e) = config.save() {"
Sibling file with same key still wins on next load (config.toml loads first, siblings later) | D/usage/03-configuration.md:198 | "`config.toml` loads first." ... "Tables merge recursively and later scalar values replace earlier ones."
--mcp-providers replaces list | D/reference/01-cli-reference.md:132 | "**Replace** the MCP server list: clears all configured servers, then adds each named one as a `builtin` server"
Mutating flags apply changes then save | D/reference/01-cli-reference.md:126 | "**Mutating flags** (apply changes, then save to the loaded config path):"
Stray *.toml in config dir merged | D/troubleshooting/01-common-issues.md:146 | "A stray `.toml` backup can therefore change the loaded settings."

## Model settings
default = "assistant:concierge" is tag for bare run | D/reference/03-config-reference.md:72 | "Default tag for bare `run`, `acp`, and `server`."
[model] fields and defaults | D/reference/03-config-reference.md:137-145 | "| `reasoning_effort` | enum | `\"medium\"` | `\"none\"`, `\"low\"`, `\"medium\"`, `\"high\"`, `\"xhigh\"`, or `\"max\"`." (max_tokens 32768, temperature 0.3, top_p 0.7, top_k 20, max_retries 1, request_timeout_seconds 300)
request_timeout_seconds, stream_timeout_seconds, request_transport | config-templates/default.toml:124-129 | "# Total deadline for one HTTP or WebSocket request attempt; 0 disables it." ; "# SSE/WebSocket response inactivity limit; 0 disables it." ; "# Persistent WebSocket for octohub/chatgpt requests (\"http\" opts out); other providers ignore it."
Telemetry off switches | D/reference/04-environment-variables.md:236 | "turn it off with `DO_NOT_TRACK=1`, `OCTOMIND_TELEMETRY=0`, or `telemetry = false` in the config."

## Tags and taps
Tag semantics (role name / category:variant / omitted) | D/reference/01-cli-reference.md:58-60 | "A **role name** (e.g. `assistant`) — matched against `[[roles]]` in your config." ; "Omitted — uses the `default` tag from config."
`run` positional is tag not message | D/usage/02-quickstart.md:108-109 | "The positional argument is the agent tag, so pipe the prompt through stdin"
Shipped roles | D/usage/06-roles.md:28-31 | "Config role | `assistant`" / "`task_refiner`" / "`task_researcher`" / "`reduce`"
octomind complete run lists installed/cached tap tags and configured roles, not every tap agent | D/reference/01-cli-reference.md:524 | "`complete` reads cached tap data" ; "`run` lists agent tags and configured roles"
Unknown plain role names rejected | D/usage/06-roles.md:335 | "> **Unknown plain role names are rejected.**"
Local role must not share tap tag name | D/usage/06-roles.md:331-333 | "Use a unique plain name for a local customization; a local `developer`"
tap list/add/local symlink | D/reference/01-cli-reference.md:315 | "octomind tap" / "octomind tap myorg/my-tap" / "octomind tap myorg/my-tap /path/to/local/tap"
tap myorg/x clones octomind-x repo | D/troubleshooting/01-common-issues.md:270 | "octomind tap myorg/my-agents     # clones github.com/myorg/octomind-my-agents"
untap | D/reference/01-cli-reference.md:371 | "octomind untap myorg/my-tap"
muvon/tap always present, last, cannot be added/removed | D/integration/04-tap-system.md:69-70 | "always present as the **last-priority fallback**" ; "cannot be added (`tap muvon/tap` is rejected) or removed"
First-listed tap wins | D/integration/04-tap-system.md:79-80 | "the **first-listed tap wins**"
First use fetches manifest and deps | D/usage/02-quickstart.md:32 | "The first use of a tap agent fetches its manifest and dependencies, so it requires network access."
Manifest cache can hide edits; cache path | D/integration/04-tap-system.md:551-553,511 | "so an edit can remain hidden until expiry and a successful background" ; "rm -f \"${OCTOMIND_DATA_DIR:-$HOME/.local/share/octomind}/agents/developer/general.toml\""
{{ENV:KEY}} unset prompts, appends to ./.env | D/integration/04-tap-system.md:210-212 | "otherwise the user is prompted and the value is appended to `./.env` in the current directory"
/role can overwrite /model | D/usage/06-roles.md:320-321 | "The profile application can replace an earlier `/model` or `/effort` choice"

## MCP
/mcp list, full, health | D/troubleshooting/01-common-issues.md:217 | "| `/mcp list` | Tool names |" / "| `/mcp full` | Tool details and schemas |" / "| `/mcp health` | Server health checks |"
MCP elicitation declined in piped / --format runs | D/usage/07-mcp-tools.md:107-108 | "The question is **declined** without asking anyone when nobody can answer: piped and `--format` runs"

## Running
Bare octomind = run | D/usage/01-installation.md:47 | "Running `octomind` without a subcommand starts the same interactive session as `octomind run`."
Piped one-shot plain / jsonl | D/usage/02-quickstart.md:112-116 | "echo \"Explain the authentication module\" | \\ octomind run developer:general --format plain"
jq filter on assistant events | D/usage/11-structured-output.md:77-78 | "| jq -r 'select(.type == \"assistant\") | .content'"
--name creates or resumes | D/reference/01-cli-reference.md:212 | "Create a named session, or resume it if it already exists."
--resume-recent | D/reference/01-cli-reference.md:214 | "Resume the most recent session for the current directory"
Piped stdin or --format = non-interactive; plain|jsonl only | D/reference/01-cli-reference.md:195-197 | "If `--format` is given (`plain` or `jsonl`) **or** stdin is piped, the session runs **non-interactively**"
No TTY and no pipe: run goes non-interactive and fails on empty stdin (so agent shell tools pipe every run) | S/commands/run.rs:73,77-81,171-182 | "let is_interactive_session = args.format.is_none() && std::io::stdin().is_terminal();" ; "anyhow::bail!(\"No input provided via stdin\");"
Empty stdin error text | D/troubleshooting/01-common-issues.md:329 | "An empty pipe fails with `No input provided via stdin`."
--format on TTY w/o pipe errors unless --daemon | D/troubleshooting/01-common-issues.md:330-331 | "With terminal stdin, `--format` requires piped input unless you also use `--daemon`."
JSONL types include assistant, thinking, tool_use, tool_result, cost, status, error, mcp_notification, skill, evolution, injected | D/usage/11-structured-output.md:47-59 | "| `mcp_notification` | Notification forwarded from an MCP server |" ; "| `injected` | A non-user message injected into the loop"
JSONL is a stream, last assistant = final; check error events/exit status | D/usage/11-structured-output.md:81,88-89 | "collect the events and select the last" ; "Check the producer's exit status as well as any `error` events"
Exit codes 0/1/2/75/130 | D/reference/01-cli-reference.md:561-565 | "| `130` | A `run` session was force-stopped: a second Ctrl+C, or SIGTERM on Unix."
Exit 75 session saved/resumable | D/reference/01-cli-reference.md:564 | "The session is saved, so the run can be resumed (`EX_TEMPFAIL`)."
Sandbox scope + Landlock/Seatbelt | D/reference/01-cli-reference.md:548 | "restricts writes with Landlock on Linux or Seatbelt on macOS" ; "applies only to `run`, `server`, and `acp`"
Sandbox config key or flag | D/reference/01-cli-reference.md:522 | "active if **either** the config `sandbox` setting **or** the `--sandbox` flag is set"
Windows sandbox only warns | D/troubleshooting/01-common-issues.md:430 | "| Windows | Sandbox requests log a warning and continue without OS write restrictions. |"
--schema semantics (object file, structured output model, not persisted, best with jsonl) | D/reference/01-cli-reference.md:220 | "The resolved model must support structured output, or the run fails fast." ; "so pass it again when resuming. Most useful with `--format jsonl`."
Resume without tag keeps role | D/reference/01-cli-reference.md:227 | "Resuming without an explicit tag restores the saved role."
Bare --resume needs TTY | D/reference/01-cli-reference.md:573 | "Supply a session name; the picker requires an interactive TTY."
Daemon needs non-empty piped prompt | D/integration/03-daemon-and-hooks.md:25-26 | "provide a non-empty initial prompt. The process reads stdin to EOF before starting listeners"
Daemon example with redirects | D/integration/03-daemon-and-hooks.md:30-31 | "octomind run --name ci-watcher --daemon --format jsonl >ci-watcher.jsonl 2>ci-watcher.log &"
Daemon pid via & then $! | D/integration/03-daemon-and-hooks.md:30-32,38 | "ci_watcher_pid=$!" ; "kill -TERM \"$ci_watcher_pid\""
send command | D/integration/03-daemon-and-hooks.md:68 | "octomind send --name ci-watcher \"Check the build status\""
send acks queueing not completion | D/integration/03-daemon-and-hooks.md:74 | "A successful send acknowledges enqueueing, not AI completion; read the daemon output for the response."
send fails if session not running | D/integration/03-daemon-and-hooks.md:76 | "If no session by that name is running, `send` fails immediately."
send same OS user / host runtime dir | D/integration/03-daemon-and-hooks.md:93 | "Run the sender as the same OS user with the same runtime environment."
--daemon doesn't detach; stop with kill -TERM; exit 130 | D/integration/03-daemon-and-hooks.md:27,38,41 | "does not detach it from your terminal" ; "kill -TERM \"$ci_watcher_pid\"" ; "exits with status 130"
workflow command, list with no arg | D/reference/01-cli-reference.md:461 | "Omit to list available tap workflows." ; "octomind workflow"
Workflow stdin required; --dry-run prints plan to stdout; stdout otherwise empty unless jsonl | D/usage/09-workflows.md:24-25 ; S/commands/workflow.rs:50-51 | "`--dry-run` prints a plan without reading stdin or spawning steps." ; "Without it, stdout is empty" ; "(except the `--dry-run` plan)"

## Pitfalls (additional)
.env overrides exported variables; project beats user | D/reference/04-environment-variables.md:35-36 | "**`.env` overrides the system environment.**" ; "**Project-local `.env` overrides user-scope `.env`.**"
Empty assignment masks earlier value | D/reference/04-environment-variables.md:458-459 | "Both `.env` files override exports. Check the project file last; even an empty assignment masks the earlier value."
OCTOMIND_CONFIG_PATH doesn't move user .env | D/reference/04-environment-variables.md:460-461 | "User dotenv loading always uses the standard shared config directory under the data root."

## Smoke test
Diagnose command | D/usage/04-providers.md:205 | "printf '%s\\n' 'Reply with OK.' | octomind run --format plain"
config --validate command | D/usage/03-configuration.md:14 | "octomind config --validate  # validate the merged configuration"

## Read more
Public URL form and slug source | https://octomind.run/docs/usage/01-installation | HTTP 200, prerendered page (2026-10-27)
Docs repo branch master | https://raw.githubusercontent.com/muvon/octomind/master/doc/usage/01-installation.md | HTTP 200 (2026-10-27)
Raw GitHub path doc/<path> | https://raw.githubusercontent.com/muvon/octomind/master/doc/reference/03-config-reference.md | HTTP 200 (2026-10-27)
Page https://octomind.run/docs/usage/03-configuration renders the config guide | WebFetch 2026-10-27 | "Configuration: Models, Roles, and MCP Config in TOML."
Slugs used in the Read more table exist | https://octomind.run/docs/<slug> for each slug, e.g. usage/09-workflows, reference/02-session-commands | HTTP 200 (2026-10-27); slug = doc path without .md

## Config load and writers
No per-project config: only the config directory is read | S/config/loading.rs:355-364,414 | "let mut merged_value = load_and_merge_toml_from_directory(config_dir)?;"
`.agents/` holds guardrails, tools and skills | evidence/customize/guardrails.md ; evidence/customize/local-tools.md ; evidence/customize/octomind-skills.md | see those rows
Loading may create or migrate before any command runs | D/reference/01-cli-reference.md:124 | "Loading happens before command dispatch and may create or migrate config even with `--show` or `--validate`."
Default template written when no config exists | S/config/loading.rs:373-375,400-402 | "// No config files at all, inject default"
Older config.toml migrated on load, with backup path on stderr | S/config/loading.rs:407-411 ; S/config/migrations.rs:811-820,858-863 | "// Check for automatic config upgrades on config.toml" ; "Config upgraded successfully! Backup saved to: {}"
-c runs skip migration; a missing config.toml is not migrated | S/config/loading.rs:408 | "if overrides.is_empty() && config_path.exists() {"
Only config.toml is migrated, siblings are not | S/config/loading.rs:409 | "super::migrations::check_and_upgrade_config(&config_path)"
Missing keys are not filled from the template | S/config/loading.rs:421,439-440 | "Failed to parse merged TOML configuration. All required fields must be present." ; "REMOVED: No more default_with_env - config must be complete and explicit"
Unknown keys are captured; only config --validate reports them | S/config/validation.rs:324-329 ; S/commands/config.rs:95 | "pub fn deserialize_with_ignored_keys" ; "let ignored_reports = config.check_ignored_keys();"
-c dotted path must already exist | S/config/loading.rs:273-275 | "Unknown config override path '{path}'"
-c rejected on config and configure | S/main.rs:159-160 | "--config overrides cannot be used with the config or configure subcommands"
configure changes only three model names, keeps comments, keys in .env | D/reference/01-cli-reference.md:83-85 | "comments and other settings are preserved." ; "Credentials go into the user-scope"
configure needs a terminal on stdin and stdout | S/commands/configure.rs:69-70 | "octomind configure requires an interactive terminal"
login writes hub key to .env and session to auth.json | S/account.rs:358-362 | "Persist a confirmed login: hub key into the user-scope `.env`, panel session"
/model is session-only | S/session/chat/session/commands/model.rs:53 | "// The selection belongs to this session, not the shared configuration."
Runtime persist writes mcp-<name>.toml | S/mcp/runtime/dynamic.rs:524 | "Writes `<config_dir>/mcp-<name>.toml` with `[[mcp.servers]]` format"
Taps stored in data dir taps.toml | S/agent/taps.rs:131 | "Ok(crate::directories::get_octomind_data_dir()?.join(\"taps.toml\"))"

## Other commands
evaluate input, -m and [decision].model | S/commands/evaluate.rs:31-39,54-56 | "JSON request file: `{\"state\": ..., \"questions\": {\"id\": {\"type\": \"noul\", \"instructions\": \"...\"}}}`." ; "decision.model is absent or empty; pass --model"
vars flags | D/reference/01-cli-reference.md:380-381 | "| `--preview` | `-p` |" ; "| `--expand` | `-e` |"
server flags and defaults | D/reference/01-cli-reference.md:279-280 | "Bind address (default: `127.0.0.1`)" ; "Port (default: `8080`)"
server has no TLS; token optional | D/integration/01-websocket-server.md:48,862 | "unset = no authentication" ; "The server has no TLS."
acp flags (`--hook` removed from `acp` on 2026-11-09: only the run path starts listeners) | S/commands/acp.rs:19-47 ; S/session/chat/session/main_loop.rs:83 | "pub resume_recent: bool," ; "let webhooks = start_webhook_guards(args, config, &chat_session.session.info.name).await?;"
completion shells | D/reference/01-cli-reference.md:499 | "Target shell: `bash`, `zsh`, `fish`, `powershell`, `elvish`"


## Shared local inference
Hindsight model sharing, idle lifetime and hidden helper | S/supervisor/hindsight/shared.rs; S/main.rs; D/reference/03-config-reference.md | "Hindsight inference is shared across Octomind processes using the same runtime directory." ; "The worker exits after 15 minutes without clients."
