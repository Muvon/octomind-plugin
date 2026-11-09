# Octomind plugin for Claude Code and Codex

Three shared skills that teach Claude Code and Codex to set up, extend and run [Octomind](https://octomind.run):

- `octomind:cli` — install, sign in, pick a model and run the open-source Octomind CLI, and diagnose a failing setup.
- `octomind:customize` — add or change Octomind commands, roles, MCP servers, guardrails, skills, local tools, workflows, webhooks and taps, and tune its settings.
  Each kind of extension has its own reference file that loads only when needed.
- `octomind:cloud` — drive Octomind Cloud from outside: the Developer API (octos, conversations, routines) and the
  OpenAI-compatible hub.

Each skill is written to fire only on Octomind requests. In a test on 2026-11-08 with Claude Sonnet, 45 prompts run three times each picked the right skill for all 23 Octomind requests and stayed silent on all 22 near-misses, such as adding a Claude Code slash command or hook.

## Install

### Claude Code

```
/plugin marketplace add muvon/octomind-plugin
/plugin install octomind@octomind
```

Shell equivalent: `claude plugin marketplace add muvon/octomind-plugin && claude plugin install octomind@octomind`.

### Codex

```bash
codex plugin marketplace add muvon/octomind-plugin
codex plugin add octomind@octomind
```

Open `/plugins` inside Codex to manage plugins. Invoke the skills with `$octomind:cli`, `$octomind:customize` or `$octomind:cloud`.

For local development, run `codex plugin marketplace add .` from this checkout instead of adding the GitHub repository.
Codex [supports the existing Claude-compatible marketplace](https://learn.chatgpt.com/docs/enterprise/plugin-management#supported-formats)
at `.claude-plugin/marketplace.json`. Its [portable manifest](https://developers.openai.com/plugins/build/plugins#plugin-structure)
at `plugins/octomind/plugin.json` shares the existing skills with the Claude Code plugin.

### Cloud credentials

For the cloud skill, create keys at https://octomind.run/app/keys and export them in your own shell — never paste them
into the chat:

```bash
export OCTOMIND_API_KEY=om_...   # Developer API (api.octomind.run)
export OCTOHUB_API_KEY=...       # hub (hub.octomind.run/v1)
```

## Keeping it true

Every claim in a skill is traced to a source (`claim | path:line or URL | quote`). The CLI skill's table is `evidence/cli.md`; the customize skill's tables are in `evidence/customize/`, one file per reference.
Change a claim, change its row. The Octomind and octohub repos remind contributors in their AGENTS.md
to update these skills when a CLI flag, config key, public route or hub behaviour changes.
