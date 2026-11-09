# Evidence for references/roles.md

`D` = `doc/` and `S` = `src/` in https://github.com/muvon/octomind (branch `master`); line numbers as of 2026-11-08, Octomind 0.70.0.
`T` = tested with the installed binary on 2026-11-08 in a throwaway `OCTOMIND_DATA_DIR`.

claim | source | verbatim quote

## Roles vs tap agents
Role = `[[roles]]` entry, bare name | D/usage/06-roles.md:15 | "> **Role vs. tap agent.** A **role** is a plain `[[roles]]` entry in your config, addressed by its bare name (e.g."
Tap agent addressed by tag; `:` goes to registry | D/usage/06-roles.md:17 | "> `category:variant` **tag** (e.g. `developer:general`). Any tag containing `:` is resolved through the registry,"
Resolver branches on `:` | S/agent/resolver.rs:43 | "if tag.contains(':') {"
Plain name uses config as-is | S/agent/resolver.rs:98 | "Ok((config.clone(), tag.to_string()))"
Shipped config roles | D/usage/06-roles.md:23 | "It helps to know what actually exists out of the box. The default config ships four plain roles, and the default tap"
Shipped role names | D/reference/03-config-reference.md:218 | "| `assistant` | Helpful assistant with working directory;"
Default data dir | D/reference/03-config-reference.md:15 | "The default path is `~/.local/share/octomind/config/config.toml` on macOS/Linux and"

## Custom role
Sibling `roles.toml` beside config.toml | D/usage/06-roles.md:50 | "Add this complete role entry in `roles.toml` alongside the generated `config.toml`."
Use `[[roles]]` with a name | D/usage/06-roles.md:50 | "Use `[[roles]]` with a `name`, not a"
All `*.toml` merged | D/reference/03-config-reference.md:27 | "Octomind supports split-file configuration. All `*.toml` files in the config directory are merged:"
config.toml first, then alphabetical | D/reference/03-config-reference.md:29 | "1. `config.toml` is loaded first"
Others alphabetical | D/reference/03-config-reference.md:30 | "2. Other `*.toml` files are loaded alphabetically"
Same-name entry: whole last entry wins | D/reference/03-config-reference.md:32 | "4. Array entries with the same `name` are deduplicated (the entire last entry wins; it is not a field patch)"
Same-name last entry wins (code) | S/config/loading.rs:127 | "// Last entry wins for same-name items (dedup by \"name\" field)"
Replace role by redefining; repeat fields | D/usage/06-roles.md:352 | "**Why does a role edit remove old settings?** Multi-file config merging keeps the last whole same-named role entry."
Replace verified | T | `reduce` redefined in roles.toml; `octomind config --show` printed "16 char prompt" (length of "Replaced reduce.")
name/system/welcome required | D/usage/06-roles.md:84 | "| `name` | string | yes | Role identifier |"
system required | D/usage/06-roles.md:85 | "| `system` | string | yes | System prompt"
welcome required | D/usage/06-roles.md:86 | "| `welcome` | string | yes | Welcome message on session start |"
Empty welcome = no banner | D/reference/03-config-reference.md:211 | "use `\"\"` for no banner."
Missing welcome fails | T | "Failed to parse TOML configuration: missing field `welcome`"
mcp table optional; if present both fields | D/usage/06-roles.md:88 | "| `mcp` | table | no | If present, include both `server_refs` and `allowed_tools`; if absent, both start empty |"
model table optional, inherits | D/usage/06-roles.md:87 | "| `model` | table | no | Partial model profile; omitted fields inherit from `[model]` |"
Pattern syntax | D/usage/06-roles.md:161 | "- `\"server:*\"` — all tools from a server"
Prefix pattern | D/usage/06-roles.md:163 | "- `\"server:prefix_*\"` — prefix match within a server"
Single tool pattern | D/usage/06-roles.md:164 | "- `\"server:tool_name\"` — one specific tool"
Empty allowed_tools = no role filter | D/usage/06-roles.md:166 | "- Empty array `[]` — no role-level filter; each registered server's `tools` filter still applies"
Unknown server_refs silently dropped | D/usage/06-roles.md:129 | "A `server_refs` entry that names a server not present in the global registry is **silently dropped**"
Dropped (code) | S/config/mcp.rs:457 | "Server '{server_name}' referenced by role but not found in global registry"
`/role` switch behaviour | D/usage/06-roles.md:319 | "A successful `/role` switch rebuilds the system prompt, reinitializes MCP, applies the new role's resolved model"
Saves role to session | D/usage/06-roles.md:320 | "profile, and saves the role to the session. It leaves the global config unchanged."
Switch can replace /model choice | D/usage/06-roles.md:320 | "The profile application can replace"
Config read at startup, so new session (inferred, not stated verbatim) | D/reference/03-config-reference.md:17 | "Startup can create or migrate configuration even for inspection commands."
Run by local name | D/usage/06-roles.md:74 | "octomind run helper"
Validate command | D/usage/06-roles.md:73 | "octomind config --validate"
Inspect tools | D/usage/06-roles.md:155 | "/mcp list"
Unknown plain role rejected | S/session/chat/session/setup.rs:99 | "\"Role '{}' not found. Available roles: {}\","
Rejected check | S/session/chat/session/setup.rs:96 | "if !config.has_role(&role) {"
Config role wins over manifest role | S/config/loading.rs:674 | "// Concatenate roles (additive, skip duplicates by name)"
Skip when name exists | S/config/loading.rs:684 | "if !existing_names.contains(name) {"
Docs state base wins | D/usage/06-roles.md:328 | "role already defined in your base config, the **base (config) role wins** and the manifest's same-named role is"
Local role named like a tag can block resolution | D/usage/06-roles.md:332 | "resolution when the merge skips it. Use a unique plain name for a local customization; a local `developer`"
Tag injected as first role name | S/agent/resolver.rs:72 | "// Always inject the tag as the role name — manifests never need to declare it."
Needs a new role | S/agent/resolver.rs:86 | "\"Agent manifest for '{tag}' must define at least one new [[roles]] entry\""
Tools beyond server_refs | D/usage/06-roles.md:349 | "**Why are tools present with empty `server_refs`?** Check auto-bind, the interactive `schedule`/`monitor` addition,"
Project tools added | D/usage/06-roles.md:184 | "when the session has MCP servers. Inspect `/mcp list` after activation or role changes."
auto_bind adds server | D/usage/06-roles.md:125 | "Empty `server_refs` removes explicit grants, but exact-match `auto_bind` servers still join the role."

## Model profile
Fields accepted | D/usage/06-roles.md:90 | "`[roles.model]` accepts every field from the main `[model]` table: `name`, `reasoning_effort`, `max_tokens`,"
Fields cont. | D/usage/06-roles.md:91 | "`temperature`, `top_p`, `top_k`, `max_retries`, `retry_timeout`, and `request_timeout_seconds`."
temperature range | D/usage/06-roles.md:96 | "- `temperature` — `0.0` to `2.0`"
top_p range | D/usage/06-roles.md:97 | "- `top_p` — `0.0` to `1.0`"
top_k range | D/usage/06-roles.md:98 | "- `top_k` — `0` to `1000` (`0` disables it)"
Range error text | T | "Error: roles.bad.temperature must be between 0.0 and 2.0, got: 3.5"
Precedence | D/usage/06-roles.md:104 | "runtime override  >  role model profile  >  tap name mapping  >  [model]"
CLI --model first | S/cli (octomind run --help) | "Priority: CLI --model > role.model > config.model"
Model names `provider:model` | D/usage/09-workflows.md:289 | "use `provider:model` format (e.g. `anthropic:claude-sonnet-4-6`)"
Foreign-provider model validates without credential | T | role `helper` with `name = "openai:gpt-5.6-luna"`: "Configuration is valid."

## Tap model mapping
`[taps]` maps tag to model | D/reference/03-config-reference.md:185 | "String-to-string map, default `{}`, of tap agent tag to model name. It changes only `name`; all other parameters come"
Example | D/reference/03-config-reference.md:190 | "\"developer:general\" = \"ollama:glm-5.3\""
Only tap agents | D/reference/03-config-reference.md:201 | "`[taps]` only applies to tap agents (tags with `:`). Plain roles resolve `[roles.model]` directly against `[model]`."
Code sets model by exact tag | S/agent/resolver.rs:90 | "if let Some(tap_model) = config.taps.get(tag) {"
Code sets main model name | S/agent/resolver.rs:91 | "merged.model_profile.model = tap_model.clone();"
Role model above taps mapping | D/reference/03-config-reference.md:194 | "**Priority (highest wins):** explicit runtime override > the active role's `[roles.model]` > the tap name mapping >"
Sibling file validates | T | `taps-models.toml` with `[taps]` entry: "Configuration is valid."

## Placeholders
Placeholder list and descriptions | S/commands/vars.rs:38 | "\"{{DATE}}\" => \"Current date and time with timezone\","
HOME | S/commands/vars.rs:43 | "\"{{HOME}}\" => \"User home directory\","
ROLE meaning | S/commands/vars.rs:49 | "\"{{ROLE}}\" => \"Active session role, resolved separately for each session\","
ROLE value from session role | S/session/helper_functions.rs:373 | "placeholders.insert(\"{{ROLE}}\", role_name.to_string());"
Vars flags | S/commands/vars.rs:27 | "/// Show preview of placeholder values (3 lines)"
Vars --expand | S/commands/vars.rs:31 | "/// Show full expanded values for placeholders"
ROLE only in list mode | S/commands/vars.rs:125 | "if !args.expand && !args.preview {"
ROLE row added there | S/commands/vars.rs:127 | "\"{{ROLE}}\","
Vars list output | T | "✓ vars · 11 placeholder(s)" (12 names incl. ROLE)
Placeholders usable in system/welcome | D/usage/06-roles.md:85 | "| `system` | string | yes | System prompt (supports [template variables]"
Welcome supports | D/reference/03-config-reference.md:211 | "Supports template variables; use `\"\"` for no banner."
