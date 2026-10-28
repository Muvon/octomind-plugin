# Octomind plugin for Claude Code and Codex

Two shared skills that teach Claude Code and Codex to set up and run [Octomind](https://octomind.run):

- `octomind:cli` — install, configure (`config.toml`, roles, taps, MCP servers) and run the open-source Octomind CLI.
- `octomind:cloud` — drive Octomind Cloud from outside: the Developer API (octos, conversations, routines) and the
  OpenAI-compatible hub.

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

Open `/plugins` inside Codex to manage plugins. Invoke the skills with `$octomind:cli` or `$octomind:cloud`.

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

Every claim in a skill is traced to a source (`claim | path:line or URL | quote`); the CLI skill's table is
`evidence/cli.md`. Change a claim, change its row. The Octomind and octohub repos remind contributors in their AGENTS.md
to update these skills when a CLI flag, config key, public route or hub behaviour changes.
