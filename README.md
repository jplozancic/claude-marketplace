# claude-marketplace

My [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces).
It ships no skills of its own. It is a list of other people's repos and instructions for fetching
them, so every machine I work on installs the same set with one command and then keeps itself
current.

## Why this exists

Skills live in `~/.claude/`. That directory is per-machine and nothing syncs it, so I was copying
`SKILL.md` files between machines by hand. Every machine drifted, and none of them ever picked up
an upstream fix.

Claude Code already solves this with marketplaces. The catch is that it only reads a repo that
ships `.claude-plugin/marketplace.json`, and plenty of good skills don't. `cursor/plugins`
publishes a Cursor manifest instead, so Claude Code cannot read it at all.

This repo is the thin layer over that gap. It says where each skill set actually lives and lets
Claude Code fetch from the source. Nothing here is a copy, so there is no fork to rebase and no
vendored directory to refresh.

## What's in it

| Plugin | Skills | Fetched from | Author | License |
|---|---|---|---|---|
| `skills` | 0 | this repo | me | |
| `unslop` | 1 | [`cursor/plugins`](https://github.com/cursor/plugins/tree/main/pstack/skills/unslop), path `pstack/skills/unslop` | Lauren Tan | MIT |
| `mattpocock-skills` | 25 | [`mattpocock/skills`](https://github.com/mattpocock/skills) | Matt Pocock | MIT |

`unslop` rewrites text to strip the patterns that make writing read as machine-generated. It is
one skill and it earns its place.

`mattpocock-skills` covers TDD, code review, diagnosing bugs, domain modelling, grilling a plan,
resolving merge conflicts, and about twenty more.

`skills` is the bundle. It has no skills, no commands and no hooks. Its manifest is a name, a
version and a `dependencies` array, which is enough to make it the only thing you ever install.

## Install

Two commands, once per machine.

```bash
claude plugin marketplace add jplozancic/claude-marketplace
```

```bash
claude plugin install skills@jplozancic --scope user
```

Claude Code resolves the bundle's dependencies and installs `unslop` and `mattpocock-skills` for
you. The install prints `+ 2 dependencies` when it works. Restart the session or run
`/reload-plugins`, and the skills are live in every project.

This repo is private, so that first command needs GitHub credentials on the machine. Claude Code
tries SSH and falls back to HTTPS. Run `gh auth login` first if neither is set up.

Check the result with `claude plugin list`. You want three entries, all enabled.

## How it works

Three layers, each doing one job.

The manifest at `.claude-plugin/marketplace.json` is the only file Claude Code reads when you add
the marketplace. Every entry names a plugin and says where to fetch it.

Each entry picks a source type to match how that upstream repo is laid out. `mattpocock/skills`
is a plugin at its own repo root and already has `.claude-plugin/plugin.json` listing all 25
skills, so a `github` source points at the repo and Claude Code uses that manifest as published.
`unslop` sits one directory deep inside a monorepo of 45 skills, so a `git-subdir` source names
the path and Claude Code sparse-clones just that directory.

The bundle at `plugins/skills/` is a manifest with a `dependencies` array and no components.
Installing it installs everything it lists. Adding a fourth skill set later changes what one
command installs, not how many commands you run.

Because every source points at an upstream repo instead of a copy, an update pulls from Cursor
and from Matt Pocock. It does not pull from me.

## Why a bundle and not one big plugin

The obvious idea is a single plugin that literally contains every skill. I tried to talk myself
into it and the numbers said no.

Claude Code namespaces skills by plugin name, which is how you get `mattpocock-skills:tdd`. Put
every skill inside one plugin and they all share one flat namespace. `cursor/plugins` ships 45
skills and `mattpocock/skills` ships 25, and two names already appear in both lists.

```
tdd
teach
```

So a single flat plugin breaks on day one, before I add a third repo. Separate plugins with a
bundle on top keep each set in its own namespace, and the collision cannot happen. The one
command you type stays the same either way, which is the whole point.

A `command` source can assemble a plugin directory on the fly, and I looked hard at it. Claude
Code refuses to install a command-sourced plugin as a dependency of another plugin, the command
runs through `cmd.exe` on Windows, and every machine has to accept it interactively. Wrong tool.

## Staying current

The `jplozancic` entry in `~/.claude/settings.json` sets `autoUpdate`, so Claude Code refreshes
the marketplace and updates installed plugins in the background after each session starts. There
is nothing to run.

```json
{
  "extraKnownMarketplaces": {
    "jplozancic": {
      "source": { "source": "github", "repo": "jplozancic/claude-marketplace" },
      "autoUpdate": true
    }
  }
}
```

`claude plugin marketplace add` writes that entry without `autoUpdate`, so add the flag on a new
machine, or toggle it under Marketplaces in `/plugin`. Third-party marketplaces have auto-update
off by default.

To force it by hand:

```bash
claude plugin marketplace update jplozancic && claude plugin update skills@jplozancic
```

## Adding a skill set

1. Add an entry to `plugins` in `.claude-plugin/marketplace.json`. Use a `github` source when the
   repo root is the plugin, `git-subdir` when the plugin sits in a subdirectory, and `url` for a
   git host that isn't GitHub.
2. Add its name to `dependencies` in `plugins/skills/.claude-plugin/plugin.json`.
3. Bump `version` in that same file. Skip this and nobody gets the new set.
4. Validate before pushing.

   ```bash
   claude plugin validate . && claude plugin validate ./plugins/skills
   ```

5. Machines pick it up on the next auto-update. To pull it now, run
   `claude plugin update skills@jplozancic` and then `/reload-plugins`.

Dependencies are bare names on purpose. A semver constraint makes Claude Code resolve against git
tags shaped like `{plugin}--v{version}`, and neither upstream tags that way, so a constrained
entry fails to resolve. Bare names track whatever the source publishes right now. Leave them
alone.

## Things that will bite you

`unslop` describes itself as "Must always apply", so it fires on any writing task rather than
waiting to be asked. It bans em dashes outright and pushes for first person and stated opinions.
I like it. If a machine disagrees, turn it off without uninstalling.

```bash
claude plugin disable unslop@jplozancic
```

Removing the bundle leaves its dependencies behind unless you say otherwise.

```bash
claude plugin uninstall skills@jplozancic --prune
```

If a machine still has `mattpocock/skills` added as its own marketplace, remove it. Otherwise the
same 25 skills load twice.

```bash
claude plugin uninstall mattpocock-skills@mattpocock && claude plugin marketplace remove mattpocock
```

Both upstreams are MIT and Claude Code fetches them from their own repos at install time. I
redistribute nothing.
