# Guardrails

Lets a project change user input, block tool calls, and push corrective messages to the model, using scripts and rules in one file.

All four extensions live in `<workdir>/.agents/guardrails.toml`. Only `[[pipe]]` and `[[guard]]` can block. `[[hook]]` and `[[validator]]` only queue a message that the model reads on a later request; they cannot undo anything. Order in one turn: pipe, model, guards (before tools run), tools, hooks, final answer, validators.

## pipe

Where it goes: a `[[pipe]]` table in `<workdir>/.agents/guardrails.toml`; the script can be anywhere (path relative to the workdir, or absolute).

Minimal valid example:

```toml
[[pipe]]
name = "prepare"
command = "./scripts/prepare-input.sh"
```

The script reads the user message on stdin and prints the replacement message on stdout.

Required fields and allowed values:
- `name`: non-empty, unique among pipes.
- `command`: path to one executable. No arguments, no shell string.
- `match` (optional): regex on the user message.
- `when` (optional): `"first"` (first message only) or `"any"`.
- `roles` (optional): list; `"developer:general"` matches exactly, `"developer"` matches the whole domain. `"developer:*"` is not a glob.

How it activates: rules load at session start, so start a new session after any edit. The pipe runs on each user message that passes `roles`, then `when`, then `match`. At most one pipe may match a message; two matches stop the message with an error. Exit 0 replaces the message with stdout (used as is, not trimmed). Any non-zero exit, a 300 second timeout, or a spawn failure stops the message; stderr is shown in the error. Environment variables set: `OCTOMIND_ROLE`, `OCTOMIND_WORKDIR`, `OCTOMIND_SCRIPT_DIR`, `PIPE_NAME`, `PIPE_RUN_COUNT`, `SESSION_MESSAGE_COUNT`.
How to verify:
- Script alone: `printf 'hi' | ./scripts/prepare-input.sh`
- In Octomind (sends a prompt): `printf '%s\n' 'Explain the project rule added to this request.' | octomind run developer:general --format jsonl`

Pitfalls:
- A catch-all pipe plus any other pipe that matches the same message blocks every such message. Narrow with `match`, `when`, or `roles`.
- Known slash commands such as `/exit` are handled separately; a pipe cannot forbid them.

## guard

Where it goes: a `[[guard]]` table in `<workdir>/.agents/guardrails.toml`. No script.

Minimal valid example:

```toml
[[guard]]
match = 'shell(command=^rm\s+-rf?\b)'
message = "Recursive deletion is denied in this project."
```

Required fields and allowed values:
- `match`: a target, one of `capability`, `capability(regex)` (regex on the whole arguments JSON), or `capability(arg=regex)` (regex on one argument; strings match without quotes, other values match their JSON text).
- `message`: text the model sees.
- `has` (optional): string or list of MCP server names that must be configured for the role, for example `"filesystem"`. Not capability names.
- `when` (optional): list of `"+target"` (a matching call was allowed earlier this session) or `"-target"` (none was). All entries must hold.

The capability in a target is the exact name of the tap capability that owns the tool (`shell`, `filesystem-read`), found from the capabilities' `allowed_tools`. It is not the tool name or server name. A tool with no owning capability never matches.

How it activates: new session after an edit. Guards are checked in file order before any tool in a batch runs, and the first match wins. A blocked call does not run; the model gets the tool error `[guardrail] <message>`. `when` history counts allowed attempts, not successes.
How to verify: check TOML syntax (see the last section), then in a session ask the model to make the call and look for `[guardrail]` in the tool result. To learn a tool's capability name, add a temporary hook with `on = "any"` and no `match` whose script prints the `capability` field of its stdin JSON to stderr and exits 1; run `/loglevel debug` and make a call.

Pitfalls:
- Matching on a tool name or server name silently never fires.
- `has` with a capability name never fires unless a server has that exact name.
- The regexes see the literal argument text, not shell meaning; `rm -rf` variants are not all covered.

## hook

Where it goes: a `[[hook]]` table in `<workdir>/.agents/guardrails.toml`; the script path is relative to the workdir or absolute.

Minimal valid example:

```toml
[[hook]]
match = "shell(command=cargo (build|test|check))"
result = "error\\["
script = ".agents/hooks/cargo-summary.sh"
```

Required fields and allowed values:
- `script`: required.
- `match` (optional): same target syntax as `[[guard]]`; empty means any tool.
- `result` (optional): regex on the result text.
- `on` (optional): `"success"`, `"error"`, or `"any"`.

How it activates: new session after an edit. After each tool result, every matching hook runs (no first-match rule). Hooks are skipped for calls a guard blocked. The script gets JSON on stdin: `capability` (null if unowned), `tool`, `tool_id`, `params`, `result`, `success`. Exit 0 does nothing. Non-zero exit with non-empty trimmed stdout sends that stdout to the model as feedback; empty stdout sends nothing. Timeout is 300 seconds and gives no feedback. Env: `OCTOMIND_CAPABILITY`, `OCTOMIND_TOOL`, `OCTOMIND_SUCCESS` (1 or 0), `OCTOMIND_WORKDIR`, `OCTOMIND_SCRIPT_DIR`.

How to verify:
- Script alone: `printf '%s' '{"capability":"shell","tool":"shell","tool_id":"t1","params":{"command":"cargo check"},"result":"error[E0308]","success":false}' | .agents/hooks/cargo-summary.sh; echo $?`. In a session, pipe `octomind run <role> --format jsonl` through `jq 'select(.type == "injected" and .source_kind == "guardrail_hook")'`.

Pitfalls:
- Feedback must go to stdout. Text on stderr is only logged at debug level.
- A hook that exits 0 never reaches the model, however much it prints.

## validator

Where it goes: a `[[validator]]` table in `<workdir>/.agents/guardrails.toml`.

Minimal valid example:

```toml
[[validator]]
name = "test-after-edit"
when = ["+filesystem-write", "-shell(command=cargo test)"]
script = ".agents/validators/remind-tests.sh"
```

Required fields and allowed values:
- `name`: non-empty, unique among validators.
- `script`: required.
- `match` (optional): regex on the assistant's final message.
- `when` (optional): signed targets as in `[[guard]]`, checked only against calls since this validator last ran.
- `roles` (optional): same matching as `[[pipe]]`.

How it activates: new session after an edit. At the end of every assistant turn, validators are tried in file order: `roles`, then `when`, then `match`. With no filters it runs every turn. A validator that passes spawns its script and its history position advances at once, even if the script fails or times out. The script gets JSON on stdin: `validator`, `role`, `assistant_text`, `triggered_by`. Non-zero exit with non-empty trimmed stdout queues `<validation validator="name">stdout</validation>` for the model. Timeout is 300 seconds. There is no retry cap, and `[skills].auto_validation` does not affect it.
How to verify:
- Script alone: `printf '%s' '{"validator":"test-after-edit","role":"developer:general","assistant_text":"done","triggered_by":[]}' | .agents/validators/remind-tests.sh; echo $?`. In a session, pipe `octomind run <role> --format jsonl` through `jq 'select(.type == "injected" and .source_kind == "guardrail_validator")'`.

Pitfalls:
- A script that keeps printing feedback after the issue is fixed nags every turn. Make it exit 0 once satisfied.
- `-shell(command=cargo test)` means no matching call was attempted, not that tests passed.

## Loading, reload, and errors

- A missing file means no rules, silently.
- Any parse error (bad TOML, bad regex, `when` entry not starting with `+` or `-`, empty or duplicate `name`, missing required field) drops every authored rule for the session. Octomind prints `guardrails: failed to parse <path>: <reason>` to stderr and keeps running.
- Misspelled optional keys such as `role` for `roles` are not rejected; the parser has no unknown-key check.
- `octomind config --validate` does not read this file. Check TOML syntax with `python3 -c 'import tomllib; tomllib.load(open(".agents/guardrails.toml","rb"))'`; that does not check regexes or `when` signs, so start a session and watch stderr.
- Scripts are copied when rules load. Editing a script mid-session changes nothing until the next load, and a script missing at load stays missing. Use `$OCTOMIND_SCRIPT_DIR`, not `dirname "$0"`, to find files next to the script.
- Reload happens at session start. `/done` also reloads, but only when `supervisor.learning.enabled` is true.
- Make scripts executable (`chmod +x`).
