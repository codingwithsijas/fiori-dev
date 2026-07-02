# Fiori Project Setup — Onboarding

Get the full SAP Fiori / UI5 development toolkit (skills, agents, MCP servers) in two commands — no cloning, no copying files.

---

## Prerequisites

- [Claude Code](https://claude.ai/code) installed (`claude --version`)
- `node` / `npm` available (`node --version`)

---

## Install the Plugin

**Step 1 — Clone the repo (run in your terminal, one-time):**

```bash
git clone https://github.com/codingwithsijas/fiori-dev.git ~/fiori-dev
```

**Step 2 — Add the marketplace and install (run in Claude Code):**

```bash
/plugin marketplace add ~/fiori-dev
/plugin install fiori-setup@sap-fiori-toolkit
```

Restart Claude Code when prompted.

---

## What Gets Installed

### Skills

| Skill | Trigger | Purpose |
| --- | --- | --- |
| `setup-prerequisites` | `/setup-prerequisites` | Installs MCP servers and Claude Code plugins |
| `project-setup` | `/project-setup` | Analyses requirements, recommends floorplan + wizard config, then implements features once user scaffolds the project via Fiori Tools |
| `odata-v4-reader` | `/odata-v4-reader` | Explores OData V4 service structure before building |
| `fiori-feature-dev` | `/fiori-feature-dev` | Implements UI features post-scaffold (views, controllers, routing, i18n) |
| `ui5-code-review` | `/ui5-code-review` | Reviews code against UI5 best practices |

### Agents

| Agent | Purpose |
| --- | --- |
| `fiori-frontend-dev` | Researches Fiori/UI5 patterns against MCP servers before code is written |
| `annotation-expert` | Writes and validates Fiori Elements OData annotations |
| `odata-v4-reader` | Explores OData V4 service metadata and entity structure |
| `cap-service-explorer` | Introspects CAP service definitions and endpoints |

### MCP Servers (auto-activated)

| Server | Purpose |
| --- | --- |
| `fiori-mcp-server` | Fiori Elements scaffolding and floor-plan guidance |
| `ui5-mcp-server` | UI5 API reference, linting, and project creation |
| `cap-mcp-server` | CAP service introspection |

---

## Start a New Project

**Run `/project-setup`** — Claude will ask you for the project details interactively (app type, backend, entity set, service URL, etc.), then scaffold the app and run all validations automatically.

Optionally, you can create a `requirements.md` in your app project root to pre-fill the answers and skip the questions:

```markdown
# Project Requirements

App Type:     # Fiori Elements / Fiori Freestyle
Template:     # Worklist / List Report / Basic / Master-Detail
Main Entity:  # Main OData entity set name (case-sensitive)
Service URL:  # Full OData service URL (V2 or V4)
Backend:      # OData V2 / OData V4 / CAP / RAP / mock
Namespace:    # e.g. com.myorg.myapp
TypeScript:   # yes / no
Deployment:   # BTP / ABAP / local
```

If the file exists, the skill reads it and only asks about missing fields.

---

## Work on an Existing Project

If a `webapp/` folder already exists, skip `/project-setup`. Use the skills directly:

| Task | Command |
| --- | --- |
| Add a view, table, form, or navigation | `/fiori-feature-dev` |
| Review code against UI5 best practices | `/ui5-code-review` |

---

## Keeping the Plugin Up to Date

Pull the latest changes in your terminal, then update the plugin in Claude Code:

```bash
# In your terminal
cd ~/fiori-dev && git pull
```

```bash
# In Claude Code
/plugin update fiori-setup@fiori-toolkit
```
