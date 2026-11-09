# Local tools

Lets a project expose its own scripts to the model as tools, with no MCP server configuration.

## Local tool script

Where it goes: `<workdir>/.agents/tools/<tool-name>`, one executable file per tool. `<workdir>` is the directory the session runs in. The file name is the tool name: letters, digits, `_` and `-` only, no extension, not starting with `-` or `.`. Run `chmod +x` on it.

Minimal valid example:

```bash
#!/usr/bin/env bash
# @description Echo a message back, optionally uppercased.
# @param *message string The text to echo
# @param uppercase boolean Uppercase the output

set -euo pipefail
: "${OCTOMIND_PARAM_MESSAGE:?message is required}"
if [[ "${OCTOMIND_PARAM_UPPERCASE:-false}" == "true" ]]; then
  printf '%s\n' "$OCTOMIND_PARAM_MESSAGE" | tr '[:lower:]' '[:upper:]'
else
  printf '%s\n' "$OCTOMIND_PARAM_MESSAGE"
fi
```

Required fields and allowed values: the header is the leading comment block of the file. A shebang on line 1 is skipped. Lines may start with `#`, `//`, or `--`. Reading stops at the first line that is not a comment, at the first blank line after the comments began, or after 80 lines.
- `@description` (or `@desc`): required, non-empty. Comment lines after it that start with no `@` continue the text.
- `@param NAME TYPE DESCRIPTION` (or `@arg`): one per parameter. Put `*` directly before the name (`*message`, not `* message`) to mark it required in the schema shown to the model. `TYPE` is `string`, `number`, `integer`, `boolean`, `array`, or `object`; `str`, `int`, `num`, `float`, `bool`, `list`, `obj`, `map` are accepted aliases, and an omitted or unknown type becomes `string`. A `@param` with no name is dropped.
- Other `@tags` are ignored.

How it activates: Octomind scans the directory each time it builds the tool list, so a new, edited, or removed script takes effect without a new session. Tools appear under the server name `local` and are added to every role, with no `allowed_tools` entry. The scan uses the session's current working directory. The model sends parameters; the script gets:
- stdin: one JSON object of all supplied parameters.
- environment: `OCTOMIND_PARAM_<NAME>` per supplied parameter (name upper-cased; arrays and objects as JSON text; null as empty), plus `OCTOMIND_TOOL_NAME` and `OCTOMIND_WORKDIR`.
- working directory: the session workdir. No command-line arguments.
Stdout becomes the result. Non-empty stderr is appended under an `[stderr]` marker even when the script succeeds. A non-zero exit becomes an error result `local tool '<name>' exited with status <code>`, followed by `[stderr]` and `[stdout]` blocks when non-empty. A run is killed after 300 seconds; the limit cannot be changed.

How to verify:
- Run the script the way Octomind does, without a model: `printf '%s' '{"message":"hello","uppercase":true}' | OCTOMIND_PARAM_MESSAGE=hello OCTOMIND_PARAM_UPPERCASE=true .agents/tools/echo` prints `HELLO`.
- Start `octomind run <role>` in the project directory, then type `/mcp list` (the tool should appear under `local`) and `/mcp full` (shows the description and parameters with `*` on required ones).
- Call it: ask the model "Call the echo tool with message hello and uppercase true", then check that a tool call, not just a sentence, appears. Skipped files are explained after `/loglevel debug`, which logs lines starting `local_tool:`.

Pitfalls:
- Tool missing from `/mcp list`: not executable, name has a `.` (for example `echo.sh`), `@description` missing or empty, or the header sits below code.
- A blank line between `@description` and the `@param` lines hides the params.
- Required parameters are not checked before the script starts, and types are not enforced. Validate inputs in the script.
- The role must have at least one configured MCP server. A role with none returns an empty tool list before local tools are added.
- If a local tool has the same name as a tool from one of the role's servers, session start fails with `Duplicate tool names` naming both sources. Rename the script.
- A parameter name with characters other than letters, digits, and `_` (for example `file-path`) makes an `OCTOMIND_PARAM_FILE-PATH` variable that shells cannot read; read it from the stdin JSON instead.
- Scripts run with Octomind's own privileges. Review third-party projects' `.agents/tools/` before using them.
- Relative paths resolve from the session workdir, not from `.agents/tools/`.
