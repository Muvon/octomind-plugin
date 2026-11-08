# Workflows

Lets the user chain several Octomind runs into one repeatable job, using a TOML file that is run by `octomind workflow`.

## Workflow file (ordered steps)

Where it goes: any path, for example `./myflow.toml`. A local file is never added to `config.toml`. To share it by name, put it in a tap (see "Tap workflow").

Minimal valid example:

```toml
name = "summarize"

[[steps]]
name = "summary"
role = "assistant"
prompt = "Summarize this in one sentence:\n{{input}}"
```

Required fields and allowed values:

- Top level: `name`; at least one `[[steps]]`. Optional: `description`, `max_cost` (positive USD number, stop check between steps).
- Every step needs `name` (unique across the whole file, including sub-steps; `input` and `$end` are reserved), `role`, `prompt`.
- Optional per step: `session = "fresh"` (default) or `"continue"`, `timeout` (seconds, 0 = none), `retries`, `model` (`provider:model`, not empty), `workdir`, `skills = [...]`, `capabilities = [...]`.
- A step becomes a block with exactly one flag: `parallel = true`, `loop = true` or `conditional = true`. A block holds its sub-steps as `[[steps.run]]` tables, which take the same fields.
- Parallel block: at least 2 sub-steps. Optional `min_success`, `max_parallel`. A sub-step may set `count` (2 or more) to run the same prompt several times. Sub-step `session = "continue"` is ignored.
- Loop block: `exit_when = { output = "<sub-step>", contains = "..." }` is required; use `matches = "<regex>"` for a regex. `max_iterations` defaults to 10. When the limit is reached the loop ends with a warning, not a failure.
- Conditional block: `condition = { output, contains | matches }`, plus `on_match` and/or `on_no_match` listing sub-step names. Without `output` the condition tests the previous step.
- Variables in a prompt: `{{input}}` is stdin; `{{<step-name>}}` is that step's output (a block name gives all its sub-step outputs joined). Built-ins such as `{{CWD}}`, `{{DATE}}`, `{{ROLE}}`, `{{GIT_STATUS}}` also work. `<context>path:start:end</context>` inlines file lines. A reference to a later step is rejected.

Block headers, for reference: `parallel = true`; `loop = true` with `max_iterations = 3` and `exit_when = { output = "tester", contains = "NO ISSUES" }`; `conditional = true` with `condition = { output = "spec", contains = "security" }` and `on_match = ["deep"]`. Sub-steps follow as `[[steps.run]]` tables.

How it activates: nothing is installed. Run `echo "task text" | octomind workflow myflow.toml`. Input must arrive on stdin; an empty or missing stdin fails with `workflow requires input via stdin`. A new session is not relevant: each step starts its own `octomind run` process. A `session = "continue"` step reuses its own session across loop passes and retries of the same run only; on a repeat pass its prompt is replaced by the previous step's output.

Output: progress and replies go to stderr. stdout stays empty unless `--format jsonl` is given (one `assistant` event per step, then one `cost` event). `--format` accepts only `plain` and `jsonl`.

How to verify:

```bash
octomind workflow myflow.toml --dry-run
```

This validates the file, prints the plan to stdout, reads no stdin and runs no step. It does not check that roles, credentials or `workdir` exist.

Pitfalls:

- A block missing its required part (loop without `exit_when`, parallel with one sub-step, conditional naming an unknown sub-step) fails validation before any step runs.
- Two flags on one step, or `count` outside a parallel sub-step, fail validation.
- A prompt that refers to `{{name}}` of a step declared later fails; in ordered mode, order the steps by dependency.
- A conditional block that is the first step and has no `output` fails at run time (nothing to test).
- Skipped conditional sub-steps expand to empty text in later prompts.

## Graph routing

Where it goes: the same workflow file; add top-level keys and `[[edges]]` tables.

Minimal valid example:

```toml
name = "review-cycle"
entry = "implement"
max_transitions = 12

[[steps]]
name = "implement"
role = "developer:general"
prompt = "Implement:\n{{input}}"

[[steps]]
name = "review"
role = "developer:general"
session = "continue"
prompt = "Review. Reply PASS when complete."

[[steps]]
name = "fix"
role = "developer:general"
session = "continue"
prompt = "Apply this review:\n{{review}}"

[[edges]]
from = "implement"
to = "review"

[[edges]]
from = "review"
to = "$end"
when = { contains = "PASS" }

[[edges]]
from = "review"
to = "fix"

[[edges]]
from = "fix"
to = "review"
```

Required fields and allowed values:

- `entry` names a top-level step; `max_transitions` is 1 or more and caps total step executions across cycles. Setting `max_transitions` without graph mode is an error.
- Each `[[edges]]` has `from`, `to` (a top-level step name or `$end`) and optional `when = { contains | matches, output }`; `when` needs `contains` or `matches`.
- Edges from one step are tried in file order. Each step needs exactly one edge without `when`, declared last for that step. Every step must be reachable from `entry`, and some route must reach `$end`.
- Blocks (parallel, loop, conditional) can be graph nodes; edges name the block.

How it activates: same command as an ordered workflow. Graph mode is on when `entry` or any edge is present.

How to verify: `octomind workflow review-cycle.toml --dry-run`. The plan shows `mode: graph`, the entry and the routes.

Pitfalls:

- In graph mode the file order does not make a variable available. `{{review}}` works only if `review` ran on the chosen route before the step that uses it; otherwise the run fails, not the dry run.
- Reaching `max_transitions` is a budget stop; put a `$end` route on a PASS-style condition.

## Tap workflow

Where it goes: `workflows/<name>.toml` inside a tap repository (see taps.md). It is listed by `octomind workflow` (no argument) and run by bare name: `echo "task" | octomind workflow <name>`.

Minimal valid example: the sequential example above, with `role` changed to a tap agent tag such as `developer:general`.

Required fields and allowed values: same as a local file, with one extra rule. Every step `role` must be a public tap role, a `category:variant` tag that the installed taps provide. A local `[[roles]]` name such as `assistant` is rejected for tap workflows.

How it activates: the argument is treated as a path if it contains `/` or `\` or ends in `.toml`, or if such a file exists; otherwise as a tap workflow name. Taps are searched in order and the first match wins; the built-in default tap is last. Inside a session, `/workflow` lists tap workflows and `/workflow <name> <input>` runs one. Local file paths work only from the command line.

How to verify: `octomind workflow <name> --dry-run`. It fetches the named workflow and checks the public-role rule. If the name is missing the error lists the tap paths that were tried.

Pitfalls:

- A workflow that works from a local file can fail once moved into a tap because of the public-role rule.
