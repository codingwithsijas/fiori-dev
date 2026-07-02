# Sync Plugin

Sync `.claude/skills/` and `.claude/agents/` into `plugins/fiori-setup/` to keep the plugin up to date.

Run from the repo root:

```bash
rsync -a --delete .claude/skills/ plugins/fiori-setup/skills/
rsync -a --delete .claude/agents/ plugins/fiori-setup/agents/
```

Then show what changed:

```bash
git diff --name-only plugins/fiori-setup/
```

If there are changes, report the list of changed files and remind the user to commit:

```bash
git add plugins/fiori-setup/
git commit -m "sync plugin with .claude changes"
```

If nothing changed, say: "Plugin folder is already up to date."
