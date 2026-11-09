# Octomind skills

Lets a project or user give Octomind sessions reusable instructions that load by command, by environment variable, or automatically when a rule matches. These are Octomind's own skills, separate from Claude Code or Codex skills.

## SKILL.md

Where it goes, one directory per skill, directory name equal to the `name` field:
- `<workdir>/.agents/skills/<name>/SKILL.md` for one project.
- `~/.config/agents/skills/<name>/SKILL.md` for the user.
- `<tap>/skills/<name>/SKILL.md` inside a registered tap (needed if the skill has a `validate` script).
- Skills under an Octomind plugin (`plugins/<plugin>/skills/<name>/`) can be listed and loaded by hand but are not in the automatic pool.

When two skills share a `name`, the first found wins: taps, then project, then user, then plugins, then machine-generated skills.

Minimal valid example:

```markdown
---
name: project-review
description: Review project changes with explicit evidence and validation limits.
---
Report each finding with its file path, the observed behavior, and the smallest correction.
```

Required fields and allowed values: the file must start with `---` and close the frontmatter with a `---` line. `name` and `description` are required; if either is missing the skill is skipped without any message. Read keys are `name`, `description`, `compatibility`, `license`, `allowed-tools` (space-separated tool names; a hint, not a grant), `capabilities` (space-separated or `["a", "b"]`; loaded when the skill activates), `domains`, `rules`. Other keys, including `title`, are ignored. Keep every value on one line and add no trailing comments. The body after the frontmatter is the injected instruction text. Files in the skill's `scripts/`, `references/`, `assets/` (one level only) are listed by absolute path when it activates.

How it activates: without `domains` and `rules` a skill is manual only (see the sections below). Start a new session after adding or renaming a skill; the automatic pool is built at session start. Manual loading reads the disk each time.

How to verify: there is no lint command. Run `octomind run developer:general`, then `/skill` (lists skills, active first) or `/skill *project*` (filter). Toggle with `/skill project-review`. Type `/loglevel debug` before a request to log why automatic activation skipped it.

Pitfalls:
- `name` different from the directory name: listed, but `/skill <name>`, `OCTOMIND_SKILLS`, and auto-activation cannot find it.
- A missing `description` or `name` hides the skill silently.

## rules and domains (automatic activation)

Where it goes: in the SKILL.md frontmatter of a skill in a tap, `.agents/skills`, or `~/.config/agents/skills`.

Minimal valid example:

```yaml
domains: developer
rules:
  - content(review)
  - file(Cargo.toml) content(rust)
```

Required fields and allowed values:
- Both a matching `domains` and a non-empty `rules` list are needed. A skill with no `domains`, or none matching the session's domain, never enters the automatic pool.
- A role's domain is the part of its tag before the first `:`. `developer:general` has domain `developer`; a role named `assistant` has domain `assistant`. `domains: "*"` matches every domain. Several domains are space-separated or `[a, b]`.
- `rules:` must be a line by itself, followed by indented `- ` lines. Each line is an alternative (OR). Checks on one line are all required (AND).
- Checks: `file(glob)`, `content(word)` (whole word, case-insensitive, on the user message), `grep(pattern)` or `grep(pattern, glob)`, `env(VAR)` or `env(VAR=value)`, `match(regex)` (on the user message), `bin(name)`, `session(text)`, `workdir(text)`, `semantic(phrase)` or `semantic(phrase, threshold)`.

How it activates: checked on each new user message in the session. Messages under 8 non-whitespace characters never activate anything, and `<tag>…</tag>` blocks in the message are ignored. Skills already active are skipped. `[skills].auto_activation` must be true. Semantic-only matches fire only if the top skill beats the runner-up by a margin, and need the local embedding model; the default threshold in code is 0.65.

How to verify: start a new session, send a request containing the trigger word and at least 8 characters, then run `/skill`; the skill should be listed as active.

Pitfalls:
- A role defined in your config with a name like `conventions` has domain `conventions`, so `domains: developer` does not match it.
- Putting `file(...)` next to `semantic(...)` does not remove the semantic margin test.

## validate script

Where it goes: `<tap>/skills/<name>/validate`, an executable file next to SKILL.md. A validate script under `.agents/skills` or `~/.config/agents/skills` is never run.

Minimal valid example:

```bash
#!/usr/bin/env bash
grep -qi 'validation' && exit 0
echo 'Include a validation statement.' >&2
exit 1
```

Required fields and allowed values: it receives the literal first argument `assistant`, the final assistant message on stdin, and runs in the project directory. Exit 0 means valid. It sends feedback to the model only if it exits non-zero and writes something (stderr, or stdout when stderr is empty). A silent non-zero exit does nothing.

How it activates: runs at the end of a turn for active skills, only when `[skills].auto_validation = true`. After `max_retries` failures in a row for one skill it stops running until a new session starts; `max_retries = 0` removes the cap. It is killed after `validation_timeout` seconds (0 means one hour). Timeouts and spawn errors are only debug-logged.

How to verify: `printf 'text' | <tap>/skills/<name>/validate assistant; echo $?`. Then register the tap, set `auto_validation = true`, and start `OCTOMIND_SKILLS=<name> octomind run developer:general`.

Pitfalls:
- The script must be executable (`chmod +x`).
- Project guardrail validators in `.agents/guardrails.toml` are a different mechanism and ignore `auto_validation`.

## activate script

Octomind 0.70.0 does not run an `activate` file. `/skill` only shows the word `activate` beside a skill that has one. Use `rules` for automatic activation.

## OCTOMIND_SKILLS and /skill

Where it goes: an environment variable set before starting the session, and slash commands typed in the session.

Minimal valid example:

```bash
OCTOMIND_SKILLS=project-review,customer-domain octomind run developer:general
```

Required fields and allowed values: comma-separated exact skill names, looked up in taps, project and user skill directories, and plugins. No globs or aliases. Unknown names print a warning and are skipped. Rules are not evaluated for these skills, and `domains` is not needed.

How it activates: at session start, so it needs a new session. In a running session: `/skill` lists, `/skill 2` shows page 2 (15 per page), `/skill *review*` filters, `/skill <name>` loads the skill if inactive and unloads it if active. Manual loading ignores `domains` and `rules`.

How to verify: after starting, run `/skill` and look for the name marked active.

Pitfalls: typing `/skill <name>` twice unloads the skill.

## [skills] settings

Where it goes: a `[skills]` table in a sibling file such as `<data-dir>/config/settings.toml`, holding only the keys you change. For one run only, put `-c skills.auto_validation=true` before `run`, as in `octomind -c skills.auto_validation=true run developer:general`.

Minimal valid example:

```toml
[skills]
auto_activation = true
auto_validation = false
activation_timeout = 3
validation_timeout = 60
max_retries = 3
```

Required fields and allowed values: `auto_activation` and `auto_validation` are booleans; `validation_timeout` is seconds; `max_retries` is a count; `activation_timeout` is reserved and has no effect.

How it activates: read when a session starts; start a new session after editing.

How to verify: `octomind config --validate`.

Pitfalls: `auto_validation` is off in the generated config, so `validate` scripts do nothing until it is set to true.
