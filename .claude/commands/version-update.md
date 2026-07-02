# Version Update

Bump the plugin version, update `CLAUDE.md` version history, and sync the plugin folder.

## Step 1 — Check if plugin files have changed

Run sync first and check for differences:

```bash
rsync -a --delete .claude/skills/ plugins/fiori-setup/skills/
rsync -a --delete .claude/agents/ plugins/fiori-setup/agents/
git diff --name-only plugins/fiori-setup/
git status --short plugins/fiori-setup/
```

Also check if `.mcp.json` inside the plugin folder has changed:

```bash
git diff plugins/fiori-setup/.mcp.json
```

Plugin files that warrant a version bump:
- Any file under `plugins/fiori-setup/skills/`
- Any file under `plugins/fiori-setup/agents/`
- `plugins/fiori-setup/.mcp.json` — MCP server additions, removals, or version changes

**If none of the above have changed** — say: "No plugin files have changed. Version bump not needed." and stop.

**If any have changed** — show the list and continue to Step 2.

## Step 2 — Ask for version and changelog

Ask the user:
1. **New version number** — e.g. `1.0.2`, `1.1.0`, `2.0.0`
2. **What changed** — a brief description to record in the version history (can be split into Breaking / New / Fixed / Removed sections if relevant)

Do not proceed until both are provided.

## Step 3 — Determine change type

Confirm the semver bump matches the change type:

| Change | Expected bump |
| --- | --- |
| Doc fix, wording, no behaviour change | patch |
| New skill or agent added | minor |
| New or updated MCP server in `.mcp.json` | minor |
| Breaking change to skill behaviour | major |
| MCP server removed | major |

If the bump doesn't match, flag it to the user before continuing.

## Step 4 — Bump version in plugin.json

Edit `plugins/fiori-setup/.claude-plugin/plugin.json` to set the new version.

## Step 5 — Add version history entry to CLAUDE.md

Prepend a new entry under `## Version history` in `CLAUDE.md`:

```markdown
### {{version}} — {{today's date YYYY-MM-DD}}

#### {{Breaking changes | New | Fixed | Removed}}
- {{change description}}
```

Only include sections that apply. Keep entries concise — one line per change.

## Step 6 — Report and remind

Show a summary:

```
Changed files:    {{list}}
Version bumped:   {{old}} → {{new}}
CLAUDE.md:        ✓
Plugin synced:    ✓
```

Remind the user to commit and push:

```bash
git add .
git commit -m "release v{{version}}"
git push
```
