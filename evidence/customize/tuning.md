# Evidence for references/tuning.md

`D` = `doc/`, `S` = `src/`, `T` = `config-templates/default.toml` in https://github.com/muvon/octomind (checkout used: Octomind 0.70.0); line numbers as of 2026-11-08.
`Test` = run in a throwaway `OCTOMIND_DATA_DIR=$(mktemp -d)` with `octomind config --validate` (or `octomind -c ... vars` for `-c`); the quote is the program output, probes in /tmp/tuning_probe. Code beats docs; conflicts are listed at the end.

claim | source | verbatim quote

## Version and method

Checked against 0.70.0 | Test `octomind --version` | "octomind 0.70.0"
Each test started from the config that `config --validate` creates | Test (fresh data dir) | "Configuration is valid."
That first run writes config.toml from the template embedded in the binary | S/config/loading.rs:292 | "const DEFAULT_CONFIG_TEMPLATE: &str = include_str!(\"../../config-templates/default.toml\");"
 | S/config/loading.rs:300 | "fs::write(config_path, DEFAULT_CONFIG_TEMPLATE).context(format!("

## How to change a setting

All `*.toml` files in the config directory merge | D/reference/03-config-reference.md:27 | "Octomind supports split-file configuration. All `*.toml` files in the config directory are merged:"
config.toml first, others alphabetical | D/reference/03-config-reference.md:29-30 | "1. `config.toml` is loaded first" / "2. Other `*.toml` files are loaded alphabetically"
Same order in code | S/config/loading.rs:178 | "/// Order: config.toml first, then other *.toml files in alphabetical order"
Later files win | S/config/loading.rs:201 | "// Merge is last-wins, so `config.toml` MUST sort first — plain alphabetical"
Tables merge key by key, scalars replaced | S/config/loading.rs:110 | "/// Arrays of tables are concatenated (e.g. [[mcp.servers]]), tables merge deeply, scalars override"
Config dir is `<data-dir>/config` | S/directories.rs:72 | "let config_dir = data_dir.join(\"config\");"
`OCTOMIND_DATA_DIR` moves the data dir | S/directories.rs:23 | "/// `OCTOMIND_DATA_DIR` overrides the location of every piece of state (config,"
Default data dir `~/.local/share/octomind` | S/directories.rs:48 | "let path = home.join(\".local\").join(\"share\").join(\"octomind\");"
`OCTOMIND_CONFIG_PATH` sets the file; directory is its parent | S/config/loading.rs:348 | "let config_path = if let Ok(custom_path) = std::env::var(\"OCTOMIND_CONFIG_PATH\") {"
 | S/config/loading.rs:355 | "// Get the config directory (config file's parent). A relative single-"
 | D/reference/03-config-reference.md:49 | "The selected `OCTOMIND_CONFIG_PATH` determines the directory to merge and the primary migration/save path; it does not"
Sibling file with only the changed key is valid (config.toml keeps the required keys) | Test `90-override.toml` = `[supervisor.condense]` + `tokens_threshold = 8000` | "Configuration is valid."
Same for gate, supervisor, learning, evolution, authorizer, advise, hindsight, plan, condense, compression, decision, skills, registry, timing, sandbox, thresholds | Test (38 sibling cases in /tmp/tuning_probe/siblings.sh; the 31 positive ones exit 0, the 7 negative ones are quoted in their own rows) | "✓ config validate · valid"
Root key placed after a table header becomes a key of that table and is rejected | Test sibling `[model]` + `name` + `sandbox = true` | "warning: unknown configuration key `model.sandbox`"
 | S/config/validation.rs:393 | "\"in TOML, a scalar placed after a [header] becomes a field of that table; did you mean to place this in the parent section before the header?\".to_string()"
 | T:683-684 | "# NOTE: scalar compression keys must remain before the nested attention and"
Octomind does not fill missing keys; a missing required key fails loading | S/config/loading.rs:421 | "\"Failed to parse merged TOML configuration. All required fields must be present.\""
 | D/reference/03-config-reference.md:52-53 | "exists, loading creates the embedded default config. Existing files must collectively supply required fields; they are" / "not automatically overlaid on the template."
 | D/reference/03-config-reference.md:21 | "The tables below follow template order. “Default” means the value shipped in that template, not permission to omit a"
Error text is `missing field` | Test delete `sandbox` | "missing field `sandbox`"
Disable a mechanic with `enabled = false`, keep the table | D/usage/14-supervisor.md:26 | "are optional overrides. Required mechanic tables must remain present even when disabled. Detectors and recitation run"
Deleting a table fails | Test delete `[supervisor.gate]` | "missing field `gate`"
Required root keys: `sandbox` | S/config/mod.rs:432 | "pub sandbox: bool,"
 `auto_capabilities` | S/config/mod.rs:459 | "pub auto_capabilities: bool,"
 `cache_keepalive_enabled` | S/config/mod.rs:375 | "pub cache_keepalive_enabled: bool,"
 `commit_attribution` | S/config/mod.rs:442 | "pub commit_attribution: String,"
 both spending thresholds | S/config/mod.rs:380-382 | "pub max_session_spending_threshold: f64," / "pub max_request_spending_threshold: f64,"
 both token thresholds | S/config/mod.rs:365-366 | "pub mcp_response_tokens_threshold: usize," / "pub max_session_tokens_threshold: usize,"
 each one individually | Test delete each key | "missing field `max_session_tokens_threshold`" (same form for the other seven)
Every key of `[model]` required | S/config/model.rs:53-66 | "pub struct ModelProfile {" ... "pub request_transport: RequestTransportConfig,"
 | Test delete `model.request_transport`, `model.stream_timeout_seconds` | "missing field `request_transport`" / "missing field `stream_timeout_seconds`"
`[skills]` all five keys required | S/config/mod.rs:239-255 | "pub struct SkillsConfig {" ... "pub max_retries: u32,"
 | Test delete each `skills.*` key | "missing field `activation_timeout`"
`[timing]` all six keys required | S/config/mod.rs:259-273 | "pub struct TimingConfig {" ... "pub switch_cost_dhe: f64,"
 | Test delete `timing.think_overhead_min` | "missing field `think_overhead_min`"
`[decision]` table and five booleans required | S/config/mod.rs:413 | "pub decision: DecisionConfig,"
 | S/config/decision.rs:22-26 | "pub recall: bool," / "pub skills: bool," / "pub capabilities: bool," / "pub compression: bool," / "pub supervisor: bool,"
 | Test delete table `decision` | "missing field `decision`"
`[supervisor]` requires `enabled`, `max_identical_rounds` and the seven tables | S/supervisor/mod.rs:214-236 | "pub enabled: bool," / "pub max_identical_rounds: usize," / "pub learning: learning::LearningConfig," / "pub gate: GateConfig," / "pub authorizer: authorizer::AuthorizerConfig," / "pub plan: PlanConfig," / "pub condense: CondenseConfig," / "pub hindsight: hindsight::HindsightConfig," / "pub advise: advise::AdviseConfig,"
 | Test delete each | "missing field `max_identical_rounds`" / "missing field `authorizer`" / "missing field `hindsight`" / "missing field `advise`" / "missing field `plan`" / "missing field `condense`"
`condense` keys required | S/supervisor/mod.rs:249-257 | "pub enabled: bool," / "pub adaptive: bool," / "pub tokens_threshold: usize,"
 | Test | "missing field `tokens_threshold`"
`gate` keys required | S/supervisor/mod.rs:263-265 | "pub enabled: bool," / "pub unsettled_advisory: bool,"
 | Test | "missing field `unsettled_advisory`"
`plan`, `authorizer`, `advise` have `enabled`; `hindsight` has `enabled` and `model` | S/supervisor/mod.rs:271 | "pub enabled: bool,"
 | S/supervisor/hindsight.rs:41-43 | "pub enabled: bool," / "pub model: String,"
 | Test delete `supervisor.hindsight.model` | "missing field `model`"
`learning` table required, `evolution` inside it required with six keys | S/supervisor/learning/mod.rs:263 | "pub evolution: evolution::EvolutionConfig,"
 | S/supervisor/learning/evolution/mod.rs:42-55 | "pub enabled: bool," / "pub min_samples: u32," / "pub noise_margin: f64," / "pub cost_allowance: f64," / "pub cost_per_gain: f64," / "pub max_trial_uses: u32,"
 | Test delete `evolution` table / `min_samples` / `max_trial_uses` | "missing field `evolution`" / "missing field `min_samples`" / "missing field `max_trial_uses`"
 | Test remove both learning tables | "missing field `learning`"
`supervisor.learning.enabled` may be missing and then means false | S/supervisor/learning/mod.rs:260-261 | "#[serde(default)]" / "pub enabled: bool,"
 | Test delete `supervisor.learning.enabled` | "Configuration is valid."
 | D/usage/13-learning.md:165 | "> tables are required by deserialization. `LearningConfig::default()` has `enabled = false`, while the shipped template"
`[compression]` requires `threshold`, `idle`, `analysis_findings_max_tokens` | S/config/mod.rs:197 | "pub threshold: usize,"
 | S/config/mod.rs:208 | "pub analysis_findings_max_tokens: usize,"
 | S/config/mod.rs:211 | "pub idle: bool,"
 | Test delete each | "missing field `idle`" / "missing field `analysis_findings_max_tokens`" / "missing field `threshold`"
Optional: `compression.model`, `knowledge_retention`, `attention` | S/config/mod.rs:200-216 | "#[serde(default)]" / "#[serde(default = \"default_knowledge_retention\")]" / "#[serde(default)]"
 | Test delete each | "Configuration is valid."
Optional: `supervisor.model` | S/supervisor/mod.rs:220-221 | "#[serde(default)]" / "pub model: crate::config::ModelProfileOverride,"
 | Test delete table `supervisor.model` | "Configuration is valid."
Optional: `decision.model` | S/config/decision.rs:21 | "pub model: Option<String>,"
 | Test delete `decision.model` | "Configuration is valid."
Optional: `[capabilities]`, `[taps]`, `[registry]`, `telemetry` | S/config/mod.rs:447-448 | "#[serde(default)]" / "pub capabilities: HashMap<String, String>,"
 | S/config/mod.rs:454-455 | "#[serde(default)]" / "pub taps: HashMap<String, String>,"
 | S/config/mod.rs:472-473 | "#[serde(default)]" / "pub registry: crate::config::registry::RegistryConfig,"
 | S/config/mod.rs:437-438 | "#[serde(default = \"default_telemetry\")]" / "pub telemetry: bool,"
 | Test delete each | "Configuration is valid."
`version` triggers an upgrade when behind the binary | T:9-10 | "# Configuration version (DO NOT MODIFY - used for automatic upgrades)" / "version = 29"
 | S/config/migrations.rs:15-16 | "//! Automatic upgrades of `config.toml` when the embedded template's schema" / "//! version moves ahead of the user's file."
Upgrade copies missing keys from the template | S/config/migrations.rs:727 | "/// are preserved; only missing keys are copied from the embedded template."
Upgrade writes a backup | S/config/migrations.rs:861 | "\"✅ Config upgraded successfully! Backup saved to: {}\","
 | Test delete `version` | "🔄 Upgrading config from version 0 to 29..."
An up-to-date file is not touched | S/config/migrations.rs:809-810 | "/// Returns whether the file was rewritten. The common case — an up-to-date" / "/// config — takes no lock and touches nothing."
Upgrade runs only when no `-c` overrides are given | S/config/loading.rs:408 | "if overrides.is_empty() && config_path.exists() {"
Wrong types fail: string for integer | Test `tokens_threshold = "8000"` | "invalid type: string \"8000\", expected usize in `supervisor.condense.tokens_threshold`"
 string for boolean | Test `enabled = "false"` | "invalid type: string \"false\", expected a boolean in `supervisor.gate.enabled`"
 negative count | Test `threshold = -1` | "invalid value: integer `-1`, expected usize in `compression.threshold`"
 integer accepted for a decimal threshold | Test `max_session_spending_threshold = 5` | "Configuration is valid."
`[decision]` rejects unknown keys | S/config/decision.rs:19 | "#[serde(deny_unknown_fields)]"
 | Test `recal = true` | "unknown field `recal`, expected one of `model`, `recall`, `skills`, `capabilities`, `compression"
 | D/usage/14-supervisor.md:148 | "`decision.model`. Unknown keys and missing or mistyped booleans fail config parsing."
`-c key=value` overrides for one run | S/main.rs:33-34 | "/// Override a config value for this process using a dotted path and TOML value (e.g. `-c supervisor.enabled=false`)."
Key must already exist | S/config/loading.rs:255 | ".ok_or_else(|| anyhow::anyhow!(\"Unknown config override path '{path}'\"))?,"
 | Test `-c supervisor.gate.bogus=1 vars` | "Error: Unknown config override path 'supervisor.gate.bogus'"
Value parsed as TOML | S/config/loading.rs:232-236 | "fn parse_config_override_value(value: &str) -> toml::Value {" / "match value.parse() {"
Unquoted `-c model.request_transport=http` accepted | Test | "exit=0" (`octomind -c model.request_transport=http vars`)
`-c` refused with `config` | S/main.rs:159-160 | "if !config_overrides.is_empty() && matches!(&command, Commands::Config(_) | Commands::Configure) {" / "anyhow::bail!(\"--config overrides cannot be used with the config or configure subcommands\");"
 | Test | "Error: --config overrides cannot be used with the config or configure subcommands"
Other commands load and check `-c` values | S/main.rs:215 | "let config = Config::load_with_overrides(&config_overrides)?;"
 | Test `-c model.request_transport=tcp vars` | "unknown variant `tcp`, expected `http` or `websocket`"
`config --validate` prints one warning per unknown key and then fails | S/commands/config.rs:96-97 | "for report in &ignored_reports {" / "block_line(&report.format_warning().bright_yellow().to_string());"
 | S/commands/config.rs:120 | "anyhow::bail!(\"{err_msg}\");"
 | S/config/validation.rs:245 | "format!(\"warning: unknown configuration key `{}`\", self.path)"
 | Test `[supervisor.gate]` + `bogus = 1` | "warning: unknown configuration key `supervisor.gate.bogus`" / "Error: Found 1 unknown or ignored configuration key" (exit 1)
Typo'd table name is also flagged | Test `[supervisor.gates]` | "warning: unknown configuration key `supervisor.gates`"
Changes apply to a new session | D/use-case 11-keep-ai-spend-predictable.md:179 | "Start a fresh session so the new configuration loads:"
`config --show` is a summary only | D/reference/03-config-reference.md:17 | "path and credential overrides. `--show` displays selected settings, not a complete TOML dump. Startup can create or"
 | D/use-cases/11-keep-ai-spend-predictable.md:221 | "`config --show` is a selected summary, not a full TOML dump: it does not print every budget or keepalive field."
 | S/commands/config.rs:662 | "block_row(\"root model\", &format_root_model(config).bright_white().to_string(), kw);"
`/info` shows supervisor activity | D/usage/14-supervisor.md:20-21 | "`/plan` displays the supervisor-owned plan and retained critical knowledge; routine work may have no plan. `/info`" / "includes supervisor activity and model usage. You do not type the hidden status protocol yourself."

## Make done mean done

Keys and defaults: supervisor, gate, unsettled_advisory, plan on | T:736 | "enabled = true"
 | T:790-793 | "[supervisor.gate]" / "enabled = true" / "# Offer one nonblocking evidence advisory per genuine user turn, then re-gate." / "unsettled_advisory = true"
 | T:826-827 | "[supervisor.plan]" / "enabled = true"
With supervisor and gate on every stop is checked by a separate model pass | D/usage/14-supervisor.md:282-283 | "With supervision and the gate enabled, every stop of a user-owned turn — a `done` claim, a hand-back, or a turn that ends" / "with no status — is checked by an independent model verification pass before it is accepted."
Gaps send the agent back, bounded retries | D/usage/14-supervisor.md:324 | "- **Gaps** → an advisory listing the gaps is injected and the turn re-runs, bounded by a fixed re-entry budget."
Hand-back passes only if the work truly needs the user | D/usage/14-supervisor.md:296-297 | "passes one only when the remaining work truly needs you (a decision, access the environment lacks, or an action the" / "request forbids), and otherwise sends the agent back to clear the blocker itself."
Completion pre-gates need the gate switch | D/usage/14-supervisor.md:26-27 | "are optional overrides. Required mechanic tables must remain present even when disabled. Detectors and recitation run" / "when the supervisor is enabled; completion pre-gates also require the gate to be enabled."
`unsettled_advisory` meaning | D/reference/03-config-reference.md:754 | "| `unsettled_advisory` | bool | `true` | Offer one nonblocking evidence advisory per genuine user turn, then re-gate normally |"
Plan manager is for dependent multi-phase work | D/usage/14-supervisor.md:337-338 | "Planning is exceptional and supervisor-owned. Focused answers and routine work stay plan-free. For work with meaningful" / "dependent phases, context-loss risk, or a real branch to track, the specialist emits a sparse hidden `request` signal"
The gate judges evidence, not tests | D/usage/14-supervisor.md:292-293 | "and exercise behavior, so its output remains available as evidence. The verifier judges the requested outcome against" / "the resulting artifacts and recorded outputs, and must identify a concrete gap before requesting more work."
 | D/use-cases/12-make-done-mean-done.md:185 | "It does not promise that every model-written completion sentence implies a passing suite."
Guardrail validator is a separate file and system | D/use-cases/12-make-done-mean-done.md:159-160 | "Save `.agents/guardrails.toml`. This validator is independent of `[skills].auto_validation` and runs after every" / "assistant turn for the exact `completion` role."
Verify with `/info`, `/plan`, `/loglevel debug` | D/usage/14-supervisor.md:457-460 | "/loglevel debug" / "/info" / "/plan" / "/loglevel info"
Gate retries are bounded | D/use-cases/12-make-done-mean-done.md:268 | "Gate retries are bounded, and post-turn feedback does not retract the displayed answer. Keep the independent test."

## Stop runaway loops

Key, default, `0` = off, needs `enabled` | T:738-743 | "# Hard stop for runaway tool loops: when this many tool rounds in a row repeat" / "# Needs `enabled`. 0 = off." / "max_identical_rounds = 10"
Turn ends with a notice, no further request, labelled failed for learning | D/usage/14-supervisor.md:273-278 | "One limit is not advisory. When `supervisor.max_identical_rounds` tool rounds in a row (default 10) repeat calls and" / "notice naming the repeated call and its result. No further model request is made, and neither the verify-gate nor the" / "ends: this round's results are recorded, and the turn's final answer is a" / "unfinished-hand-back check re-runs the turn. The trajectory is labelled failed for learning."
Hint thresholds are fixed | D/usage/14-supervisor.md:254 | "Detectors run in-process after tool rounds without another model call. Their thresholds are fixed constants. The status"

## Cap spending

Keys, root, decimals, `0.0` disables | T:90-97 | "# Session spending threshold in USD (0.0 = no limit)" / "# Request spending threshold in USD (0.0 = no limit)"
 | S/session/chat/session/messages.rs:117,190 | "if config.max_session_spending_threshold <= 0.0 {" / "if config.max_request_spending_threshold <= 0.0 {"
Request threshold stops execution once the cost since the request began reaches it | S/session/chat/session/messages.rs:196,199 | "let cost_since_request_start = current_cost - self.request_spending_checkpoint;" / "if cost_since_request_start >= threshold {"
 | S/session/chat/session/messages.rs:215 | "\"Request execution stopped to prevent overspending.\""
Session threshold asks to continue | S/session/chat/session/messages.rs:123,161 | "let cost_since_checkpoint = current_cost - self.spending_threshold_checkpoint;" / "eprint!(\"{}\", \"Do you want to continue? (y/N): \".bright_white().bold());"
Non-interactive runs decline and stop | S/session/chat/session/messages.rs:154 | "\"Spending threshold reached but automatically declining in non-interactive mode. Stopping execution.\".bright_red()"
 | D/reference/03-config-reference.md:108 | "| `max_session_spending_threshold` | f64 | `0.0` | USD spent since last accepted checkpoint; interactive CLI asks to continue, piped/ACP/WebSocket declines. `0.0` = no limit. |"
Notices printed | S/session/chat/session/messages.rs:139,206 | "\"⚠️  SPENDING THRESHOLD REACHED ⚠️\"" / "\"⚠️  REQUEST SPENDING THRESHOLD EXCEEDED ⚠️\""
Checks use recorded cost, may overshoot, not a billing limit | D/use-cases/11-keep-ai-spend-predictable.md:225-226 | "These thresholds use costs already recorded at checkpoints. In-flight requests can overshoot, and a single final" / "answer need not reach a tool-follow-up check. They are not prepaid per-task reservations. Detached exit learning and"
 | D/use-cases/11-keep-ai-spend-predictable.md:240 | "reduce output limits, and inspect tool follow-ups and separate services. Do not use it as a provider billing ceiling."
Verify with `/info`, `/report` | D/use-cases/11-keep-ai-spend-predictable.md:154 | "`/info` includes session cost and tokens. `/report` reads the session log and groups costs and tool use by request."
Example values validate | Test `max_request_spending_threshold = 0.5` + `max_session_spending_threshold = 5.0` in a sibling | "Configuration is valid."

## Lower cost

Supervisor model calls share one profile | T:745 | "# One shared profile for gate, resolve, plan, condense, and learning."
 | D/reference/03-config-reference.md:658-660 | "Optional partial profile shared by every supervisor mechanic: authorizer, gate, resolve, plan, condense, extraction, recall," / "retention, verification, and evolution. It accepts every field from `[model]`; omitted fields inherit main. Omitting the"
 Advise uses the same call | D/usage/14-supervisor.md:358 | "shared supervisor-model call reads your request, the specialist's standing instructions, the live plan, the last tool"
`reasoning_effort` values | T:114 | "# Valid values: \"low\", \"medium\", \"high\", \"xhigh\", \"max\"."
 `none` also accepted | S/config/mod.rs:285 | "None,"
 | D/reference/03-config-reference.md:137 | "`\"none\"`, `\"low\"`, `\"medium\"`, `\"high\"`, `\"xhigh\"`, or `\"max\"`."
`name` must be `provider:model` and valid | Test `[supervisor.model] name = "nonexistent-provider:foo"` | "Error: supervisor.name 'nonexistent-provider:foo' is invalid: Unsupported provider: nonexistent-provider."
Compression model writes the summaries | D/reference/03-config-reference.md:579 | "Model used for compression decisions and summary generation."
`supervisor.enabled = false` turns off the control plane | T:735 | "# Master switch for the whole control plane"
 | D/usage/14-supervisor.md:466-467 | "**How do I turn supervision and memory off?** Set both switches in the existing config; recall and extraction check" / "`supervisor.learning.enabled` separately:"
Hindsight and authorizer need the master switch | S/supervisor/hindsight.rs:60 | "config.supervisor.enabled && config.supervisor.hindsight.enabled"
 | D/reference/03-config-reference.md:774 | "evidence and independent confirmation by the same shared model. Requires the supervisor master switch for root sessions."
Keepalive pings keep the prompt cache warm | T:61 | "# Keep the prompt cache warm with periodic pings while the session waits: for"
Keepalive is opt-in, costs cache-read tokens, provider-dependent | T:72 | "# a ping misses the cache. Each ping costs cache-read tokens, so this is opt-in."
 | T:74 | "# Provider-aware: only providers with a keepalive policy are pinged; others are"
`/info` shows model usage | D/usage/14-supervisor.md:21 | "includes supervisor activity and model usage. You do not type the hidden status protocol yourself."
`/usage` | D/use-cases/11-keep-ai-spend-predictable.md:152 | "Send each line separately and wait for the answer before the next line. `/usage` fetches your Octomind account's"

## Compress later or sooner

`threshold` meaning, `0` disables | T:658-659 | "# Single compression trigger in absolute tokens (0 = compression disabled)." / "# Compression becomes eligible once the full context exceeds this. How deep each"
Depth is computed, no ratio key | D/usage/08-compression.md:183 | "- Increase `compression.threshold`; compression depth is computed and has no `target_ratio` config key."
`/done` still compresses with threshold 0 | D/usage/08-compression.md:171 | "Manual `/done` remains available. The final context-ceiling check can still reject an oversized request."
Context ceiling is the lower of the token threshold and the model window | D/usage/08-compression.md:231-232 | "The context ceiling is the lower of `max_session_tokens_threshold` (root config, default `300000`) and the session" / "model's physical window minus the reserved completion budget (`max_tokens`)."
Raising the threshold cannot fix one oversized request | D/usage/08-compression.md:191-192 | "that request, reduce tool output, or select a model with more usable context; raising the soft threshold cannot fix it."
`idle` meaning | T:666-667 | "# Idle compaction (on by default): when you send a message after the provider's" / "# prompt cache has expired, compact first if the cost math says it saves money."
`knowledge_retention` | T:647-651 | "# Maximum number of critical knowledge entries retained across compressions." / "knowledge_retention = 25"
`analysis_findings_max_tokens = 0` | T:655 | "# they do not all fit. 0 disables analysis-finding retention."
`[compression.attention] enabled` switches on attributed summaries | D/usage/08-compression.md:71 | "| `compression.attention.enabled` | `false` | Enable attributed PACT summaries |"
Scalars before nested tables | D/usage/08-compression.md:77-78 | "To enable attributed PACT summaries, set these tables after scalar `[compression]` fields; replace existing tables" / "rather than defining the same table twice:"
 | T:683-685 | "# NOTE: scalar compression keys must remain before the nested attention and" / "# model tables. A scalar placed after a nested table header is parsed as a" / "# field of that table and silently ignored."
Verify: `/info` compression block, debug log | D/usage/08-compression.md:139-142 | "compression" / "  runs               3 · 3 conversation" / "  tokens saved       45,000"
 | D/usage/08-compression.md:199 | "Debug logs explain fire-line eligibility, feasibility, and fold economics; `/info` shows the accumulated results."
Example validates | Test `[compression]` + `threshold = 150000` | "Configuration is valid."
`threshold = 0` validates | Test | "Configuration is valid."

## Shrink or cut large tool results

`mcp_response_tokens_threshold` truncates responses, `0` unlimited | T:53-54 | "# Global token limit for ALL MCP tool responses (0 = unlimited)" / "# When exceeded, responses are automatically truncated with notice"
Condense narrows a result above the threshold to needed lines | D/usage/14-supervisor.md:373-376 | "When a plain-text tool result exceeds `[supervisor.condense] tokens_threshold` (with a 512-token minimum floor), it" / "becomes a condense candidate. Results under the threshold are passed through exactly as returned and are never shown to" / "the condenser."
Original is saved to a file the agent can read | D/usage/14-supervisor.md:385-386 | "It is recoverable: condensation runs only for plain-text results when the active role has a local file-reading tool, the" / "body being narrowed is spilled to a temporary file first, and every condensed result carries the path so the agent can"
`tokens_threshold = 0` turns it off | T:844 | "# Per-result trigger (estimated tokens of that single text result). 0 = off."
 | D/usage/14-supervisor.md:464 | "selections with no token savings stay unchanged. `tokens_threshold = 0` disables condensation."
`adaptive` learns within a bounded range | T:840-842 | "# Learn a bounded per-session trigger from realized condenser savings. The" / "# runtime multiplier starts at 1.0 and stays within 0.5x..2.0x of"
Keep it well below the truncation limit | T:845-846 | "# Keep well below mcp_response_tokens_threshold: that cap is the ceiling applied" / "# first, this is the smart narrowing applied to what survives it."
Nothing condensed: small results, rich payloads, missing reader tool | D/usage/14-supervisor.md:463-464 | "**Why was nothing condensed?** Results below the effective threshold, rich MCP payloads, missing spill-reader tools, and"
`max_session_tokens_threshold`: ceiling, `0` removes the configured limit | T:58-59 | "# Maximum tokens per session before truncation kicks in (0 = disabled, >0 = enabled)"
 | D/usage/08-compression.md:178 | "- A zero `max_session_tokens_threshold` removes the configured ceiling; the usable model window still applies."
Upper bound on the value | S/config/validation.rs:141-143 | "if self.max_session_tokens_threshold > 2_000_000 {" / "\"Max session tokens threshold too high: {}. Maximum allowed: 2,000,000\","
 | Test `max_session_tokens_threshold = 3000000` | "Error: Max session tokens threshold too high: 3000000. Maximum allowed: 2,000,000"

## Turn learning off or limit it

Learning extracts rules and recalls them later | D/usage/13-learning.md:27-28 | "1. **Extraction** — after `/done` (or during auto-compaction), an LLM analyzes the conversation and extracts a small" / "  number of lessons from your corrections and stated rules."
 | D/usage/13-learning.md:364-371 | "- **`/done`** — extracts (if `supervisor.learning.enabled`) regardless of the compression result, and marks the session"
No model of its own | D/usage/13-learning.md:63-64 | "Learning does not own a separate model. Extraction, recall, verification, retention, and evolution all use" / "`[supervisor.model]`, which itself inherits omitted fields from `[model]`."
Off switch | D/usage/14-supervisor.md:469-475 | "[supervisor]" / "enabled = false" / "[supervisor.learning]" / "enabled = false"
Example validates | Test `[supervisor.learning]` + `enabled = false` | "Configuration is valid."
`/learning` commands | D/usage/13-learning.md:215-220 | "| `/learning` | List lessons (page 1). |" / "| `/learning delete <index>` | Delete a lesson by its **1-based index** in the current **unfiltered** hot list. Aliases: `rm`, `remove`. |" / "| `/learning clear` | Delete all hot and cold lessons for the current role + project scope; global rules are untouched. |"
Evolution off by default; compiles skills, guidance, guardrails; trial vs control | T:764-765 | "# Compile grounded memories into scoped skills and guardrail candidates." / "# Disabled by default: enable to allow detached synthesis and low-risk trials."
 | D/usage/13-learning.md:65-69 | "`[supervisor.learning.evolution]` is off by default (`enabled = false`). When enabled, the detached learner may" / "compile grounded memories from each extraction into machine-local skill, guidance, or guardrail candidates, at most one"
 | D/usage/13-learning.md:81 | "arm (behavior applied) and half to the **control** arm (trigger observed, behavior not applied), so both arms run over"
Evolution needs the gate | D/usage/13-learning.md:123 | "Evolution needs `[supervisor.gate]` enabled: without verdicts no arm collects samples, and trials end inconclusive."
Knobs | T:767-784 | "min_samples = 10" / "noise_margin = 0.15" / "cost_allowance = 0.10" / "cost_per_gain = 2.0" / "max_trial_uses = 40"
 | D/reference/03-config-reference.md:739-743 | "| `min_samples` | integer | `10` | Verify-gate verdicts in the shadow screen, then in each concurrent arm, before comparison |"
Inspect with `/learning evolution` | D/usage/13-learning.md:221 | "| `/learning evolution` | List evolved behavior matching the current project/domain. |"
Evolution switch on validates | Test sibling `[supervisor.learning.evolution]` + `enabled = true` | "Configuration is valid."

## Check tool calls against what the user asked

Off by default, needs supervisor | T:798-799 | "[supervisor.authorizer]" / "enabled = false"
 | D/reference/03-config-reference.md:774 | "Requires the supervisor master switch for root sessions."
Allow-first; blocks only a confirmed conflict | D/usage/14-supervisor.md:97-98 | "The rule is allow-first: if uncertain, allow. Reasonable investigation and intermediate steps are permitted. Only a" / "concrete prohibition or scope conflict can block a call."
 | D/usage/14-supervisor.md:135 | "Unavailable, malformed, over-budget, or timed-out judgments allow calls; failure of the judge is not evidence against"
Runs before the tool batch executes | D/usage/14-supervisor.md:102 | "Native pre-call guards run first. The authorizer checks the remaining batch before execution, including inline capability"
Block message | D/usage/14-supervisor.md:109 | "[authorizer] Tool not executed: Tests conflict with the user's restriction. user source user:0: \"Do not run tests\"."
One judge call per ordinary allow | D/usage/14-supervisor.md:103 | "activation and delegated tool loops. An ordinary allow needs one judge call. For a proposed block, the model selects a"
`/info` counts | D/usage/14-supervisor.md:121 | "the bounded denial cache is process-local. `/info` shows this session's checked, blocked, cached, and unavailable counts."
Switch on validates | Test sibling `[supervisor.authorizer]` + `enabled = true` | "Configuration is valid."

## Model-written hints and turn scoring

Advise: off by default, replaces the fixed note on loop/recovery alarm | T:813-821 | "# advise: on a loop or recovery alarm the steer note can carry content. One" / "[supervisor.advise]" / "enabled = false"
 | D/usage/14-supervisor.md:356-357 | "With `[supervisor.advise] enabled = true` (off by default), a loop or recovery alarm can carry content instead of the"
One in four is a control; logged to `advise.jsonl` | D/usage/14-supervisor.md:365-366 | "One in four eligible alarms is a control and gets the plain note, so the effect of advice can be measured. Every alarm is" / "appended to `advise.jsonl` in the data directory with its arm, signature, receipt, and later whether the call-set"
Cites a tool receipt | D/usage/14-supervisor.md:361-362 | "Tool output is untrusted data. The answer is an observation that cites a receipt (`#N`) and the question it raises. It is"
Hindsight: off, local, observe-only, logs, model is HF repo or directory | T:801-811 | "# hindsight: a local, calibrated verifier (octolib::hindsight, ONNX on CPU)" / "# `model` is a Hugging Face repo (files under onnx/; HF_TOKEN for a private" / "enabled = false"
 | D/reference/03-config-reference.md:678-680 | "An optional, local verifier scores the session event trace when the agent hands a turn back. It estimates whether the" / "user's next turn will be a correction and, if so, its class. This is observe-only: scores are logged, shown in a" / "supervisor notice, tallied in `/info`, and appended to `hindsight.jsonl` beside the verify-gate verdict."
 Needs supervisor on | S/supervisor/hindsight.rs:60 | "config.supervisor.enabled && config.supervisor.hindsight.enabled"
Switches validate | Test sibling advise / hindsight `enabled = true` | "Configuration is valid."

## Calibrated decisions

Keys, all flags off in the template | T:721-732 | "[decision]" / "model = \"cloudflare:typesafe/jev\"" / "recall = false" / "skills = false" / "capabilities = false" / "compression = false" / "supervisor = false"
Routing needs supervisor, non-empty model, matching flag; empty/absent model turns it off | D/usage/14-supervisor.md:146-147 | "and the shared `supervisor` flag default to false. Routing requires `supervisor.enabled`, a nonempty `decision.model`," / "and the corresponding flag. Absent or empty model disables routing; malformed nonempty models fail validation naming"
One attempt, short timeout, no retries; unavailable falls back | D/usage/14-supervisor.md:188-190 | "Every decision call is one attempt with a 5-second timeout, no retries, and a 24,000-token state cap. Missing keys," / "HTTP failures, transport errors, timeouts, over-cap states, and malformed or incomplete answers produce the seam's" / "unavailable outcome above."
Flag meanings and fallbacks | D/reference/03-config-reference.md:624-628 | "| `recall` | bool | `false` | Filter recalled scoped lessons for relevance. If unavailable, keep the candidates. |" / "| `capabilities` | bool | `false` | Confirm or promote capability selection. If unavailable, keep the cosine result. |" / "| `supervisor` | bool | `false` | Route supervisor decisions and evolution grading through this model. |"
Credentials from environment | T:719-720 | "# Keys: CLOUDFLARE_API_KEY + CLOUDFLARE_ACCOUNT_ID, TYPESAFE_API_KEY," / "# or OCTOHUB_API_KEY (+ OCTOHUB_API_URL), according to the provider."
Malformed model fails validation even with flags off | S/config/validation.rs:49-50 | "/// and it resolves the pair without touching the network. Checked even when" / "/// the supervisor is off: a typo here is a typo regardless of which switches"
 | Test `[decision] model = "foo"` | "Error: decision.model: invalid evaluation model 'foo'; expected provider:model"
Empty model validates | Test `[decision] model = ""` | "Configuration is valid."
`/info` per-seam counters | D/usage/14-supervisor.md:199 | "`/info` shows, per seam, `calls` (attempts), `unavailable` (unavailable decisions), and `applied` (recall: candidates excluded;"

## Tools appear on their own, or should not

`auto_capabilities` on by default; on matches triggers, off means manual enable | T:100-106 | "# Enable/disable automatic capability activation on each user message." / "# Disable to require manual `capability(action=\"enable\")` calls only." / "auto_capabilities = true"
Other activation paths remain | D/usage/16-token-efficiency.md:186-187 | "pre-request automatic path. Named enable calls, env preloading, skill-declared capabilities, and the explicit `tap`" / "capability-intent action remain separate activation paths. The latter accepts this arguments object on the `tap` MCP"
Env preload | D/usage/16-token-efficiency.md:347 | "- **Force-loading at boot.** `OCTOMIND_CAPABILITIES=project-read octomind run developer:general` force-activates the"
Fewer bundles, fewer schemas | D/usage/16-token-efficiency.md:19-20 | "Capabilities are bundles from installed taps. Loading fewer optional bundles can reduce tool-schema context; the amount"
On: each user message is matched to capability triggers | D/usage/16-token-efficiency.md:162-163 | "Asking the model \"do you need a database tool?\" before every turn would burn a routing turn for every message. Instead," / "Octomind embeds the user's message and matches it against the hand-authored triggers in each capability's `config.toml`."
`[capabilities]` maps name to provider; empty keeps defaults | T:131-134 | "# Capability provider overrides for tap agents" / "# Each key is a capability name, value is the provider to use instead of \"default\"" / "# Example: codesearch = \"octocode\" → uses capabilities/codesearch/octocode.toml" / "# Leave empty to use default providers for all capabilities"
Decision second opinion | D/usage/16-token-efficiency.md:164-165 | "No LLM in the routing loop. With `[decision] capabilities = true` the cosine decision gets a second opinion" / "from the calibrated evaluation model: one Choice over the five best-scored candidates plus `none` can veto a winner it"
Verify | D/usage/16-token-efficiency.md:382-384 | "/loglevel debug" / "/mcp full"
Example validates | Test `auto_capabilities = false` and `[capabilities]` `codesearch = "octocode"` | "Configuration is valid."

## Skills

Switches and defaults | T:580-602 | "auto_activation = true" / "auto_validation = false" / "validation_timeout = 60" / "max_retries = 3"
Skills with matching rules load on a user message | T:581-582 | "# Enable automatic skill activation via declarative rules in SKILL.md frontmatter." / "# When enabled, skills with `domains` matching the current agent's role"
`auto_validation` runs `validate` scripts of active skills after each turn | T:586-587 | "# Enable automatic validation via `validate` scripts at end of each assistant turn." / "# When enabled, active skills with a `validate` script are run automatically."
Off means manual `/skill` only | T:583 | "# are checked on each user message. Disable to use manual /skill only."
`auto_validation` gates only `validate` scripts, not guardrail validators | D/reference/03-config-reference.md:501 | "> **`auto_validation` scope:** this flag gates only the `validate` scripts declared inside `SKILL.md` files. It does **not** gate the separate guardrail `[[validator]]` system"
`validation_timeout` `0` unlimited; `activation_timeout` reserved | T:591-595 | "# Reserved. Rules are evaluated in-process (no script timeout needed)." / "# Timeout in seconds for `validate` scripts. 0 = unlimited."
Validates | Test `[skills]` `auto_activation = false` / `auto_validation = true` | "Configuration is valid."

## Requests time out or stall

Total deadline and inactivity limit, `0` disables | T:124-127 | "# Total deadline for one HTTP or WebSocket request attempt; 0 disables it." / "# SSE/WebSocket response inactivity limit; 0 disables it. Does not extend the total deadline."
Retry settings | T:121-123 | "# Retry count and exponential-backoff base delay." / "max_retries = 1" / "retry_timeout = 30"
Transport values, scope | S/config/model.rs:30-34 | "pub enum RequestTransportConfig {" / "Http," / "WebSocket,"
 | Test `request_transport = "tcp"` | "unknown variant `tcp`, expected `http` or `websocket`"
 | T:128 | "# Persistent WebSocket for octohub/chatgpt requests (\"http\" opts out); other providers ignore it."
No silent fallback to HTTP | D/reference/03-config-reference.md:149-150 | "calls that inherit it. Set `request_transport = \"http\"` to opt out. WebSocket errors do not silently fall back to" / "HTTP or replay a request."
Supervisor and compression profiles carry their own timeout values | T:746-757 | "[supervisor.model]" / "request_timeout_seconds = 300" / "stream_timeout_seconds = 0"
 | T:690-706 | "[compression.model]" / "request_timeout_seconds = 300" / "stream_timeout_seconds = 0"
They inherit unset fields (so `request_transport`, absent from both) from `[model]` | S/config/model.rs:212 | "request_transport: self.request_transport.unwrap_or(base.request_transport),"
 | D/reference/03-config-reference.md:148-149 | "`request_transport` applies to every `octohub:` and `chatgpt:` model call, including supervisor and compression" / "calls that inherit it."
Tool timeout is per MCP server, idle deadline, 1..3600 | T:358-359 | "# Per-operation timeout. For tool calls this is an idle deadline: each MCP" / "# progress notification resets it. Set a larger explicit value only for a"
 | S/config/validation.rs:156-163 | "if server_config.timeout_seconds() == 0 {" / "if server_config.timeout_seconds() > 3600 {"
Example validates | Test `[model]` `stream_timeout_seconds = 60`, `request_timeout_seconds = 600`, `request_transport = "http"` | "Configuration is valid."

## Sandbox

Key and meaning | T:28-33 | "# Sandbox mode: restrict all filesystem writes to the current working directory" / "# When enabled, the process and all child processes (shell, MCP servers) cannot" / "sandbox = false"
Linux and macOS | T:31 | "# Supported on Linux (Landlock, kernel 5.13+) and macOS (Seatbelt)."
`--sandbox` for run, acp, server | S/main.rs:320-322 | "Commands::Run(a) => config.sandbox || a.sandbox," / "Commands::Server(a) => config.sandbox || a.sandbox," / "Commands::Acp(a) => config.sandbox || a.sandbox,"
Unenforced on older Linux kernel, says so | S/sandbox/linux.rs:97 | "\"Sandbox requested but Landlock is not enforced on this kernel — running without restrictions.\""
Verification log line | S/sandbox/macos.rs:101 | "crate::log_info!(\"Sandbox active (Seatbelt): writes locked to {}\", dir.display());"
 | S/sandbox/linux.rs:85 | "\"Sandbox active (Landlock fully enforced): writes locked to {}\","
Validates | Test `sandbox = true` in a sibling | "Configuration is valid."

## Other settings

`[registry] cache_ttl_hours`, optional | S/config/registry.rs:25-27 | "/// How long to cache fetched manifests before re-checking (hours)" / "#[serde(default = \"default_cache_ttl_hours\")]" / "pub cache_ttl_hours: u64,"
Stale copy served while refreshing | D/reference/03-config-reference.md:886-888 | "Fetched manifests are cached at `<data>/agents/<category>/<variant>.toml`. Within the TTL the cached manifest is served" / "immediately; once stale, the cached copy is still served (stale-serve) while a background refresh fetches the latest"
`[timing]` calibrates the `/report` estimate | D/reference/03-config-reference.md:505 | "Per-person calibration of the human-time and energy estimate in `/report`. The defaults are starting guesses; see"
 | S/config/mod.rs:257 | "/// Per-person calibration of the `/report` human-time and energy estimate."

## Conflicts between docs and code (code followed)

`compression.threshold` default | D/usage/08-compression.md:8 ; T:664 | "Compression is enabled by default at `105000` tokens." ; "threshold = 105000"
`max_session_tokens_threshold` default | D/usage/08-compression.md:231 ; T:59 | "(root config, default `300000`)" ; "max_session_tokens_threshold = 300000"
Config version in reference | D/reference/03-config-reference.md:70 | "| `version` | u32 | `22` | Config version." — T:10 "version = 29"
`config --validate` on unknown keys | task brief said "warns" | S/commands/config.rs:111-120 | "if !ignored_reports.is_empty() {" ... "anyhow::bail!(\"{err_msg}\");" (warns, then exits with an error)
Evolution fragment | D/usage/13-learning.md:55-56 | "[supervisor.learning.evolution]" / "enabled = false" (valid only as a sibling; alone in config.toml it fails on `min_samples`, per Test above)
`learning.enabled` missing | D/usage/13-learning.md:61 | "| `enabled` | Enable the learning system. | `true` |" — S/supervisor/learning/mod.rs:260 "#[serde(default)]" (a missing key is `false`)
