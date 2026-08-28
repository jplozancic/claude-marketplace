# claude-marketplace

A private [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces).
It holds a manifest that points at skill repos, plus any custom plugins kept in this repo.
Installing one plugin, `skills`, installs everything listed here.

## What's in it

| Plugin | Skills | Source | Author | License |
|---|---|---|---|---|
| `skills` | 0 | `./plugins/skills` in this repo | | |
| `unslop` | 1 | [`cursor/plugins`](https://github.com/cursor/plugins/tree/main/pstack/skills/unslop), path `pstack/skills/unslop` | Lauren Tan | MIT |
| `mattpocock-skills` | 25 | [`mattpocock/skills`](https://github.com/mattpocock/skills) | Matt Pocock | MIT |

`skills` is the bundle. Its manifest is a name, a version and a `dependencies` array. Installing
it installs every plugin it lists.

Skills are namespaced by plugin name, so they appear as `unslop:unslop`,
`mattpocock-skills:tdd`, and so on.

## Repository layout

```
.claude-plugin/
  marketplace.json              the plugin list
plugins/
  skills/
    .claude-plugin/plugin.json  the bundle manifest
README.md
```

## Install

```bash
claude plugin marketplace add jplozancic/claude-marketplace
```

```bash
claude plugin install skills@jplozancic --scope user
```

The second command prints `+ 2 dependencies` and installs `unslop` and `mattpocock-skills`.
Restart the session or run `/reload-plugins` to load them.

The repo is private, so the first command needs GitHub credentials on the machine. Claude Code
tries SSH and falls back to HTTPS. Run `gh auth login` if neither is configured.

Confirm with `claude plugin list`, which should show three enabled plugins.

## Update

Add `"autoUpdate": true` to the marketplace entry in `~/.claude/settings.json` and Claude Code
refreshes the marketplace and updates every installed plugin in the background after each session
starts.

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

`claude plugin marketplace add` writes that entry without the flag. Add it by hand, or toggle it
under Marketplaces in `/plugin`.

To update by hand:

```bash
claude plugin marketplace update jplozancic && claude plugin update skills@jplozancic
```

## Remove

`--prune` also removes the dependencies that no other plugin needs.

```bash
claude plugin uninstall skills@jplozancic --prune
```

To remove the marketplace as well:

```bash
claude plugin marketplace remove jplozancic
```

## Add a hosted plugin

For a skill set that lives in someone else's repo.

1. Add an entry to `plugins` in `.claude-plugin/marketplace.json`. Pick the `source` that matches
   the upstream layout.

   | Upstream layout | Source |
   |---|---|
   | Repo root is the plugin | `{"source": "github", "repo": "owner/repo"}` |
   | Plugin sits in a subdirectory | `{"source": "git-subdir", "url": "https://github.com/owner/repo.git", "path": "a/b"}` |
   | Git host other than GitHub | `{"source": "url", "url": "https://gitlab.com/team/repo.git"}` |

   ```json
   {
     "name": "example-skills",
     "description": "What it does.",
     "source": { "source": "github", "repo": "owner/repo" }
   }
   ```

2. Add the plugin name to `dependencies` in `plugins/skills/.claude-plugin/plugin.json`.
3. Bump `version` in that same file.
4. Validate, then push.

   ```bash
   claude plugin validate . && claude plugin validate ./plugins/skills
   ```

Leave dependencies as bare names. A semver constraint resolves against git tags shaped like
`{plugin}--v{version}`, which these upstreams do not publish.

## Add a custom skill

For a skill written here rather than fetched from elsewhere. Group related skills into one plugin.

1. Create the plugin directory and its manifest.

   ```
   plugins/my-plugin/
     .claude-plugin/plugin.json
     skills/
       my-skill/
         SKILL.md
   ```

   ```json
   {
     "name": "my-plugin",
     "version": "1.0.0",
     "description": "What these skills do.",
     "author": { "name": "Peter Lozancic" }
   }
   ```

2. Write `SKILL.md` with YAML frontmatter. The `description` decides when Claude loads the skill.

   ```markdown
   ---
   name: my-skill
   description: What this does and when to use it.
   ---

   # My skill

   Instructions go here.
   ```

3. Add an entry to `plugins` in `.claude-plugin/marketplace.json` using a relative path.

   ```json
   {
     "name": "my-plugin",
     "source": "./plugins/my-plugin",
     "description": "What these skills do."
   }
   ```

4. Add `my-plugin` to `dependencies` in `plugins/skills/.claude-plugin/plugin.json`, then bump the
   bundle's `version`.
5. Validate, then push.

   ```bash
   claude plugin validate . && claude plugin validate ./plugins/my-plugin
   ```

Bump the plugin's own `version` whenever you change its skills, otherwise machines keep the copy
they already have.

## Picking up changes

Machines with `autoUpdate` on get new plugins after the next session start. To pull immediately:

```bash
claude plugin update skills@jplozancic
```

Then run `/reload-plugins`.
