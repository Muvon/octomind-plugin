# Taps

Lets the user install, create, and publish agents (and skills, capabilities, workflows) that Octomind runs by a `category:variant` tag.

A tap is a Git repository or local directory that follows a fixed layout. A tag such as `demo:helper` means the file `agents/demo/helper.toml` inside a tap.

## Tap (add, list, remove, publish)

Where it goes: nothing is edited by hand. `octomind tap` writes the registration to `<data-dir>/taps.toml`. GitHub taps are cloned to `<data-dir>/taps/<user>/octomind-<repo>/`. Local taps are symlinked at that same path. `<data-dir>` is `OCTOMIND_DATA_DIR` when set.

Minimal valid example:

```bash
octomind tap myorg/my-agents                      # GitHub clone
octomind tap myorg/my-agents ./octomind-my-agents # local directory (must exist)
```

Required fields and allowed values:
- The name must be exactly `user/repo`.
- For GitHub, `user/repo` is cloned from `github.com/user/octomind-repo`. The repository must be named `octomind-<repo>`.
- The local path must exist. Use `./` or an absolute path.
- `muvon/tap` is built in, always last in search order, and cannot be added or removed.

How it activates: immediately, no new session needed for the next `octomind run`. Search order for a tag is user taps in the order added, then `muvon/tap`. The first tap that has the file wins. GitHub taps refresh in the background at most every 15 minutes. Local taps are live.

How to verify: `octomind tap` lists user taps. `octomind complete run` prints every tag found in the tap files.

Remove with `octomind untap myorg/my-agents`. This removes the registration and, for a local tap, the symlink only. A GitHub clone stays on disk. The manifest cache (below) is not cleared.

Publish: create a public GitHub repository named `octomind-<repo>` under your account, with `agents/<category>/<variant>.toml` at the repository root. Others install it with `octomind tap <user>/<repo>`.

Pitfalls:
- Repository `myorg/my-agents` on GitHub will not install. It must be `myorg/octomind-my-agents`.
- Tap folders other than `agents/` do not replace the others: skills, capabilities, workflows, and plugins have their own folders (`skills/`, `capabilities/`, `workflows/`, `plugins/`).

## Agent manifest

Where it goes: `<tap>/agents/<category>/<variant>.toml`. Exactly two path levels.

Minimal valid example:

```toml
# Title: Demo Helper
# Description: Answer questions about one topic in short sentences.
capabilities = ["octomind/filesystem-read"]

[[roles]]
name = "demo:helper"
system = "You answer questions. Working directory: {{CWD}}."
welcome = "Ready."
```

Required fields and allowed values:
- `# Title:` and `# Description:` comments before the first TOML table. Both are required when agents are listed or discovered.
- At least one `[[roles]]` entry. The first role's `name` is overwritten with the tag, so keep it equal to the path.
- `capabilities = [...]` is optional and must sit at the top level, before any table header. Each entry is `name` (the agent's own tap first, then others in order), `octomind/name` (only the built-in tap), or `org/name` (the first connected tap owned by `org`). The capability's tools, servers, and `[deps]` are merged into the role.
- `[deps] require = ["org/tool"]` runs `<tap>/deps/org/tool.sh` with bash on every start, before tools load. Exit 0 means ok. The script must be safe to run repeatedly.
- `{{INPUT:KEY}}` asks the user once and saves the answer in `<data-dir>/inputs.toml`. `{{ENV:KEY}}` reads an environment variable. If unset, the user is asked and the value is appended to `./.env`. An empty but set variable is accepted. `{{CWD}}` is filled in when the prompt is built. Write `{{{{INPUT:KEY}}}}` for literal text.
- A capability whose wiring uses an unset or empty `{{ENV:KEY}}` is skipped and the agent starts without it.

How it activates: `octomind run demo:helper` fetches the manifest, expands capabilities, fills placeholders, runs deps, then merges it into config. A new session is needed after an edit, and the cache can hide the edit (see Pitfalls).

How to verify: `octomind complete run` must list the tag. This only checks the path, not the file content. Content errors show when the agent starts.

Pitfalls:
- Manifest cache: fetched manifests are copied to `<data-dir>/agents/<category>/<variant>.toml`. A cached copy is used while fresh (`[registry] cache_ttl_hours`). A stale copy is returned first and refreshed in the background. After editing a local tap, delete only that cache file before the next run.
- Removing a tap does not clear the cache, so an untapped tag may still run.
- `{{INPUT:KEY}}` in an unattended run (server, ACP, tap tool) fails if no value was stored. Run the agent once interactively first.
- `category:variant@version` parses but the version is ignored.

## New tap (`tap init`)

Where it goes: `./octomind-<repo>` in the current directory, or the path given with `--dir`. It must not exist or must be empty.

Minimal valid example:

```bash
octomind tap init myorg/my-agents --agent my-agents:assistant
```

Required fields and allowed values:
- `user/repo` as the tap name.
- `--agent domain:spec` is optional. The domain defaults to the repo name. The spec comes from the downloaded scaffold.
- `--dir DIR` is optional.

How it activates: the command renders the scaffold from the `muvon/tap` clone, runs the scaffold's validate command, runs `git init`, and registers the directory as a local tap. It clones `muvon/tap` on first use, so it needs network then. Then run `octomind run my-agents:assistant`.

How to verify: the output prints `name`, `dir`, `agent`, and `next`. `octomind tap` then lists the new tap as local.

Pitfalls:
- It fails if the tap name is already registered. Run `octomind untap` first.
- A scaffold validate failure removes the directory it created.
- The scaffold validator may need Python 3.11 or later with TOML support, and bash.

## Model for one tap agent

Where it goes: a sibling file such as `<data-dir>/config/taps-models.toml`, never `config.toml`. Don't confuse it with `<data-dir>/taps.toml`, which records registered taps.

```toml
[taps]
"demo:helper" = "octohub:auto"
```

Required fields and allowed values: key is the exact tag as typed after `octomind run`; value is `provider:model`. It replaces only the model name.

How it activates: on the next `octomind run demo:helper`. The runtime `-m` flag and an explicit model in the manifest role win over this mapping.

How to verify: `octomind config --validate`.

Pitfalls: a tag typed with a different spelling (for example with `@version`) does not match the key.
