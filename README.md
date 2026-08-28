# claude-marketplace

A personal [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces).
It holds no skills of its own. It is a manifest that points at other people's repos, so
every machine I work on can install the same skill sets with one command and keep them current.

## Why this exists

Claude Code skills live in `~/.claude/`, which is per-machine and not synced. Copying `SKILL.md`
files around by hand means each machine drifts, and none of them ever pick up upstream fixes.

Claude Code already solves this with marketplaces, but only for repos that ship a
`.claude-plugin/marketplace.json`. Not every good skill does. `cursor/plugins`, for example,
publishes a Cursor plugin manifest instead, so Claude Code cannot read it directly.

This repo is the thin layer that fixes both problems. It declares where each skill set really
lives, and Claude Code fetches from those repos directly. Nothing here is a copy, so there is
nothing to keep in sync by hand and no fork to maintain.

## What you get

| Plugin | Skills | Source | Author | License |
|---|---|---|---|---|
| `unslop` | 1 | [`cursor/plugins`](https://github.com/cursor/plugins/tree/main/pstack/skills/unslop) at `pstack/skills/unslop` | Lauren Tan | MIT |
| `mattpocock-skills` | 25 | [`mattpocock/skills`](https://github.com/mattpocock/skills) | Matt Pocock | MIT |
| `skills` | 0 | this repo | | |

`unslop` rewrites text to strip the patterns that make writing read as machine-generated.

`mattpocock-skills` covers TDD, code review, diagnosing bugs, domain modelling, grilling a plan,
resolving merge conflicts, and about twenty more.

`skills` is the bundle. It ships nothing itself and only lists the other two as dependencies, so
installing it installs them.

## Install

Two commands, once per machine.

```bash
claude plugin marketplace add jplozancic/claude-marketplace
```

```bash
claude plugin install skills@jplozancic --scope user
```

Claude Code resolves the bundle's dependencies and installs `unslop` and `mattpocock-skills`
automatically. Restart your session, or run `/reload-plugins`, and the skills are available in
every project.

This repo is private, so the first command needs GitHub credentials on that machine. Claude Code
tries SSH and falls back to HTTPS. If neither is configured, run `gh auth login` first.

To confirm it worked:

```bash
claude plugin list
```

You should see `skills@jplozancic`, `unslop@jplozancic`, and `mattpocock-skills@jplozancic`, all
enabled.

## How it works

Three layers, each doing one job.

The marketplace manifest at `.claude-plugin/marketplace.json` is the only file Claude Code reads
when you add the marketplace. Every entry in it names a plugin and where to fetch it.

Each entry uses a source type matched to how that upstream repo is laid out.
`mattpocock/skills` is a plugin at its repo root, with its own `.claude-plugin/plugin.json`
listing all 25 skills, so a `github` source points at the repo and Claude Code reuses that
manifest as published. `unslop` is one directory deep inside a large monorepo, so a `git-subdir`
source names the path and Claude Code does a sparse clone of just that directory.

The bundle plugin at `plugins/skills/` is a manifest with a `dependencies` array and no
components. Claude Code installs whatever it lists, which turns any number of skill sets into a
single install command.

Because the sources point at the upstream repos rather than at copies, an update pulls from
Cursor and from Matt Pocock, not from here.

## Updating

Auto-update is off by default for marketplaces that Anthropic does not publish. To pull the
latest from every upstream:

```bash
claude plugin marketplace update jplozancic && claude plugin update skills@jplozancic
```

Then update the skill sets themselves:

```bash
claude plugin update unslop@jplozancic && claude plugin update mattpocock-skills@jplozancic
```

To have Claude Code do this on its own, turn on auto-update for the `jplozancic` marketplace in
the `/plugin` interface.

## Adding a skill set

1. Add an entry to the `plugins` array in `.claude-plugin/marketplace.json` with a `source` that
   matches the upstream layout. Use `github` when the repo root is the plugin, `git-subdir` when
   it sits in a subdirectory, and `url` for a git host other than GitHub.
2. Add its name to `dependencies` in `plugins/skills/.claude-plugin/plugin.json`.
3. Bump `version` in that same file.
4. Validate before pushing:

   ```bash
   claude plugin validate . && claude plugin validate ./plugins/skills
   ```

5. On each machine, run `claude plugin update skills@jplozancic` and then `/reload-plugins` to
   install the new dependency.

Dependencies are listed as bare names on purpose. A semver constraint would make Claude Code
resolve against git tags named `{plugin}--v{version}`, and neither upstream tags that way, so a
constrained entry would fail to resolve. Bare names track whatever the source currently publishes.

## Things worth knowing

`unslop` describes itself as "Must always apply", so it triggers on essentially any writing task
and not just when you ask for it. Among other things it bans em dashes outright and pushes for
first person and stated opinions. If that is wrong for a given machine, disable it without
uninstalling:

```bash
claude plugin disable unslop@jplozancic
```

Plugin names are global per marketplace. If you had already added `mattpocock/skills` as its own
marketplace, remove it, otherwise the same 25 skills load twice:

```bash
claude plugin uninstall mattpocock-skills@mattpocock && claude plugin marketplace remove mattpocock
```

Both upstreams are MIT licensed and are fetched from their own repos at install time. This repo
redistributes nothing.
