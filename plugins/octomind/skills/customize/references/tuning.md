# Tuning Octomind: completion checks, cost, compression, learning, tools and timeouts

This file lets you pick the Octomind setting that matches a user's goal and change it without breaking the config. Checked against Octomind 0.70.0.

## How to change a setting

- Octomind merges every `*.toml` file in the config directory `<data-dir>/config/`. The data directory is `~/.local/share/octomind` unless `OCTOMIND_DATA_DIR` moves it; if `OCTOMIND_CONFIG_PATH` is set, the directory is that file's parent.
- `config.toml` loads first, the other files follow alphabetically, and later files win. Tables merge key by key, scalars are replaced, so a sibling file such as `90-tuning.toml` holding only the changed key is valid. This was tested: a file with only `[supervisor.condense]` and `tokens_threshold` validates.
- Keep each key in the table that owns it. A root key (`sandbox`, `auto_capabilities`, the spending and token thresholds) must come before the first `[table]` header of its file. After a header it becomes a key of that table and `octomind config --validate` rejects it as unknown.
- Do not delete keys from `config.toml`. Octomind does not fill missing keys from its template, and a missing required key fails loading with `missing field`. To disable a mechanic set `enabled = false`; never delete its table.
- Required, even when the mechanic is off: root `sandbox`, `auto_capabilities`, `cache_keepalive_enabled`, `commit_attribution`, both spending thresholds, both token thresholds; every key of `[model]`, `[skills]`, `[timing]`; `[decision]` keys `recall`, `skills`, `capabilities`, `compression`, `supervisor`.
- Required in `[supervisor]`: `enabled`, `max_identical_rounds`, and the tables `learning`, `gate`, `authorizer`, `plan`, `condense`, `hindsight`, `advise`, each with all its keys. `learning` also needs `[supervisor.learning.evolution]` with `enabled`, `min_samples`, `noise_margin`, `cost_allowance`, `cost_per_gain`, `max_trial_uses`. Required in `[compression]`: `threshold`, `idle`, `analysis_findings_max_tokens`.
- May be missing: `[supervisor.model]`, `[compression.model]`, `compression.knowledge_retention`, `[compression.attention]`, `decision.model`, `[capabilities]`, `[taps]`, `[registry]`, `telemetry`. Exception: a missing `supervisor.learning.enabled` is read as `false`, not as the template's `true`, so keep it.
- Do not edit `version`. When it is behind the binary, Octomind upgrades the file on start, copies missing keys from the template and writes a `.bak` backup. A same-version file is never repaired.
- Values must have the right type: integers for counts, unquoted `true`/`false`, quoted strings for names, no negative counts. `[decision]` rejects unknown keys.
- One run only: `octomind -c supervisor.gate.enabled=false run ...`. The key must already exist, the value is parsed as TOML, and `-c` is refused with `config`. `config --validate` does not check `-c` values; any other command, such as `octomind vars`, loads and checks them.
- Afterwards run `octomind config --validate`. It prints `warning: unknown configuration key` for each key Octomind ignores and then exits with an error, so treat a warning as a failure. Changes apply to the next session.
- `octomind config --show` prints only a summary (model, theme, MCP servers); it does not show these settings. Read the files, or use `/info` inside a session.
- Shipped values change between releases. For the current number, look in `config-templates/default.toml` in the Octomind repository, or run `octomind config --validate` with a fresh `OCTOMIND_DATA_DIR=$(mktemp -d)` and read the `config.toml` it creates from the installed binary's template. This file gives no numbers.

## Make done mean done (stop it finishing early)

- Keys: `[supervisor] enabled`, `[supervisor.gate] enabled`, `[supervisor.gate] unsettled_advisory`, `[supervisor.plan] enabled`; all booleans, on in the 0.70.0 template.
- With the supervisor and gate on, every stop of a turn (a `done` claim, or a `blocked`/`need_input` hand-back) is checked by a separate model pass against your request. A concrete gap sends the agent back to work, within a fixed retry budget; a hand-back passes only if the remaining work truly needs the user.
- Gate off: no stop gets that model pass. `unsettled_advisory` adds one non-blocking note per user turn about evidence still unsettled. `plan.enabled` lets the supervisor create and track a plan for dependent multi-phase work.
- The gate judges evidence; it does not run tests. For a hard check use a guardrail validator in `.agents/guardrails.toml`, which is a separate file.
- Verify: `/info` shows supervisor activity, `/plan` shows the plan, `/loglevel debug` explains why a turn re-ran.

## Stop runaway loops sooner, or turn the stop off

- Key: `[supervisor] max_identical_rounds`, an integer; `0` turns the stop off. It needs `enabled = true`.
- After that many tool rounds in a row that repeat calls and results already seen, the turn ends with a notice naming the repeated call. No further model request is made and the run is marked failed for learning. Lower hint thresholds are fixed and not configurable.

## Cap spending

- Keys (root, before any table): `max_request_spending_threshold` and `max_session_spending_threshold`, USD as decimals; `0.0` disables.

```toml
max_request_spending_threshold = 0.5
max_session_spending_threshold = 5.0
```

- Request: execution stops when the cost since the user's request began reaches the value. Session: Octomind asks `Do you want to continue? (y/N)` once the cost since the last accepted checkpoint reaches the value; piped, ACP and WebSocket runs decline and stop.
- Both compare costs already recorded, so an in-flight request can overshoot. They are not a provider billing limit. Verify with `/info` and `/report`; a stop prints `REQUEST SPENDING THRESHOLD EXCEEDED` or `SPENDING THRESHOLD REACHED`.

## Lower cost

- The supervisor's model calls (gate, plan, condense, learning, authorizer, advise) share one profile, `[supervisor.model]`; omitted fields inherit `[model]`. Pointing `name` at a cheaper `provider:model` lowers all of them at once; `reasoning_effort` accepts `none`, `low`, `medium`, `high`, `xhigh`, `max`.
- The same applies to `[compression.model]`, which writes the compression summaries.
- Largest switch: `[supervisor] enabled = false` removes the gate, plan manager, condensing, loop stop and the other supervisor mechanics. Also set `[supervisor.learning] enabled = false`, because learning checks its own switch.
- `cache_keepalive_enabled = true` is opt-in: it sends pings that cost cache-read tokens to avoid a cache rewrite; only providers with a keepalive policy are pinged.
- Verify: `/info` (model usage and cost per mechanic), `/usage`, `/report`.

## Compress later or sooner

```toml
[compression]
threshold = 150000
```

- Key: `[compression] threshold`, in tokens. Compression becomes eligible once the full context exceeds it; higher means fewer, later folds; `0` disables automatic compression (`/done` still compresses). Depth is computed from the session's growth; there is no ratio key.
- The effective limit is also capped by the context ceiling, the lower of root `max_session_tokens_threshold` and the model's usable window. Raising `threshold` cannot fix a single request that is too large.
- `idle` (bool): compact when you return after the provider's prompt cache expired, only if the cost math says it saves money. `knowledge_retention` keeps that many critical-knowledge entries; `analysis_findings_max_tokens = 0` drops retained findings.
- `[compression.attention] enabled = true` switches on attributed summaries. In one file, write scalar `[compression]` keys before nested `[compression.*]` headers.
- Verify: `/info` shows runs, messages removed and tokens saved; `/loglevel debug` shows why a fold fired or not.

## Shrink or cut large tool results

- `mcp_response_tokens_threshold` (root): truncates each tool response above it; `0` = unlimited.
- `[supervisor.condense]`: `enabled`; `tokens_threshold` (per result; `0` turns condensing off); `adaptive` (learns a trigger within a bounded range). A result above the threshold is narrowed to the lines the task needs, and the original is saved to a file the agent can read. Keep `tokens_threshold` well below the truncation limit.
- `max_session_tokens_threshold` (root): your half of the context ceiling; `0` removes the configured limit, the model window still applies; above the allowed maximum loading fails.
- Nothing condensed? Results under the threshold, non-text payloads, or a role without a file-reading tool stay unchanged.

## Turn learning off or limit it

```toml
[supervisor.learning]
enabled = false
```

- Learning extracts rules you stated or corrected (after `/done`, on exit and on automatic compression) and recalls them in later sessions. It has no model of its own; it uses `[supervisor.model]`.
- Keep `[supervisor.learning.evolution]` in the file. `/learning list`, `/learning delete <n>` and `/learning clear` inspect and remove stored lessons for the current role and project.
- Evolution (`[supervisor.learning.evolution] enabled`, off by default) lets the learner compile grounded memories into skill, guidance or guardrail candidates and trial them against a control group. It needs the gate on. Leave `min_samples`, `noise_margin`, `cost_allowance`, `cost_per_gain`, `max_trial_uses` alone unless you want stricter or looser promotion. Inspect with `/learning evolution`.

## Check tool calls against what the user asked

- Key: `[supervisor.authorizer] enabled` (off by default). It needs the supervisor on. Before a tool batch runs, a model compares the calls with the role and the user's words and blocks only a confirmed conflict; if unsure or unavailable it allows. A block appears as `[authorizer] Tool not executed: ...`.
- An ordinary allow costs one judge call. `/info` shows checked, blocked and cached counts.

## Model-written hints and turn scoring

- `[supervisor.advise] enabled` (off by default): on a loop or recovery alarm one model call writes a short observation citing a tool receipt instead of the fixed note. One alarm in four gets the fixed note as a control. Every alarm is logged to `advise.jsonl` in the data directory.
- `[supervisor.hindsight] enabled` (off by default, needs the supervisor on): a local model scores whether the user's next turn will be a correction. Observe-only; scores go to `/info` and `hindsight.jsonl`. `model` is a Hugging Face repository or a local export directory.

## Calibrated decisions (`[decision]`)

- Keys: `model` (`provider:model`, empty or absent turns routing off) and five booleans, all off in the template: `recall`, `skills`, `capabilities`, `compression`, `supervisor`.
- A flag takes effect only with `[supervisor] enabled = true`, a non-empty `model` and that flag set. Each call is one attempt with a short timeout and no retry; if the answer is unavailable the mechanic falls back as its flag describes (for example `capabilities` keeps the embedding choice). Credentials come from environment variables of the chosen provider.
- A badly formed `model` fails validation even when every flag is off. `/info` shows calls, unavailable and applied counts per seam.

## Tools appear on their own, or should not

```toml
auto_capabilities = false
```

- Key: `auto_capabilities` (root bool, on by default). On: each user message is matched to capability triggers and matching tool bundles load. Off: bundles load only by an explicit `capability` call, `OCTOMIND_CAPABILITIES`, a skill, or the `tap` capability action. Fewer bundles mean fewer tool schemas in the prompt.
- `[capabilities]`: maps a capability name to the provider file it should use, for example `codesearch = "octocode"`. An empty table keeps each default.
- `[decision] capabilities = true` adds a second opinion on the match. Verify with `/mcp full`; `/loglevel debug` shows scores and why nothing activated.

## Skills activate or validate on their own

- `[skills] auto_activation` (on): skills whose rules match the user message load; off means manual `/skill` only. `auto_validation` (off): runs the `validate` scripts of active skills after each turn; it does not control guardrail validators. `validation_timeout` (seconds, `0` = unlimited), `max_retries`. `activation_timeout` is reserved.

## Requests time out or stall

```toml
[model]
stream_timeout_seconds = 60
request_timeout_seconds = 600
request_transport = "http"
```

- In `[model]`: `request_timeout_seconds` is the total deadline for one HTTP or WebSocket attempt (`0` disables). `stream_timeout_seconds` limits silence on an SSE or WebSocket response (`0` disables) and never extends the total deadline. `max_retries` and `retry_timeout` set retry count and the backoff base in seconds.
- `request_transport` is `"websocket"` or `"http"` and applies to `octohub:` and `chatgpt:` models only. `"http"` opts out of WebSocket; WebSocket errors are not silently retried over HTTP.
- `[supervisor.model]` and `[compression.model]` carry their own timeout values in the template, so change them there too. They inherit `request_transport` from `[model]`.
- Tool timeouts are per server: `timeout_seconds` in `[[mcp.servers]]`, an idle limit for tool calls, valid from 1 to 3600.

## Restrict writes (sandbox)

- Key: `sandbox = true` (root), or `--sandbox` for one `run`, `acp` or `server` command. Writes by Octomind, its shell and its MCP servers are limited to the launch directory. Linux uses Landlock (kernel 5.13 or newer), macOS uses Seatbelt; on an older Linux kernel it runs unrestricted and says so.
- Verify: the info-level log line `Sandbox active`.

## Other settings

- `[registry] cache_ttl_hours`: hours a fetched tap manifest is trusted before a background refresh; the stale copy is served meanwhile.
- `[timing]`: six numbers that calibrate the `/report` time and energy estimate; they do not affect agent behavior.
