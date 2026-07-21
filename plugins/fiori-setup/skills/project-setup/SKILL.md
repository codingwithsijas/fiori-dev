---
name: project-setup
description: Bootstrap a new SAP Fiori / UI5 project. Analyses requirements, recommends the correct floorplan and project config for the user to run in the Fiori Tools Application Wizard, then resumes after the user provides the generated project path to plan and implement the feature requirements.
---

# Project Setup

This skill runs in two distinct phases separated by a user action (running the Fiori Tools Application Wizard).

**Phase 1** — Read requirements → consult `fiori-frontend-dev` → produce a ready-to-use wizard config for the user.
**Phase 2** — User provides the generated project path → skill resumes, writes `plan.md`, implements features via `develop-fiori-feature`.

> **Fiori Tools owns project scaffolding.** This skill never runs generators or creates project files. It analyses, recommends, and then builds on top of what Fiori Tools generated.

---

## Phase 1 — Analyse Requirements

---

### Step 1 — Run Prerequisites Check

```skill
setup-prerequisites
```

Resolve any missing tools before continuing.

---

### Step 2 — Read or Gather Requirements

Check for a `requirements.md` in the current directory:

```bash
ls requirements.md 2>/dev/null || ls REQUIREMENTS.md 2>/dev/null
```

**If found**: Read it in full. Extract every field listed below. Present a summary of what was found and only ask the user to fill in what is missing.

**If not found**: Ask the user for the following. Collect all answers before proceeding — do not ask one at a time.

| Field | Question |
|---|---|
| **App goal** | What does this app need to do? (one sentence) |
| **Backend type** | OData V2, OData V4, CAP, RAP, or mock? |
| **Service URL** | Full OData service URL (leave blank for mock/offline) |
| **Main entity** | The primary OData entity set the app will work with |
| **Features** | What should the app do? (e.g. list orders, view order details, create orders) |
| **TypeScript** | Yes or no? |
| **App namespace** | e.g. `com.myorg.myapp` |
| **Target** | BTP, on-premise ABAP, or local dev only? |

---

### Step 3 — Explore the Service

Before any analysis, understand the service structure.

**OData V4**: Invoke the `odata-v4-reader` agent with the service URL. Ask it to return entity sets, key fields, navigation properties, draft enablement, and relevant SAP annotations (`@UI.*`, `@Common.*`). Present its output to the user directly — do not reformat it.

**OData V2**: Use `download_odata_service_metadata` to fetch the EDMX:

```tool
mcp__fiori-mcp-server__download_odata_service_metadata
  sapSystemQuery: "{{service_url}}"
  servicePath:    "{{service_path}}"
  appPath:        "{{current_directory}}"
```

> **If the download fails due to an authorisation error**: Ask the user to add the system via the **Service Manager** in SAP Fiori Tools (VS Code: *SAP Fiori Tools — Service Manager* panel, BAS: *Service Centre*), then re-run. Do not ask for credentials directly.

**CAP**: Invoke the `cap-service-explorer` agent. Ask it to return all exposed services, entity sets, key properties, associations, and existing UI annotations.

**Mock / offline**: Skip this step. The service structure will be defined after scaffolding.

---

### Step 4 — Analyse and Recommend with `fiori-frontend-dev`

**Always** invoke the `fiori-frontend-dev` agent for this step — never decide the floorplan or project config autonomously.

```agent
fiori-frontend-dev
```

Pass it the complete picture:
- App goal and listed features (from Step 2)
- Service structure (from Step 3)
- Backend type and OData version
- TypeScript preference
- Deployment target

Ask it to recommend:
1. **Floorplan** — which Fiori Elements template (List Report Object Page, Worklist, ALP, Overview Page, Form Entry Object Page) or Freestyle pattern best fits the requirements, and why
2. **Application name** — a descriptive, user-facing title (e.g. "Sales Order Management")
3. **Artifact ID / project name** — lowercase with dashes, suitable as a folder name (e.g. `sales-order-management`)
4. **Namespace** — confirm or derive from the user's input (e.g. `com.myorg.salesordermgmt`)
5. **Main entity** — confirm the correct entity set name (case-sensitive, from Step 3)
6. **Navigation entity** (if LROP) — the entity for the Object Page
7. **Feature implementation plan** — a brief ordered list of what needs to be built post-scaffold (e.g. "1. Configure List Report columns, 2. Add Object Page header, 3. Enable create action")

If the agent recommends a floorplan different from what the user expected, explain the reason clearly before presenting the output.

---

### Step 5 — Present Wizard Configuration to the User

Present the agent's recommendations as a ready-to-use configuration block:

---

> ## Fiori Tools Application Wizard — Configuration
>
> Please install and open the **SAP Fiori Tools Extension Pack** if you haven't already:
> 👉 [marketplace.visualstudio.com — SAPSE.sap-ux-fiori-tools-extension-pack](https://marketplace.visualstudio.com/items?itemName=SAPSE.sap-ux-fiori-tools-extension-pack)
>
> Then open the Application Wizard:
> - **VS Code**: Command Palette → *Fiori: Open Application Generator*
> - **BAS**: *Application Generator* tile in the welcome screen
>
> Enter the following values:
>
> | Wizard Field | Value |
> |---|---|
> | Template | `{{floorplan}}` |
> | Data source | `{{service URL or destination}}` |
> | Main entity | `{{mainEntity}}` |
> | Navigation entity | `{{navigationEntity}}` (if applicable) |
> | Application title | `{{applicationName}}` |
> | Application namespace | `{{namespace}}` |
> | Project folder name | `{{artifactId}}` |
> | TypeScript | `{{yes / no}}` |
> | Target folder | *(choose your workspace folder)* |
>
> The wizard will scaffold the full project including `metadata.xml`, `manifest.json`, `localService/`, and initial annotations.
>
> **Once the wizard finishes, come back and tell me the path to the generated project folder** — e.g. `/Users/you/projects/sales-order-management`. The skill will then continue with feature implementation.

---

> ⏸ **Waiting for user to run the Application Wizard and provide the project path.**

---

## Phase 2 — Plan and Implement

*Resume here once the user provides the generated project path.*

---

### Step 6 — Verify the Generated Project

Confirm the project was scaffolded correctly before planning any work.

Run in parallel:

```tool
mcp__ui5-mcp-server__get_project_info
  projectDir: "{{project_path}}"
```

```tool
mcp__ui5-mcp-server__run_manifest_validation
  manifestPath: "{{project_path}}/webapp/manifest.json"
```

```tool
mcp__ui5-mcp-server__run_ui5_linter
  projectDir:              "{{project_path}}"
  fix:                     false
  provideContextInformation: true
```

```tool
mcp__ide__getDiagnostics
```

Report any errors found. Fix manifest and linter issues by delegating to `fiori-frontend-dev` — do not patch files directly. Only proceed to Step 7 once validation is clean.

---

### Step 7 — Write plan.md

Create `{{project_path}}/plan.md` with the full implementation plan derived from the requirements and the `fiori-frontend-dev` agent's feature list (Step 4, item 7).

Structure:

```markdown
# Implementation Plan — {{applicationName}}

## Project
- Floorplan: {{floorplan}}
- Namespace: {{namespace}}
- Entity: {{mainEntity}}
- Backend: {{backendType}} / {{odataVersion}}
- Generated: {{date}}

## Features

### [ ] 1. {{Feature name}}
{{What needs to be done — controls, bindings, annotations, routing changes}}

### [ ] 2. {{Feature name}}
...

## Decisions
{{Any floorplan or architectural decisions made in Phase 1 and the reasoning behind them}}

## Status
Phase 1 complete. Phase 2 in progress.
```

Mark each feature `[ ]` (pending), `[x]` (done), or `[~]` (in progress). Update this file as work progresses.

---

### Step 8 — Write skill memory

Create `{{project_path}}/.claude/skill-memory/project-setup/memory.md` to record what has been done, so the skill can resume correctly if the session breaks.

```markdown
# project-setup memory — {{applicationName}}

## Phase 1 (complete)
- Requirements source: {{requirements.md | user input}}
- Service explored: {{yes / no / mock}}
- Floorplan recommended: {{floorplan}}
- Agent consulted: fiori-frontend-dev ✓

## Phase 2
- Project path: {{project_path}}
- Validation: {{clean / issues found — list them}}
- plan.md written: {{yes / no}}
- Current feature: {{feature name or "not started"}}

## Completed features
{{List features marked [x] in plan.md}}

## Pending features
{{List features marked [ ] in plan.md}}

## Notes
{{Any project-specific quirks, workarounds, or decisions to remember}}
```

Update this file after each completed feature.

---

### Step 9 — Implement Features via `develop-fiori-feature`

Work through the features in `plan.md` one at a time, in order.

For each feature:

1. Mark it `[~]` (in progress) in `plan.md`
2. Invoke the `develop-fiori-feature` skill with:
   - The feature description
   - Project path, namespace, entity set, OData version
   - Metadata location: `{{project_path}}/webapp/localService/metadata.xml`
   - Any constraints from the plan

```skill
develop-fiori-feature
```

3. Once `develop-fiori-feature` reports the feature complete, mark it `[x]` in `plan.md`
4. Update `memory.md` — move the feature from pending to completed
5. Proceed to the next feature

**Do not implement features directly** — all views, controllers, bindings, routing, i18n, and annotations go through `develop-fiori-feature`, which delegates annotation work further to `annotation-expert`.

---

### Step 10 — Post-Implementation Checklist

When all features in `plan.md` are marked `[x]`:

- [ ] All features implemented and validated by `develop-fiori-feature`
- [ ] `plan.md` shows all items `[x]`
- [ ] `memory.md` up to date
- [ ] Linter reports zero findings
- [ ] No IDE type errors
- [ ] `manifest.json` valid
- [ ] No locale-specific i18n files present

---

### Step 11 — Final Output

Deliver to the user:

1. **Summary** — what was built, which features are complete
2. **`plan.md` final state** — link or print the completed plan
3. **How to run the app** — `npm start` or `ui5 serve` command for local preview
4. **Next steps** — e.g. "Run `/ui5-code-review` for a final code quality check before committing"
