# MCP servers

Lets the user give Octomind extra tools by registering an MCP server (a separate program or URL that offers tools over the Model Context Protocol).

`<data-dir>` below is `~/.local/share/octomind` on macOS and Linux, or `$OCTOMIND_DATA_DIR` when that variable is set. The config directory is `<data-dir>/config/`.

## External server in a config file

Where it goes: a new file `<data-dir>/config/mcp-<name>.toml`. Do not edit `config.toml`. Files named `mcp-*.toml` load after all other config files, so an entry there replaces a same-named server declared elsewhere. The replacement is whole-entry, so repeat every required field.

Minimal valid example (stdio server, a local process):

```toml
[[mcp.servers]]
name = "notes"
type = "stdio"
command = "notes-mcp"
args = ["serve"]
env = { NOTES_TOKEN = "{{ENV:NOTES_TOKEN}}" }
timeout_seconds = 30
tools = []
auto_bind = ["assistant"]
```

HTTP server (a URL that already speaks MCP):

```toml
[[mcp.servers]]
name = "search"
type = "http"
url = "http://localhost:9000/mcp"
headers = { Authorization = "Bearer {{ENV:SEARCH_TOKEN}}" }
timeout_seconds = 30
tools = ["search_*"]
auto_bind = ["developer:general"]
```

Required fields and allowed values:
- `name` (unique), `type` (`stdio`, `http`, or `builtin`), `timeout_seconds` (1 to 3600), `tools` (list; empty means all tools the server offers; trailing `*` wildcards allowed).
- stdio: `command` and `args` (use `[]` for none); optional `env` (table) and `cwd`.
- http: `url`; optional `headers` (table).
- `auto_bind` is optional: a list of exact role tags. `developer` does not match `developer:general`.
- `{{ENV:KEY}}` is replaced from the environment Octomind was started in. It works in `command`, `args`, `env` and `headers`. If a referenced variable is unset or empty, a stdio server is not spawned and an http server cannot connect.
- An http server with no `Authorization` header gets OAuth discovery when the server asks for sign-in. A configured `Authorization` header turns that off.

How it activates: files are read at startup, so start a new session. Registering a server does not enable it. It becomes visible to a role only through `auto_bind` or that role's `server_refs`.

How to verify: `OCTOMIND_DATA_DIR=<dir> octomind config --validate` checks the TOML (a stdio entry without `args` fails with ``missing field `args` ``). In a session, `/mcp list` shows servers, `/mcp full` shows each tool's schema, `/mcp health` reconnects or restarts.

Pitfalls:
- Validation does not start the server or check env variables. A wrong `command` or missing variable only shows up in the session.
- A role with a non-empty `allowed_tools` hides servers it does not name. `auto_bind` adds `<name>:*` to that list for you; `server_refs` does not.

## Binding a server to a role by reference

Where it goes: in the role entry, in a sibling file such as `<data-dir>/config/roles.toml`.

```toml
[[roles]]
name = "notes-helper"
system = "You help with notes."
welcome = "Hello"

[roles.mcp]
server_refs = ["notes"]
allowed_tools = ["notes:*"]
```

Required fields: `server_refs` lists server names defined under `[[mcp.servers]]` (an empty list leaves the role without tools). `allowed_tools` lists `<server>:<tool>` patterns, or `<server>:*` for a whole server; an empty list means no restriction.

How it activates: new session, start with `octomind run notes-helper`. How to verify: `octomind config --validate`, then `/mcp list` in the session.

Pitfalls: a name in `server_refs` that matches no server is silently skipped. A server listed in `server_refs` but not matched by any `allowed_tools` pattern is dropped when `allowed_tools` is non-empty. Role fields are covered in `roles.md`.

## Runtime `mcp` tool and persist

Where it goes: nothing is written until `persist`. The model calls the `mcp` tool during a session. It exists only when the role references the `runtime` server and allows `runtime:mcp` or `runtime:*`.

Minimal valid example (one tool call per object, in this order):

```json
{"action": "add", "name": "local_api", "server_type": "http", "url": "http://localhost:3000/mcp"}
{"action": "enable", "name": "local_api"}
{"action": "persist", "name": "local_api"}
```

Allowed values: `action` is `list`, `add`, `enable`, `disable`, `remove`, `persist` or `unpersist`. `add` takes `name`, `server_type` (`stdio` or `http`), `command` and `args` (stdio) or `url` (http), and optional `timeout_seconds` and `tools`. `add` registers without connecting; `enable` connects and applies its own optional `tools` filter. The tool cannot set `headers`, `env` or `cwd`; use a TOML file for those.

How it activates: `add` and `enable` take effect in the running session. `persist` writes `<data-dir>/config/mcp-<name>.toml`. If the server was enabled, the file gets `auto_bind = ["<current role>"]`; if disabled, no `auto_bind`, so it does not load by itself. The file loads in new sessions.

How to verify: `{"action": "list"}` shows configured and dynamic servers. Persisted ones carry a marker. Check that `<data-dir>/config/mcp-<name>.toml` exists.

Pitfalls:
- `disable` and `remove` do not change saved files. `unpersist` deletes the file but does not disconnect a live server.
- A `tools` value given to `add` is stored but not applied by `enable` alone. Pass the same filter to `enable`. `persist` saves the stored list, not the `enable` filter.
- `persist` writes under the normal data directory even when `OCTOMIND_CONFIG_PATH` points elsewhere.
- The human `/mcp` command only inspects (`info`, `list`, `full`, `health`, `dump`, `validate`). There is no `/mcp add`.

## Built-in servers

Octomind provides `core`, `orchestration`, `runtime` and `agent` with `type = "builtin"`. The default config already declares them; no file is needed. A role reaches them through `server_refs` and `allowed_tools`, for example `server_refs = ["runtime"]`, `allowed_tools = ["runtime:*"]`. `orchestration` offers `tap`, `schedule` and `monitor`; `runtime` offers `mcp`, `agent`, `skill` and `capability`; `agent` offers one `agent_<name>` tool per `[[agents]]` entry.

Filesystem tools are not built in. They come from an external server such as `octofs`, usually through a tap capability.

Verify with `/mcp list` in a session.

## Capabilities and servers

A capability is a named bundle of servers and tool filters that a tap ships in `capabilities/<name>/<provider>.toml`. The user does not write it. The user's choice of provider goes in a sibling file such as `<data-dir>/config/capabilities.toml`:

```toml
[capabilities]
codesearch = "octocode"
```

The key is a capability name, the value a provider file name without `.toml`. Applies on the next session. Verify with `octomind config --validate`. During a session, the `capability` tool lists, enables and disables capabilities. `OCTOMIND_CAPABILITIES=<exact-name>[,<name>] octomind run` forces them at start. Pitfall: names must match exactly; there is no fuzzy matching.
