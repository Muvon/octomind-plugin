# Evidence for references/webhooks-and-scheduling.md

`D` = `doc/` and `S` = `src/` in /Users/dk/Work/dev/muvon/octomind (Octomind 0.70.0, commit 25f9ae4e); line numbers as of 2026-11-08.
`T` = run against the installed `octomind` 0.70.0 with a throwaway `OCTOMIND_DATA_DIR`; the `[[hooks]]` example was added as `config/hooks.toml` and `octomind config --validate` printed "Configuration is valid." No session, hook listener or schedule was started, because that sends a prompt. Runtime behaviour is from docs and code only.
Short names: `D/03` = integration/03-daemon-and-hooks.md, `D/09` = use-cases/09-custom-hooks.md, `D/07s` = use-cases/07-scheduled-tasks.md, `D/07m` = usage/07-mcp-tools.md.

claim | source | verbatim quote

## Webhook listeners
Config dir is `<data-dir>/config`, default `~/.local/share/octomind`, moved by `OCTOMIND_DATA_DIR` | D/07m:610-611 | "`~/.local/share/octomind/config/`; on Windows it is `%LOCALAPPDATA%/octomind/config/`. `OCTOMIND_DATA_DIR` relocates"
Hook fields | S/config/hooks.rs:24-32 | "pub name: String," / "pub bind: String," / "pub script: String," / "pub timeout: u64,"
`timeout` is optional | S/config/hooks.rs:31 | "#[serde(default = \"default_hook_timeout\")]"
Names and binds must be unique | S/config/validation.rs:93-94,99-105 | "return Err(anyhow!(\"Duplicate hook name: '{}'\", hook.name));" ; "\"Hook '{}' has duplicate bind address '{}' (already used by another hook)\","
`bind` must parse as an IP and port, a hostname is rejected | S/config/validation.rs:106-112 | "if hook.bind.parse::<std::net::SocketAddr>().is_err() {"
Hostname rejected, confirmed | T | "Error: Hook 'bad-host' has invalid bind address: 'localhost:9877'"
`timeout` must be 1 to 3600 | S/config/validation.rs:116-125 | "\"Hook '{}' timeout must be > 0\"" ; "\"Hook '{}' timeout too high: {}s (max 3600)\","
Zero timeout rejected, confirmed | T | "Error: Hook 't' timeout must be > 0"
`config --validate` does not check the script file | T | A hook with `script = "/home/me/hooks/build-event.sh"` (file absent) validated: "Configuration is valid."
Script checked only when the hook is activated | D/03:137-140 | "When a hook is activated, it is validated again before binding: the bind address must parse, the script must exist and be a regular file, and on Unix it must be executable (`chmod +x`). Any failure aborts session start"
Missing or non-executable script | S/session/webhook_listener.rs:80,91 | "anyhow::bail!(\"Hook '{}': script '{}' does not exist\", hook.name, hook.script);" ; "anyhow::bail!(\"Hook '{}': script '{}' is not executable\", hook.name, hook.script);"
Unknown hook name aborts startup | S/session/chat/session/main_loop.rs:177 | "\"Hook '{}' not found in config. Define it in [[hooks]].\""
Script is run directly, not through a shell | D/09:97 | "Executable file path; invoked directly, without a shell command parser"
Same in code | S/session/webhook_listener.rs:224 | "let mut cmd = Command::new(script_path);"
Relative script path resolves from the launch directory | D/03:119-120 | "Relative script paths resolve from" / "the process launch directory; use an absolute path for service deployments."
Only POST runs the script, others get 405 | S/session/webhook_listener.rs:180-181 | "if req.method() != hyper::Method::POST {" / "return Ok(response(StatusCode::METHOD_NOT_ALLOWED, \"Only POST allowed\\n\"));"
Body arrives on stdin | D/03:174 | "| **stdin** | Raw HTTP body |"
Exit 0 with non-empty stdout injects trimmed stdout, answers 200 `ok` | S/session/webhook_listener.rs:277,300 | "let content = String::from_utf8_lossy(&output.stdout).trim().to_string();" ; "Ok(response(StatusCode::OK, \"ok\\n\"))"
Injected as a user message | D/03:180-181 | "stdout is **trimmed** (leading/trailing whitespace stripped) and injected as a user message."
Empty stdout is 204, nothing injected | S/session/webhook_listener.rs:278-280 | "if content.is_empty() {" / "return Ok(response(StatusCode::NO_CONTENT, \"\"));"
Non-zero exit is 500, stderr logged | S/session/webhook_listener.rs:301-312 | "\"Webhook '{}': script exited with code {}: {}\"," ; "StatusCode::INTERNAL_SERVER_ERROR,"
Timeout kills the script, 504 | S/session/webhook_listener.rs:229-230,323-325 | ".kill_on_drop(true)" ; "Ok(response(StatusCode::GATEWAY_TIMEOUT, \"Script timeout\\n\"))"
Body over 10 MiB gets 413 | S/session/webhook_listener.rs:44,212 | "const MAX_BODY_BYTES: usize = 10 * 1024 * 1024;" ; "return Ok(response(StatusCode::PAYLOAD_TOO_LARGE, \"Payload too large\\n\"));"
Environment variables | S/session/webhook_listener.rs:231-236 | ".env(\"HOOK_NAME\", hook_name)" / ".env(\"HOOK_METHOD\", &method)" / ".env(\"HOOK_PATH\", &path)" / ".env(\"HOOK_QUERY\", &query)" / ".env(\"HOOK_CONTENT_TYPE\", &content_type)" / ".env(\"HOOK_SESSION\", session_name);"
Header variables, uppercase with underscores | S/session/webhook_listener.rs:198 | "let env_name = format!(\"HOOK_HEADER_{}\", name.as_str().to_uppercase().replace('-', \"_\"));"
No signature check or TLS | D/03:228-229 | "The listener does not validate webhook signatures or provide" / "TLS/authentication."
`200` means queued, not answered | D/03:230 | "Hook `200` acknowledges enqueueing, not completion of the AI's work."
`[[hooks]]` differs from guardrail `[[hook]]` | D/09:7-9 | "> **`[[hooks]]` vs `[[hook]]` — two different systems.** This doc covers **HTTP webhook hooks** (`[[hooks]]`, plural, in" / "the main config)"
Listeners start only with `--hook <NAME>`, repeatable | D/03:19 | "| `--hook <NAME>` | Activate a configured webhook listener. Repeatable. |"
`run` defines `--hook` | S/commands/run.rs:59-62 | "/// Activate a webhook hook by name (defined in [[hooks]] config)." / "#[arg(long = \"hook\", value_name = \"NAME\")]"
`run` starts listeners for each `--hook` | S/session/chat/session/main_loop.rs:173-181 | "for hook_name in &args.hooks {" / "crate::session::webhook_listener::start_webhook_listener(session_name, hook_config, addr, script_path)"
Listeners start for interactive and non-interactive runs | D/09:109 | "Hook listeners start for both interactive and non-interactive `octomind run` sessions and stop when the session ends."
Listener started by `init_session_runtime`, called in the run paths | S/session/chat/session/main_loop.rs:83,263,1471 | "let webhooks = start_webhook_guards(args, config, &chat_session.session.info.name).await?;" ; "let mut runtime_guards = init_session_runtime(args, config, &chat_session, &role).await?;" ; "let _runtime_guards = init_session_runtime(args, config, &chat_session, &role).await?;"
`acp` accepts `--hook` | S/commands/acp.rs:48-51 | "/// Activate a webhook hook by name (defined in [[hooks]] config)." / "pub hooks: Vec<String>,"
`acp` forwards the names but the ACP session path never calls `init_session_runtime` | S/acp/agent.rs:367,1097-1099 | "hooks: self.hooks.clone()," ; "crate::session::context::init_session_services(&role_for_pool);" (no call to `init_session_runtime` or `start_webhook_guards` in `src/acp/`)
Docs agree: `acp` accepts the flag, starts no listener | D/03:56-57 | "accepts a `--hook` flag but does not start a webhook listener from it, and it exposes no `octomind send` socket."
`server` has no `--hook` | D/03:53-54 | "over the WebSocket connection; there is no inject socket and no `--hook` support."
`server` rejects the flag, confirmed | T | `octomind server --hook x` printed "error: unexpected argument '--hook' found"
`acp` lists the flag in its help, confirmed | T | `octomind acp --help` printed "--hook <NAME>         Activate a webhook hook by name (defined in [[hooks]] config)."
`--daemon` keeps the run alive; without it a non-interactive run exits when idle | D/03:44-45 | "Without `--daemon`, a non-interactive run still waits for pending schedules, monitors, agents, detached shell jobs, and" / "tap runs; it exits once that work is exhausted."
Piped first prompt must be non-empty | S/commands/run.rs:177 | "anyhow::bail!(\"No input provided via stdin\");"
`--name` is how you choose the session name | D/03:16 | "Provide a stable name if you want to inject messages with `octomind send`."
Verify with curl, expect 200 and `ok` | D/09:59 | "Expect HTTP `200` with body `ok`."
jsonl output shows an `injected` record from the webhook | D/09:129-131 | "\"type\": \"injected\"," / "\"source_kind\": \"webhook\","

## `octomind send`
Arguments: `--name`, message or stdin | S/commands/send.rs:31-36 | "pub name: String," / "pub message: Option<String>,"
Empty message rejected | S/commands/send.rs:56-57 | "bail!(\"message must not be empty (pass as argument or pipe via stdin)\");"
No session by that name fails | S/commands/send.rs:83 | "\"no running session named '{}' (socket not found at {:?})\","
Failure message, confirmed | T | `octomind send --name nosuch hi` printed "Error: no running session named 'nosuch' (socket not found at ..."
Success acknowledges queueing only | D/03:74 | "A successful send acknowledges enqueueing, not AI completion; read the daemon output for the response."
Only `run` opens the receiving socket | D/03:59-60 | "All entry points share the same internal inbox abstraction, but only `octomind run` binds the external `octomind send`"
Socket opened with the session | S/session/chat/session/main_loop.rs:82 | "let inject = crate::session::inject_listener::start_inject_listener(&chat_session.session.info.name);"
Same OS user; data dir does not move the socket; duplicate names | D/03:92-94 | "Run the sender as the same OS user with the same runtime environment. `OCTOMIND_DATA_DIR` does not relocate" / "these host-local endpoints. Avoid two live `run` processes with the same session name."

## Scheduled messages
`schedule` is a tool on the `orchestration` server | S/mcp/mod.rs:791,796 | "\"orchestration\" => {" / "\"schedule\" => orchestration::execute_schedule_tool(call)"
Interactive sessions always get `schedule` | S/config/merge.rs:145-146 | "/// Interactive sessions always expose the session-flow tools `schedule` and" / "/// `monitor`."
Other session types follow role grants | D/07s:19-20 | "Non-interactive, ACP," / "and WebSocket sessions use their normal role grants; the template's `assistant` role includes `orchestration:*`."
`/schedule` calls the same tool | S/session/chat/session/commands/schedule.rs:15 | "//! /schedule session command — thin wrapper over the MCP `schedule` tool."
`/schedule` is message-only | D/07m:267-268 | "The user-facing" / "[`/schedule`](../reference/02-session-commands.md#schedule-subcommand-args) slash command remains message-only."
Example form | D/07s:15 | "/schedule add when=\"in 5m\" message=\"Remind me to check the build\" description=\"Build reminder\""
Subcommands | S/session/chat/session/commands/schedule.rs:107 | "\"unknown subcommand '{other}' — use: list, add, remove, edit, help\""
Tool commands `add`, `list`, `remove`, `edit` | D/07m:274 | "| `command` | string, required | `\"add\"`, `\"list\"`, `\"remove\"`, `\"edit\"` |"
Exactly one of `message` or `tool_call` | S/mcp/orchestration/schedule/core.rs:405 | "(Some(_), Some(_)) => Err(\"supply exactly one of 'message' or 'tool_call'\".to_string()),"
`tool_call` shape | S/mcp/orchestration/schedule/core.rs:280 | "Supply {\"name\":\"tool_name\",\"arguments\":{...}}."
`remove` and `edit` take `id` | D/07m:298-299 | "| `remove` | `id` |" ; "| `edit` | `id` |"
`when` formats | S/mcp/orchestration/schedule/core.rs:291-295 | "- 'idle' — fires the next time the session is idle" / "- 'now' — fires on the next scheduler tick (immediately)" / "- Relative: 'in 5m', 'in 2h', 'in 1h30m', 'in 90s', 'in 2h 30m'" / "- Time today: '15:30', '3:30pm', '9am' (if already past, fires tomorrow)" / "- Exact datetime: '2026-03-22 15:30'"
Local time zone | S/mcp/orchestration/schedule/core.rs:290 | "'when' format (local timezone):"
`every` formats | S/mcp/orchestration/schedule/core.rs:298-300 | "- 'idle' — fires every time the session becomes idle (pairs with when=\"idle\" or no 'when', never with a time)" / "- To stop a repeating entry use remove, or clear interval with edit every='none'"
Duration units are `h`, `m`, `s` | S/mcp/orchestration/schedule/storage.rs:343 | "\"units are single-letter h/m/s — e.g. \\\"35m\\\", \\\"2h\\\", \\\"90s\\\", \\\"1h30m\\\" (not \\\"min\\\"/\\\"sec\\\"/\\\"hr\\\")\";"
No day or week unit; use `24h` | D/07s:239-240 | "There is no day or week unit — `1d` is invalid." ; D/07s:204: "Use `24h` for a day; `1d`, `1w`, and `10min` are invalid."
Neither `when` nor `every` means next idle, one-shot | S/mcp/orchestration/schedule/core.rs:302 | "If neither 'when' nor 'every' is provided, the entry defaults to when=\"idle\" (one-shot at next idle). A time interval in 'every' without a time-based 'when' is rejected."
Clock `when` with idle `every` is rejected | S/mcp/orchestration/schedule/core.rs:493,500 | "\"cannot combine a time-based 'when' with idle scheduling — use when=\\\"idle\\\" or omit when\"" ; "\"cannot combine time-based 'every' with idle scheduling — use every=\\\"idle\\\" or omit every\""
`edit` cannot switch idle and timed | D/07m:299 | "Cannot switch an entry between idle and time modes; remove and add to change modes."
Multi-word values need quotes in `/schedule` | D/07s:159 | "Multi-word values must be quoted (`when=\"in 1h 30m\"`, `message='hello world'`)."
Takes effect at once, no restart | D/07m:266-267 | "The agent's MCP `schedule` tool schedules either a message for future injection or a direct MCP tool call in the current" / "session"
Saved with the session, restored on resume, not fired while stopped | D/07m:303-304 | "Schedules are saved in the session log and restored when that session resumes; they do not fire while the process is stopped."
Resume command | D/07s:195 | "octomind run --resume project-monitor"
Non-interactive run without `--daemon` stays alive while schedules pend | D/07s:198-199 | "**Why won't a piped run exit?** Without `--daemon`, a non-interactive run waits while schedules, monitors, tap runs,"
`/schedule` with no arguments lists | S/session/chat/session/commands/schedule.rs:38 | "None => (\"list\", \"\"),"
`list` shows entries with countdown | D/07m:297 | "| `list` | -- | Show pending entries with payloads and countdown |"
`remove` cancels future firings only | D/07m:298 | "Cancel future firings by ID, not work already launched"
Repeating messages need a stop condition | D/07m:313-314 | "Give repeating messages a stop" / "condition."
Idle repeat is an autonomous loop | D/07s:206-207 | "It can fire again after its own response settles; it is an autonomous loop," / "not a timer."

## Doc and code conflicts and open items
Doc status table omits 413; code returns it | D/09:559-566 | "| Body could not be read | `400 Bad Request` | no |" (no 413 row; code: S/session/webhook_listener.rs:212)
The note that `--hook` is unavailable on `server`/`acp` is exact only for `server`; `acp` parses the flag and ignores it | D/03:56-57 | "accepts a `--hook` flag but does not start a webhook listener from it" (matches code S/commands/acp.rs:48-51 and the missing `init_session_runtime` call in `src/acp/`)
Not exercised: a live hook listener, `octomind send` to a live session, and a `schedule` firing. Each needs a running session and a model turn.
