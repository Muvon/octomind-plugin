# Webhooks, sent messages, and scheduled messages

Feed a running Octomind session from outside (an HTTP request or a shell command) or from a timer inside it.

## Webhook listeners (`[[hooks]]`, started with `--hook`)

Where it goes: the listener definition in `<data-dir>/config/hooks.toml` (`<data-dir>` is `$OCTOMIND_DATA_DIR`, or `~/.local/share/octomind`); the script anywhere you can mark executable. Never rewrite `config.toml`.

```toml
[[hooks]]
name = "build-events"
bind = "127.0.0.1:9876"
script = "/home/me/hooks/build-event.sh"
timeout = 30
```

```sh
#!/bin/sh
body=$(cat)
[ -n "$body" ] || exit 0
printf 'Build event from %s:\n%s\nSummarize it.\n' "$HOOK_NAME" "$body"
```

Required fields: `name` (unique), `bind` (a numeric `IP:port`, unique, not a hostname), `script` (path to an executable file; run directly, not through a shell). `timeout` is optional, in seconds, from 1 to 3600.

Script contract: only POST requests run the script. The request body arrives on stdin. On exit 0, trimmed stdout is injected into the session as a user message (HTTP 200 `ok`). Empty stdout injects nothing (204). A non-zero exit is 500 and stderr is logged. A script that outlives `timeout` is killed (504). Non-POST is 405, a body over 10 MiB is 413. The script gets `HOOK_NAME`, `HOOK_METHOD`, `HOOK_PATH`, `HOOK_QUERY`, `HOOK_CONTENT_TYPE`, `HOOK_SESSION`, and one `HOOK_HEADER_<NAME>` per header (uppercase, hyphens become underscores).

How it activates: a defined hook listens only when the session starts with `--hook <name>` (repeat the flag for several hooks). Edit, then start a new session.

```sh
printf '%s\n' 'Wait for build events.' | octomind run --name ci-watcher --daemon --format jsonl --hook build-events
```

- `octomind run` is the only command that starts listeners. `octomind acp` accepts `--hook` and then ignores it. `octomind server` rejects it (`unexpected argument '--hook'`).
- The listener lives as long as the `run` process. Without `--daemon`, a non-interactive run exits once it has no pending work. With piped stdin the first prompt must be non-empty, or startup fails with `No input provided via stdin`.

How to verify: `octomind config --validate` checks names, bind addresses and timeouts, not the script file. At startup an unknown name, a missing script, or a script without the execute bit aborts the session. With the session running, from another terminal: `curl -i -X POST http://127.0.0.1:9876/ --data 'Build 42 passed'` must return `200` and `ok`; in `--format jsonl` output look for an `injected` record with `"source_kind":"webhook"`.

Pitfalls:
- `200` means queued, not answered. Read the session output for the reply.
- Printing nothing is a `204` and nothing reaches the model. Print the full message you want it to read.
- Relative `script` paths resolve from the directory where `octomind` was launched. Use an absolute path for services.
- There is no signature check and no TLS. Bind `127.0.0.1` unless a proxy or your script authenticates callers.
- `[[hooks]]` (plural, main config) is unrelated to the guardrail `[[hook]]` (singular, `.agents/guardrails.toml`).

## Sending a message to a running session (`octomind send`)

Where it goes: nothing to write. It is a command run from another terminal, against a session started with a known name.

```sh
printf '%s\n' 'Wait for instructions.' | octomind run --name ci-watcher --daemon --format jsonl
octomind send --name ci-watcher "Check the build status"
```

Required: `--name` of a live `octomind run` session, and a non-empty message as an argument or on stdin. Give the session `--name` when you start it; otherwise the name is generated.

How it activates: immediately, in the running session. The message is queued as a user message. The command prints a confirmation when it is queued, not when the model finishes.

How to verify: `octomind send --name nosuch hi` fails with `no running session named 'nosuch'`. That confirms the command and the lookup. A real send needs a live session and starts a model turn.

Pitfalls:
- Only `octomind run` opens the receiving socket; `acp` and `server` sessions cannot be reached.
- Run `send` as the same OS user. `OCTOMIND_DATA_DIR` does not move the socket. Two live `run` processes with one name collide.

## Scheduled messages (`schedule` tool and `/schedule`)

Where it goes: nothing to write. The model calls the `schedule` tool (on the `orchestration` server); you type `/schedule`. Interactive `octomind run` sessions always have the tool. Non-interactive, ACP and WebSocket sessions have it only if the role allows `orchestration:schedule` (or `orchestration:*`).

```text
/schedule add when="in 10m" every="10m" message="Check the deploy. If it finished, remove this schedule." description="Deploy watch"
/schedule
/schedule remove <id>
```

Required: the tool takes `command` (`add`, `list`, `remove`, `edit`). `add` needs exactly one of `message` or `tool_call` (`{"name":"tool","arguments":{}}`); `/schedule` is message-only. `remove` and `edit` need `id`.
- `when`: `idle`, `now`, `in 5m`, `in 1h30m`, `in 90s`, `15:30`, `3:30pm`, `9am` (a past time means tomorrow), or `2026-03-22 15:30`. Local time zone.
- `every`: `idle` (every idle moment) or a duration of `h`, `m`, `s` such as `10m` or `1h30m`. `edit ... every=none` stops repeating.
- With neither `when` nor `every`, the entry fires once at the next idle moment.

How it activates: at once, in the running session; no restart. Schedules are saved in the session log and come back with `octomind run --resume <name>`. Nothing fires while the process is stopped. A non-interactive run without `--daemon` stays alive while schedules are pending.

How to verify: type `/schedule`. It lists pending entries with IDs and countdowns, and costs nothing. A firing starts a model turn.

Pitfalls:
- A time `every` needs a time `when`: `every="10m"` alone is rejected. Mixing a clock `when` with `every="idle"` is rejected too.
- `1d`, `1w` and `10min` are invalid. Use `24h`.
- A repeating message with no stop condition repeats until removed.
- `edit` cannot switch an entry between idle and timed. Remove it and add a new one.
- Multi-word values need quotes in `/schedule`.
- `remove` cancels future firings only, not work already started.
