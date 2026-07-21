# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is a **Claude Code plugin template** — not a Fiori application. It ships a plugin called `fiori-dev` that users install into their own SAP Fiori / UI5 projects. The plugin provides skills, agents, and MCP server wiring for scaffolding, feature development, and code review.

There is no build system, no test runner, and no application to start. The "source of truth" for the plugin is `.claude/` — `plugins/fiori-dev/` is a synced copy.

---

## Repo structure

```text
.claude/
  skills/              ← Authoritative skill definitions (SKILL.md per skill)
  agents/              ← Authoritative agent definitions (.md per agent)
  commands/            ← Slash commands (e.g. sync-plugin.md)
  settings.json        ← Enabled plugins (project scope)
  settings.local.json  ← Allowed permissions + enabled MCP servers

plugins/fiori-dev/   ← Synced copy of .claude/skills/ and .claude/agents/
  .claude-plugin/plugin.json  ← Plugin manifest (name, version, author)
  .mcp.json                   ← MCP servers bundled with the plugin

.claude-plugin/marketplace.json  ← Local marketplace manifest (plugin registry)
.mcp.json                        ← MCP servers for this repo's own dev sessions
README.md                        ← End-user install and usage guide
```

---

## The sync workflow

`.claude/` is where all edits happen. `plugins/fiori-dev/` is never edited directly.

To push changes to the plugin folder, run:

```bash
/sync-plugin
```

This runs `rsync -a --delete` from `.claude/skills/` → `plugins/fiori-dev/skills/` and `.claude/agents/` → `plugins/fiori-dev/agents/`. Always run it before committing.

---

## Plugin contents

### Skills (slash commands)

| Skill | File | Purpose |
| --- | --- | --- |
| `/setup-prerequisites` | `.claude/skills/setup-prerequisites/SKILL.md` | Installs MCP servers and Claude Code plugins; skips if run within 7 days |
| `/project-setup` | `.claude/skills/project-setup/SKILL.md` | Two-phase: analyses requirements + recommends wizard config (Phase 1), then implements features after user scaffolds via Fiori Tools (Phase 2) |
| `/develop-fiori-feature` | `.claude/skills/develop-fiori-feature/SKILL.md` | Implements UI features post-scaffold; always consults `fiori-frontend-dev` agent before writing code |
| `/explore-odata-service` | `.claude/skills/explore-odata-service/SKILL.md` | Explores OData V4 service structure via the `odata-v4-reader` agent |
| `/ui5-code-review` | `.claude/skills/ui5-code-review/SKILL.md` | Reviews staged files; routes API validation through `fiori-frontend-dev` agent |

### Agents

| Agent | File | Purpose |
| --- | --- | --- |
| `fiori-frontend-dev` | `.claude/agents/fiori-frontend-dev.md` | Validates all Fiori/UI5 decisions against Fiori MCP and UI5 MCP servers before answering |
| `annotation-expert` | `.claude/agents/annotation-expert.md` | Writes and validates OData annotations against `localService/metadata.xml` |
| `odata-v4-reader` | `.claude/agents/odata-v4-reader.md` | Reads OData V4 service metadata and fetches live data |
| `cap-service-explorer` | `.claude/agents/cap-service-explorer.md` | Introspects CAP `.cds` files without running a server |

### MCP servers (bundled in `.mcp.json`)

| Server | Package |
| --- | --- |
| `fiori-mcp-server` | `@sap-ux/fiori-mcp-server@latest` |
| `ui5-mcp-server` | `@ui5/mcp-server` |
| `cap-mcp-server` | `@cap-js/mcp-server` |

---

## Key design rules (do not break these)

- **Fiori Tools owns Fiori Elements scaffolding.** `project-setup` never runs generators. It analyses requirements, recommends config, and waits for the user to run the Fiori Tools Application Wizard before continuing.
- **Skills never write code autonomously.** Every skill delegates implementation decisions to the `fiori-frontend-dev` agent before producing any code or annotations.
- **Annotation work always goes through `annotation-expert`.** No skill writes `annotation.xml` directly.
- **No git commits during skills.** Skills never run `git commit` or `git init` — all git operations are left to the user.
- **Skill memory lives in the target project**, not this repo. Each skill writes to `{{project_path}}/.claude/skill-memory/{{skill-name}}/memory.md` so sessions can resume after a break.

---

## Library config (`config.json` / `config.local.json`)

The plugin ships a `config.json` with default relative paths (relative to the project root):

```json
{
  "sapFeV4SourcePath": "../sap.fe",
  "sapFeV2SourcePath": "../sap.suite.ui.generic.template"
}
```

`.claude/config.json` in this repo is always kept in sync with `plugins/fiori-dev/config.json`.

If a developer's workspace layout differs, they create `config.local.json` at the **project root** with corrected paths — this file is never committed:

```json
{
  "sapFeV4SourcePath": "../../libs/sap.fe",
  "sapFeV2SourcePath": "../../libs/sap.suite.ui.generic.template"
}
```

- `config.local.json` (project root) wins on any key it defines; unset keys fall back to `.claude/config.json`.
- `setup-prerequisites` (Step 5a) resolves the effective config and checks whether each path exists on disk. Only if a path is **not found** does it prompt the user to create `config.local.json`.
- `fiori-frontend-dev` loads the effective config at Step 0 and uses the library source for development guidance and code review when a matching path resolves successfully.

---

## Versioning

Version is stored in `plugins/fiori-dev/.claude-plugin/plugin.json`.

Use standard semver:

| Change type | When to use | Example |
| --- | --- | --- |
| patch | Bug fix or correction to an existing skill/agent | `2.0.0 → 2.0.1` |
| minor | New skill or agent added | `2.0.0 → 2.1.0` |
| major | Breaking change to how a skill works | `1.x.x → 2.0.0` |

Bump the version whenever `plugins/fiori-dev/` content changes (after `/sync-plugin`). Do not bump for changes to `CLAUDE.md`, `README.md`, or repo-level config files — those do not ship to users.

Users pull updates with:

```bash
/plugin update fiori-dev@sap-fiori-toolkit
```

---

## Version history

### 1.1.1 — 2026-07-21

- Fixed: `setup-prerequisites` end message now instructs users to run `/reload-plugins` or restart Claude Code (instead of restart only)

### 1.1.0 — 2026-07-21

- Renamed: plugin `fiori-setup` → `fiori-dev`; plugin folder moved from `plugins/fiori-setup/` to `plugins/fiori-dev/`
- Changed: install command updated to `/plugin install fiori-dev@sap-fiori-toolkit --scope project`
- Changed: update command updated to `/plugin update fiori-dev@sap-fiori-toolkit`
- Changed: `CLAUDE.md` and `README.md` updated throughout to reflect new plugin name

### 1.0.5 — 2026-07-21

- Renamed: skill `fiori-feature-dev` → `develop-fiori-feature`
- Renamed: skill `odata-v4-reader` → `explore-odata-service`
- Added: `config.json` ships with the plugin — default relative paths for `sap.fe` and `sap.suite.ui.generic.template`
- Added: `setup-prerequisites` Step 5a — resolves effective config, checks library paths on disk, prompts for `config.local.json` if not found
- Added: `fiori-frontend-dev` Step 0 — loads effective config (`config.local.json` → `.claude/config.json`) at session start and uses library source for development and code review

### 1.0.4 — 2026-07-06

- Added: `fiori-feature-dev` Protocol Guarantee section (after "When to Use") — explicitly states Steps 6 and 9 are non-negotiable regardless of invocation method
- Changed: Step 6 heading now reads "Validate *(mandatory after every file write — no exceptions)*"
- Changed: Step 9 heading now reads "Update Skill Memory *(mandatory — always runs, no exceptions)*"

### 1.0.3 — 2026-07-03

- Added: `fiori-feature-dev` now generates `application.md` on first run, capturing artifact ID, app name, template type (Freestyle / Elements V2 / V4), views, controllers, fragments, util/formatter/model files, UI5 libraries, and backend dependencies (`pom.xml` for RAP/CAP Java, `package.json` for CAP Node.js)
- Added: staleness check via `git log` — `application.md` is regenerated automatically when `webapp/`, `pom.xml`, or `package.json` have new commits since the last analysis
- Changed: Step 1 now loads context from `application.md` instead of re-scanning `manifest.json` and the file system on every run

### 1.0.2 — 2026-07-02

- Fixed: `fiori-feature-dev` skill now always presents a plan and waits for user approval before writing any file (was conditional on "non-trivial")
- Fixed: skill memory skeleton is written at Step 0 if no memory file exists, preventing loss of context on session interruption
- Fixed: Step 9 memory update is now a hard gate — feature cannot be declared complete until memory is written

### 1.0.1 — 2026-07-02

- Fixed: `ONBOARDING.md` install command referenced wrong marketplace alias (`fiori-dev` → `sap-fiori-toolkit`)
- Fixed: `ONBOARDING.md` update command corrected to match marketplace alias

### 1.0.0 — 2026-07-02

Initial public release. Skills: `/setup-prerequisites`, `/project-setup`, `/fiori-feature-dev`, `/odata-v4-reader`, `/ui5-code-review`. Agents: `fiori-frontend-dev`, `annotation-expert`, `odata-v4-reader`, `cap-service-explorer`. MCP servers: `fiori-mcp-server`, `ui5-mcp-server`, `cap-mcp-server`.

---

## End-user install instructions

See `README.md` for install instructions and full feature documentation.
