# Evidence for skills/customize/SKILL.md

`D` = `doc/` and `S` = `src/` in https://github.com/muvon/octomind (branch `master`); line numbers as of 2026-11-08, Octomind 0.70.0. Each extension's table row is backed by the matching file in this directory.

claim | source | verbatim quote

Config directory is the config file's parent | S/config/loading.rs:355-364 | "// Get the config directory (config file's parent)."
OCTOMIND_CONFIG_PATH selects the config path | S/config/loading.rs:347-348 | "// Check for custom config path from environment variable"
Every `*.toml` in the directory is merged | D/usage/03-configuration.md:194-199 | "`config.toml` loads first."
Missing keys are not filled from the template | S/config/loading.rs:421,439-440 | "All required fields must be present." ; "REMOVED: No more default_with_env - config must be complete and explicit"
Older config.toml is migrated | S/config/loading.rs:407-411 | "// Check for automatic config upgrades on config.toml"
Same-named array entry replaces the whole entry | D/usage/03-configuration.md:202 | "The last same-named array entry replaces the whole entry, so repeat all required server or role fields."
Root keys go before the first table | D/usage/03-configuration.md:57-58 | "Edit these existing keys before any TOML table header; do not append them beneath `[model]`."
Do not edit version | D/reference/03-config-reference.md:69 | "Config version. Do not modify. Used for automatic upgrades."
Setters rewrite config.toml from the merged config | S/config/loading.rs:443-463 ; S/commands/config.rs:370 | "let config_str = toml::to_string_pretty(self)"
Only config --validate reports unknown keys | S/commands/config.rs:95 | "let ignored_reports = config.check_ignored_keys();"
Defining `[[layers]]` alone creates no `/run` command | D/usage/10-commands-and-layers.md:112-113 | "Defining a `[[layers]]` entry alone does not create a `/run` command."
A new session loads config changes | D/usage/10-commands-and-layers.md:15 | "Add custom definitions to a TOML file in your config directory, then start a new session to load them:"
