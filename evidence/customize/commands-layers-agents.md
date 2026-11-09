# Evidence for references/commands-layers-agents.md

`D` = `doc/` and `S` = `src/` in /Users/dk/Work/dev/muvon/octomind (Octomind 0.70.0, commit 25f9ae4e); line numbers as of 2026-11-08.
`T` = run against the installed `octomind` 0.70.0 with a throwaway `OCTOMIND_DATA_DIR`; each example in the reference was added as a sibling `.toml` in `config/` and `octomind config --validate` printed "Configuration is valid." Nothing that sends a prompt was run, so session behaviour is from docs and code only.

claim | source | verbatim quote

## Config directory and merging
Config dir is `<data-dir>/config`, default `~/.local/share/octomind`, moved by `OCTOMIND_DATA_DIR` | D/usage/07-mcp-tools.md:610-611 | "`~/.local/share/octomind/config/`; on Windows it is `%LOCALAPPDATA%/octomind/config/`. `OCTOMIND_DATA_DIR` relocates"
Every `*.toml` in the config dir is merged | D/reference/03-config-reference.md:27 | "All `*.toml` files in the config directory are merged:"
Same-name entries replace the whole entry, later file wins | D/reference/03-config-reference.md:32 | "Array entries with the same `name` are deduplicated (the entire last entry wins; it is not a field patch)"
Sibling files are read, and a missing field is a load error | T | "Failed to parse TOML configuration: missing field `input_mode`\n    in `commands`"
Configs load at session start; start a new session | D/usage/10-commands-and-layers.md:15 | "Add custom definitions to a TOML file in your config directory, then start a new session to load them:"

## Commands
`name`, `description`, `command` have no default | S/session/layers/layer_trait.rs:165-167 | "pub name: String," / "pub description: String," / "pub command: String,"
`input_mode`, `output_mode`, `output_role` are required | D/usage/10-commands-and-layers.md:53-54 | "`input_mode`, `output_mode`, and `output_role` are **all mandatory** — they have no serde defaults, so omitting any of"
Missing `description` is a load error (agents, same check applies) | T | "Failed to parse TOML configuration: missing field `description`\n    in `agents`"
`workdir` is optional | S/session/layers/layer_trait.rs:168-169 | "#[serde(default = \"default_workdir\")]"
`command` is split on spaces, no shell | D/usage/10-commands-and-layers.md:90 | "The `command` field is split on whitespace into an executable and arguments; it is not evaluated by a shell."
Same split in code | S/session/layers/processor.rs:59 | "let mut parts = self.config.command.split_whitespace();"
`octomind acp` takes a role name or tap tag | S/commands/acp.rs:20 | "/// Agent tag (e.g. `developer:general`) or role name (e.g. `assistant`)."
A `[[roles]]` entry is not created automatically | D/usage/10-commands-and-layers.md:278-279 | "`[[agents]]` name or `[[layers]]` name does not define a matching `[[roles]]` entry automatically."
`input_mode` values | S/session/layers/layer_trait.rs:55 | "\"Unknown input mode: '{}'. Valid options: last, all, summary\""
Bad input mode is rejected at load | T | "Unknown input mode: 'foo'. Valid options: last, all, summary\n    in `commands.input_mode`"
`last`: last assistant reply plus typed text | D/usage/10-commands-and-layers.md:62 | "Explicit input plus the last assistant response; with empty input, the last assistant response or last genuine user task"
`all`: whole transcript | D/usage/10-commands-and-layers.md:63 | "Chronological non-system transcript followed by the current task"
`summary`: shortened summary | S/session/layers/layer_trait.rs:278 | "crate::session::summarize_context(session, input)"
`output_mode` values | S/session/layers/layer_trait.rs:105 | "\"Unknown output mode: '{}'. Valid options: none, append, replace, last, restart\""
`none` prints only | S/session/chat/command_executor.rs:131 | "\"Output mode: none (command output only)\""
`append` adds a message | D/usage/10-commands-and-layers.md:73 | "Adds output as a new message to the session"
`last` adds only the last output | D/usage/10-commands-and-layers.md:75 | "Append only the last response to session (ignore multiple outputs)"
`replace` keeps first system message, rebuilds welcome | D/usage/10-commands-and-layers.md:74 | "Keeps the first system message, rebuilds welcome/instructions, and inserts all layer outputs"
`replace` drops the old conversation | S/session/chat/command_executor.rs:203 | "chat_session.session.messages.clear();"
`restart` clears all, keeps last output | D/usage/10-commands-and-layers.md:76 | "Clears all messages and inserts only the last layer output"
`output_role` values | S/session/layers/layer_trait.rs:135 | "\"Unknown output role: '{}'. Valid options: assistant, user\""
`/run` with no name lists commands | S/session/chat/session/commands/run.rs:40 | "\"message\": if available_commands.is_empty() { \"No commands configured\" } else { \"Available commands\" }"
Text after the name is the input; otherwise last real user message | D/usage/10-commands-and-layers.md:134-135 | "Additional words after the command name become its explicit input. Without them, `/run` supplies the last genuine user"
Same in code | S/session/chat/session/commands/run.rs:64-66 | "let command_input = if params.len() > 1 {" / "// Use the provided input after the command name"
Spending limits refuse a run | S/session/chat/session/commands/run.rs:89,117 | "\"error\": \"Command execution cancelled due to spending threshold.\"" ; "\"error\": \"Command execution cancelled due to request spending threshold.\""
`/run` looks up the command in the role's command list; unknown name errors | S/session/chat/command_executor.rs:51-53 | ".ok_or_else(|| anyhow::anyhow!(\"Command '{}' not found in configuration\", command_name))?;"
`config --validate` does not check the role in `command` | T | A `[[commands]]` entry with `command = "octomind acp no-such-role"` validated: "Configuration is valid."
No validation of command entries beyond parsing | S/config/validation.rs:30-32 | "if let Some(layers) = &self.layers {" (only `layers` is checked; `commands` is not)
Command shipped as `reduce` | D/reference/03-config-reference.md:393 | "The template ships only the `reduce` command shown above. Run it inside an existing session:"
`replace` removes the conversation | D/usage/10-commands-and-layers.md:292-294 | "**Why did replace remove my conversation?** `output_mode = \"replace\"` keeps the first system message, rebuilds"

## Layers
Rows below describe 0.70.0 (commit 25f9ae4e). The field, its validation and `LayersConfig` were removed from the Octomind source after 0.70.1.
Empty name, description or command is rejected | S/config/validation.rs:196-216 | "return Err(anyhow!(\"Layer at index {} has empty name\", index));"
Empty command rejected, confirmed | T | "Error: Layer 'e' at index 1 has empty command. Layers now execute via ACP protocol"
Docs: layers are invoked programmatically | D/usage/10-commands-and-layers.md:25 | "- **Layers** — orchestration stages invoked programmatically (`[[layers]]`)."
Docs: layers back `/run` | D/usage/10-commands-and-layers.md:36 | "layers reference roles via the `command` field. Layers back the `[[commands]]` slash-command system (`/run <name>`)."
A layers entry alone creates no `/run` command | D/usage/10-commands-and-layers.md:112-113 | "Defining a `[[layers]]` entry alone does not create a `/run` command."
Code: `get_role_config` returns layers but callers drop them | S/config/merge.rs:74 | "let (_role_config, role_mcp_config, _role_layers_config, commands, system_prompt) = self.get_role_config(mode);"
Same elsewhere | S/commands/config.rs:697 | "let (_dev_config, dev_mcp, _dev_layers, _dev_commands, dev_system) = config.get_role_config(\"developer\");"
`/run` reads only `commands` | S/session/chat/command_executor.rs:48 | "let (_, _, _, commands_config, _) = config.get_role_config(role);"
Layer runner is built only for `/run` | S/session/chat/command_executor.rs:83 | "let command_layer = LayerProcessor::new(command_config.clone());"
`get_all_layers` is defined and never called (search of `src/` finds only its definition) | S/config/layers.rs:35 | "pub fn get_all_layers(&self) -> Vec<LayerConfig> {"
Comment: layers handled by workflows | S/config/merge.rs:133 | "// Role-specific layers are now managed by workflows"
Pipelines use `octomind workflow` | D/reference/03-config-reference.md:405 | "> **Multi-step AI workflows** are no longer defined in this config. Use the external CLI: `octomind workflow <file.toml>`"

## Agents
Fields are `name`, `description`, `command`, `workdir` only | S/config/agents.rs:28-43 | "pub struct AgentConfig {" / "pub workdir: String,"
`workdir` optional | S/config/agents.rs:41 | "#[serde(default = \"default_workdir\")]"
Tool is named `agent_<name>` | S/mcp/agent/functions.rs:90 | "name: format!(\"agent_{}\", agent_config.name),"
Arguments `task` (required), `async` (default false) | S/mcp/agent/functions.rs:116-119 | "\"default\": false" / "\"required\": [\"task\"]"
Agent has no memory of the conversation | S/mcp/agent/functions.rs:111 | "The agent starts with ZERO context — it sees ONLY this text, none of your conversation or findings."
Async result returns as a user message | D/usage/10-commands-and-layers.md:217-218 | "`async: true` returns immediately. The result is injected into the conversation as a user message when complete,"
Async marker text | D/usage/10-commands-and-layers.md:218 | "prefixed `[Async agent '<name>' completed]` (or `[Async agent '<name>' failed]` on error)."
Async job cap = CPU count | D/usage/10-commands-and-layers.md:226 | "Max concurrent async jobs is fixed at the machine's CPU core count (fallback `4` if it can't be detected); it is not"
Parent role needs the `agent` server | D/usage/10-commands-and-layers.md:168 | "Enable the `agent` server for the parent role. For example, a minimal parent role is:"
`agent` server serves config agents | S/mcp/mod.rs:493-494 | "\"agent\" => {" / "let fns = agent::get_all_functions(config);"
Example parent role grant | D/usage/10-commands-and-layers.md:177-178 | "server_refs = [\"agent\"]" / "allowed_tools = [\"agent:*\"]"
Start the delegating role by name | D/usage/10-commands-and-layers.md:182 | "octomind run delegator"
Dynamic agent tool lives on the `runtime` server | S/mcp/runtime/mod.rs:48 | "crate::mcp::runtime::dynamic_agents::get_agent_tool_function(),"
`runtime` server purpose | D/usage/07-mcp-tools.md:31 | "Harness reconfiguration: register MCP servers, manage dynamic agents, load skills, capability"
Template `assistant` role includes `runtime` | config-templates/default.toml:188 | "server_refs = [\"core\", \"orchestration\", \"runtime\", \"filesystem\", \"agent\"]"
Dynamic `add`: `name`, `system` required | S/mcp/runtime/dynamic_agents.rs:99-105 | "anyhow::bail!(\"Agent name is required\");" / "anyhow::bail!(\"Agent system prompt is required\");"
Optional add fields | D/usage/07-mcp-tools.md:427-434 | "`description` | string, optional | Tool description; defaults to empty" ... "`workdir` | string, optional | Working directory; defaults to `\".\"`"
`add` does not enable; `enable` makes `agent_<name>`; other actions | S/mcp/runtime/dynamic_agents.rs:391 | "add: register a new agent config (does NOT enable it yet)\n- enable: enable a registered agent (makes it available for execution)\n- disable: disable an agent (config stays registered)\n- remove: unregister an agent entirely"
Dynamic agents are in memory and in-process | S/mcp/runtime/dynamic_agents.rs:18-22 | "//! and are stored in memory only." / "//! executed in-process using the session infrastructure."
`enable` takes effect in the running session | S/mcp/runtime/dynamic_agents.rs:135-136 | "// Register the tool in the global tool map" / "crate::mcp::tool_map::register_dynamic_agent_tool(name);"
Same name in config and dynamic is refused | S/mcp/agent/functions.rs:195 | "format!(\"Agent '{agent_name}' exists in both config and dynamic agents - ambiguous\"),"
Config agent command is space-split | S/mcp/agent/functions.rs:255 | "let mut parts = command.split_whitespace();"
Verify with `/mcp full` | D/usage/07-mcp-tools.md:19 | "`/mcp full` shows the schemas advertised by your enabled servers."
Shipped agent points at a role that does not ship | D/usage/10-commands-and-layers.md:164-166 | "`octomind acp context_gatherer`, but the shipped local `[[roles]]` list has no role with that name. Define that role or"
Shipped agent command | config-templates/default.toml:500 | "command = \"octomind acp context_gatherer\""
Shipped roles are assistant, task_refiner, task_researcher, reduce | config-templates/default.toml:180,193,230,280 | "name = \"assistant\"" / "name = \"task_refiner\"" / "name = \"task_researcher\"" / "name = \"reduce\""

## Prompt templates
Struct: `name`, `prompt` required, `description` optional | S/config/mod.rs:339-346 | "pub name: String," / "pub prompt: String," / "pub description: Option<String>,"
Missing `prompt` is a load error | T | "Failed to parse TOML configuration: missing field `prompt`\n    in `prompts`"
`/prompt` lists templates | S/session/chat/session/commands/prompt.rs:29-45 | "// Show available prompts" / "\"action\": \"list\","
`/prompt <name>` queues as a user message | D/usage/10-commands-and-layers.md:236-238 | "Reusable prompts sent into the session via `/prompt <name>`. The prompt text is queued into the session inbox and picked" / "up by the main loop as a normal **user message** on the next turn"
Queued through the inbox | S/session/chat/session/commands/prompt.rs:72 | "crate::session::inbox::push_inbox_message(crate::session::inbox::InboxMessage {"
Text sent verbatim, no placeholders | S/session/chat/session/commands/prompt.rs:91 | "// For now, return template as-is"
Same, docs | D/usage/10-commands-and-layers.md:238-239 | "The template is sent verbatim: prompt-template variable substitution (`{role}`, `{model}`, etc.) is not currently implemented."
Shipped names | D/reference/03-config-reference.md:472 | "The template ships `review`, `explain`, `optimize`, `test`, and `debug`, each with its own description and prompt text"

## Doc and code conflicts
`[[layers]]` "invoked over ACP" / "programmatically" vs no caller | D/reference/03-config-reference.md:380-381 | "`[[layers]]` are orchestration units invoked over ACP." (code: see Layers rows)
Agent `command` called a "Shell command" vs space-split, no shell | D/reference/03-config-reference.md:437 | "Shell command starting an ACP server over stdio" (code: S/mcp/agent/functions.rs:255; D/usage/10-commands-and-layers.md:199 says "Whitespace-split executable and arguments")
