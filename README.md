# Octomind plugin for Claude Code

Two skills that teach Claude to set up and run [Octomind](https://octomind.run):

- `octomind:cli` — install, configure (`config.toml`, roles, taps, MCP servers) and run the open-source Octomind CLI.
- `octomind:cloud` — drive Octomind Cloud from outside: the Developer API (octos, conversations, routines) and the
  OpenAI-compatible hub.

## Install

```
/plugin marketplace add muvon/octomind-plugin
/plugin install octomind@octomind
```

Shell equivalent: `claude plugin marketplace add muvon/octomind-plugin && claude plugin install octomind@octomind`.

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
