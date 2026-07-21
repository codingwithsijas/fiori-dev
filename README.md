# fiori-dev — SAP Fiori / UI5 Claude Code Plugin

A Claude Code plugin that brings SAP Fiori and UI5 development skills, specialist agents, and three MCP servers into any project. Every implementation decision is validated against live MCP tooling before code is written.

**Current version:** 1.1.0 · **Author:** Mohammed Sijas

---

## Prerequisites

- [Claude Code](https://claude.ai/code) installed (`claude --version`)
- Node / npm available (`node --version`)

---

## Install

Run these two commands inside Claude Code — no cloning, no copying files:

**Step 1 — Add the marketplace:**
```bash
/plugin marketplace add https://github.com/codingwithsijas/fiori-dev
```

**Step 2 — Install the plugin:**
```bash
/plugin install fiori-dev@sap-fiori-toolkit --scope project
```

To activate the plugin, either run `/reload-plugins` in Claude Code or restart Claude Code.

---

## Skills

Skills are slash commands you type directly in Claude Code.

### `/setup-prerequisites`

Installs all required MCP servers (`fiori-mcp-server`, `ui5-mcp-server`, `cap-mcp-server`) and the UI5 Claude Code plugins. Skips automatically if run within the last 7 days.

Run this once before starting any Fiori/UI5 work in a new environment.

---

### `/project-setup`

Two-phase project bootstrapper:

1. **Phase 1 — Analysis:** asks for your requirements (or reads a `requirements.md` file you provide), recommends the correct floorplan and Fiori Tools Application Wizard config.
2. **Phase 2 — Implementation:** after you run the wizard and provide the generated project path, implements the initial feature set with full validation.

Supports all templates: List Report Object Page, Worklist, Overview Page, Analytical List Page, Form Entry Object Page, Flexible Programming Model, and Fiori Freestyle Basic.

**Optional `requirements.md`** (place in your app project root to skip interactive questions):

```markdown
# Project Requirements

App Type:     # Fiori Elements / Fiori Freestyle
Template:     # Worklist / List Report / Basic / Master-Detail
Main Entity:  # OData entity set name (case-sensitive)
Service URL:  # Full OData service URL (V2 or V4)
Backend:      # OData V2 / OData V4 / CAP / RAP / mock
Namespace:    # e.g. com.myorg.myapp
TypeScript:   # yes / no
Deployment:   # BTP / ABAP / local
```

---

### `/develop-fiori-feature`

Implements UI features in an existing scaffold. Works for Fiori Freestyle and Fiori Elements on any backend.

Covers:

- XML views and fragments
- Controllers (JavaScript and TypeScript)
- OData model bindings (V2 and V4)
- Routing and navigation targets in `manifest.json`
- i18n keys and translations
- Fiori Elements annotations (delegates to `annotation-expert`)

Every run follows a strict 10-step protocol:

| Step | What happens |
| ---- | ------------ |
| 0    | Reads or regenerates `application.md` (cached project analysis); writes skill memory skeleton if absent |
| 1    | Loads project context from `application.md` + UI5 guidelines |
| 2    | Reads existing files the feature will touch |
| 3    | Consults `fiori-frontend-dev` agent to validate approach against MCP servers |
| 4    | Presents implementation plan — **waits for user approval before writing anything** |
| 5    | Implements views, controllers, i18n, routing |
| 6    | Runs UI5 linter and manifest validation — **mandatory after every file write** |
| 7    | Delegates annotation work to `annotation-expert` if needed |
| 9    | Updates skill memory — **mandatory before declaring feature done** |
| 10   | Reports summary of changes to the user |

---

### `/explore-odata-service`

Explores an OData V4 service before building against it. Returns entity structure, key properties, associations, navigation properties, and can fetch live data records.

Useful for understanding an unfamiliar backend before writing binding paths or `$expand` expressions.

---

### `/ui5-code-review`

Reviews staged files against UI5 best practices. Routes all API validation through the `fiori-frontend-dev` agent, which checks against the UI5 MCP server for deprecated APIs, correct control usage, and binding patterns.

---

## Agents

Agents are invoked automatically by skills — you do not call them directly. They are listed here so you understand what is running on your behalf.

| Agent | Invoked by | Role |
| ----- | ---------- | ---- |
| `fiori-frontend-dev` | All skills | Consults `fiori-mcp-server` and `ui5-mcp-server` to validate every API, control, and pattern before code is written. Acts as a guard against deprecated or incorrect usage. |
| `annotation-expert` | `develop-fiori-feature` | Writes and validates Fiori Elements OData annotations (`UI.LineItem`, `UI.FieldGroup`, `Common.ValueList`, etc.). Validates every property path against `localService/metadata.xml` before writing. |
| `odata-v4-reader` | `explore-odata-service` skill, `develop-fiori-feature` | Reads OData V4 service metadata and fetches live data via the service URL. Used to inform binding paths and `$expand` expressions. |
| `cap-service-explorer` | `develop-fiori-feature` (CAP projects) | Introspects CAP `.cds` files to extract entity definitions, service exposures, associations, and OData endpoints — without requiring a running server. |

---

## MCP Servers

Three MCP servers are bundled and activated automatically when the plugin is installed.

### `fiori-mcp-server` — `@sap-ux/fiori-mcp-server`

SAP's official Fiori tooling server. Provides:

- Fiori Elements app generation (all floorplans)
- Modification operations on existing apps (add page, add controller extension, change manifest settings)
- Documentation search for Fiori Elements annotations and patterns

Used by: `fiori-frontend-dev` agent, `project-setup` skill.

---

### `ui5-mcp-server` — `@ui5/mcp-server`

SAPUI5/OpenUI5 tooling server. Provides:

- UI5 API reference lookup (any module, class, property, event)
- UI5 linter (`run_ui5_linter`) — checks for deprecated APIs and best-practice violations
- Manifest validation (`run_manifest_validation`)
- App scaffolding (`create_ui5_app`)
- Integration card creation (`create_integration_card`)
- Version information

Used by: all skills and agents.

---

### `cap-mcp-server` — `@cap-js/mcp-server`

CAP (Cloud Application Programming Model) tooling server. Provides:

- CDS model search — returns entity definitions, elements, annotations, and HTTP endpoints
- Documentation search for CAP Node.js and Java APIs

Used by: `cap-service-explorer` agent, `develop-fiori-feature` skill (CAP projects).

---

## Workflow Examples

### Start a brand-new Fiori app

```text
1. /setup-prerequisites        ← installs MCP servers (once per environment)
2. /project-setup               ← analyses requirements, gives you wizard config
3. Run Fiori Tools App Wizard   ← you scaffold the project
4. /project-setup               ← resumes, implements initial feature
5. /develop-fiori-feature       ← add more features iteratively
```

### Add a feature to an existing app

```text
/develop-fiori-feature    ← describe the feature; the skill handles the rest
```

### Explore an OData V4 service

```text
/explore-odata-service      ← provide the service URL; get back entity structure and sample data
```

### Review code before a PR

```text
/ui5-code-review      ← reviews staged changes against UI5 best practices
```

---

## Keep the Plugin Up to Date

```bash
/plugin update fiori-dev@sap-fiori-toolkit
```

---

## Repository Layout

```text
.claude/
  skills/          ← authoritative skill definitions (edit here)
  agents/          ← authoritative agent definitions (edit here)
  commands/        ← slash commands (e.g. sync-plugin)
  settings.json

plugins/fiori-dev/   ← synced copy — never edit directly
  .claude-plugin/plugin.json   ← name, version, author
  .mcp.json                    ← bundled MCP server config

ONBOARDING.md      ← end-user quick-start
CLAUDE.md          ← contributor and maintainer guide
```

To push edits from `.claude/` into the plugin folder:

```bash
/sync-plugin
```

---

## Contributing

1. Edit skills in `.claude/skills/` or agents in `.claude/agents/`
2. Run `/sync-plugin` to copy changes to `plugins/fiori-dev/`
3. Bump the version in `plugins/fiori-dev/.claude-plugin/plugin.json` (semver patch/minor/major)
4. Add a version entry to `CLAUDE.md`
5. Commit and push

See `CLAUDE.md` for versioning rules and design constraints.
