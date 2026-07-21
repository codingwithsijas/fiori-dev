---
name: setup-prerequisites
description: Install all MCP servers and Claude Code plugins required for SAP Fiori / UI5 development in this project. Run once before starting any Fiori/UI5 work to ensure fiori-mcp-server, ui5-mcp-server, cap-mcp-server, and the ui5 and ui5-typescript-conversion plugins are all present.
---

# Setup Prerequisites

Install every tool this project needs before any Fiori/UI5 development work begins.

## What Gets Installed

### MCP Servers (via npm global)
| Package | Purpose |
|---|---|
| `@sap-ux/fiori-mcp-server` | Fiori Elements scaffolding, annotations, floor-plan guidance |
| `@ui5/mcp-server` | UI5 API reference, linting, project creation, guidelines |
| `@cap-js/mcp-server` | CAP service introspection (recommended companion) |

### Claude Code Plugins (project scope)
| Plugin | Agents / Skills it provides |
|---|---|
| `ui5@claude-plugins-official` | `ui5-best-practices`, `ui5-best-practices-integration-cards` skills |
| `ui5-typescript-conversion@claude-plugins-official` | `ui5-typescript-conversion` skill |
| `feature-dev@claude-plugins-official` | `code-architect`, `code-explorer`, `code-reviewer` agents |
| `code-modernization@claude-plugins-official` | `security-auditor`, `architecture-critic`, `legacy-analyst`, `test-engineer`, `business-rules-extractor` agents |
| `pr-review-toolkit@claude-plugins-official` | `code-reviewer`, `silent-failure-hunter`, `type-design-analyzer`, `comment-analyzer`, `pr-test-analyzer` agents |

---

## Step 0 — Read Skill Memory

Check for an existing memory file for this skill:

```bash
cat .claude/skill-memory/setup-prerequisites/memory.md 2>/dev/null
```

If found and the last run was recent (within 7 days), present the cached status table to the user and ask: "Prerequisites were last verified on `{{date}}`. Re-check now or skip?" If the user says skip, exit. Otherwise proceed.

If not found or outdated, proceed with a fresh check.

---

## Step 1 — Check Existing Installation

Run these checks first to avoid reinstalling what's already present.

```bash
npm list -g @sap-ux/fiori-mcp-server @ui5/mcp-server @cap-js/mcp-server --depth=0 2>/dev/null
claude plugin list 2>/dev/null
```

Report what is already installed and what is missing before proceeding.

---

## Step 2 — Install MCP Servers

Install any missing packages globally:

```bash
npm install -g @sap-ux/fiori-mcp-server@latest @ui5/mcp-server@latest @cap-js/mcp-server@latest
```

After installation, confirm versions:

```bash
npm list -g @sap-ux/fiori-mcp-server @ui5/mcp-server @cap-js/mcp-server --depth=0
```

---

## Step 3 — Install Claude Code Plugins at Project Scope

Install both plugins scoped to this project (writes to `.claude/settings.json`):

```bash
claude plugin install ui5@claude-plugins-official --scope project
claude plugin install ui5-typescript-conversion@claude-plugins-official --scope project
claude plugin install feature-dev@claude-plugins-official --scope project
claude plugin install code-modernization@claude-plugins-official --scope project
claude plugin install pr-review-toolkit@claude-plugins-official --scope project
```

If a plugin is already installed at project scope, skip it — do not reinstall.

---

## Step 4 — Verify .mcp.json

Check that `.mcp.json` exists at the project root and contains all three servers:

```bash
cat .mcp.json
```

Expected content:
```json
{
  "mcpServers": {
    "ui5-mcp-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@ui5/mcp-server"]
    },
    "fiori-mcp-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["--yes", "@sap-ux/fiori-mcp-server@latest", "fiori-mcp"]
    },
    "cap-mcp-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@cap-js/mcp-server"]
    }
  }
}
```

If `.mcp.json` is missing or incomplete, create or update it to match the above.

---

## Step 5a — Check Library Config

Resolve the effective config — project-local override wins, plugin default is the fallback:

1. `config.local.json` at the project root — user's local override
2. The plugin's own `config.json` — default, shipped with the plugin (`{{skill_base_dir}}/../config.json`)

```bash
cat config.local.json 2>/dev/null || cat "{{skill_base_dir}}/../config.json"
```

Then verify that the resolved paths actually exist on disk (relative to the project root):

```bash
ls "$(node -e "
  const fs=require('fs');
  const c=fs.existsSync('./config.local.json') ? require('./config.local.json') :
          require('{{skill_base_dir}}/../config.json');
  process.stdout.write(c.sapFeV4SourcePath)")" 2>/dev/null && echo "V4 OK" || echo "V4 NOT FOUND"

ls "$(node -e "
  const fs=require('fs');
  const c=fs.existsSync('./config.local.json') ? require('./config.local.json') :
          require('{{skill_base_dir}}/../config.json');
  process.stdout.write(c.sapFeV2SourcePath)")" 2>/dev/null && echo "V2 OK" || echo "V2 NOT FOUND"
```

**If both paths resolve** → show the effective paths and which config file was used, done.

**If one or both paths are not found**: inform the user which library could not be found at the default location and ask them to provide the correct relative path(s):

| Key | Purpose |
|---|---|
| `sapFeV4SourcePath` | Path to locally cloned `sap.fe` repo, relative to the project root (e.g. `../sap.fe`). Used by `fiori-frontend-dev` for Fiori Elements V4 projects. |
| `sapFeV2SourcePath` | Path to locally cloned `sap.suite.ui.generic.template` repo, relative to the project root (e.g. `../sap.suite.ui.generic.template`). Used for Fiori Elements V2 projects. |

Write the corrected paths to `config.local.json` in the project root:

```json
{
  "sapFeV4SourcePath": "{{user-provided path}}",
  "sapFeV2SourcePath": "{{user-provided path}}"
}
```

Remind the user to add `config.local.json` to `.gitignore` — paths are relative to each developer's workspace layout and should not be committed.

> **Note**: Providing corrected paths is optional. If the user skips this, `fiori-frontend-dev` will still work — it simply won't consult the library source during development and code review.

---

## Step 5 — Report Results

Print a summary table:

| Component | Status | Version |
|---|---|---|
| `@sap-ux/fiori-mcp-server` | ✓ installed / ✗ missing | x.x.x |
| `@ui5/mcp-server` | ✓ installed / ✗ missing | x.x.x |
| `@cap-js/mcp-server` | ✓ installed / ✗ missing | x.x.x |
| `ui5` plugin | ✓ project scope / ✗ missing | x.x.x |
| `ui5-typescript-conversion` plugin | ✓ project scope / ✗ missing | x.x.x |
| `feature-dev` plugin | ✓ project scope / ✗ missing | x.x.x |
| `code-modernization` plugin | ✓ project scope / ✗ missing | x.x.x |
| `pr-review-toolkit` plugin | ✓ project scope / ✗ missing | x.x.x |
| `.mcp.json` | ✓ complete / ✗ incomplete | — |
| `config.local.json` | ✓ configured / ⚠ using defaults / ✗ not set | — |

End with:
> To activate the MCP servers and plugins, either run `/reload-plugins` in Claude Code or restart Claude Code.

---

## Step 6 — Update Skill Memory

Create or update `.claude/skill-memory/setup-prerequisites/memory.md`:

```markdown
# setup-prerequisites memory

## Last run: {{date}}

## Installed versions
| Component | Version |
|---|---|
| `@sap-ux/fiori-mcp-server` | x.x.x |
| `@ui5/mcp-server` | x.x.x |
| `@cap-js/mcp-server` | x.x.x |
| `ui5` plugin | x.x.x |
| `ui5-typescript-conversion` plugin | x.x.x |
| `feature-dev` plugin | x.x.x |
| `code-modernization` plugin | x.x.x |
| `pr-review-toolkit` plugin | x.x.x |

## Notes
{{Any issues encountered during installation, workarounds applied, or manual steps required}}
```
