---
name: fiori-feature-dev
description: Implement any UI feature in a Fiori Freestyle or Fiori Elements project post-setup — views, controllers, OData bindings, routing, i18n, fragments, and annotations. Validates every implementation decision against the UI5 and Fiori MCP servers before writing code. Use for all development work after the project scaffold exists.
---

# Fiori Feature Dev

Implement UI features in an existing SAP Fiori / UI5 project. Every decision is validated against the UI5 and Fiori MCP servers before any file is written. Works for Fiori Freestyle and Fiori Elements projects on any backend (OData V2/V4, CAP, RAP, mock).

## When to Use
- Adding views, fragments, or XML controls
- Writing or updating controllers and event handlers
- Configuring OData model bindings
- Adding routing targets and navigation
- Managing i18n keys and translations
- Writing Fiori Elements annotations (delegates to `annotation-expert`)
- Any UI development work that follows project scaffolding

---

## Step 0 — Read Skill Memory

Before doing anything else, check for an existing memory file for this skill in the project:

```bash
cat {{absolute_path}}/.claude/skill-memory/fiori-feature-dev/memory.md 2>/dev/null
```

If found, read it in full. Use it to:
- Resume a feature that was in progress when the session broke (`[~]` status)
- Avoid re-implementing work already marked done
- Apply any project-specific patterns, quirks, or decisions recorded in previous sessions

If not found, **create the file now** (skeleton only — fill in project details after Step 1):

```markdown
# fiori-feature-dev memory — (project name TBD)

## Project
(populated after Step 1)

## Completed features
none

## In-progress feature
none

## Pending features
(populated after Step 4)

## Project-specific patterns
(populated as discovered)

## Known issues
none
```

Write the skeleton immediately so it exists before implementation begins. It will be updated with full project details at the end of Step 1 and again at the end of each feature.

---

## Step 1 — Read Project Context

Collect the information needed to make correct implementation decisions. Run in parallel:

```tool
mcp__plugin_ui5_ui5-mcp-server__get_project_info
  projectDir: "{{absolute_path}}"
```

```tool
mcp__plugin_ui5_ui5-mcp-server__get_guidelines
```

```bash
cat {{absolute_path}}/webapp/manifest.json
```

From the results extract and record:
- **UI5 version** — determines available APIs and TypeScript event type availability (≥ 1.115.0)
- **Framework** — SAPUI5 vs OpenUI5
- **Project type** — Fiori Elements vs Freestyle (check `Component.js` base class: `sap/ui/core/UIComponent` = Freestyle, `sap/suite/ui/generic/template/lib/AppComponent` = Fiori Elements)
- **App namespace** — needed for correct `sap.ui.define` module paths
- **OData version** — V2 or V4 (from `manifest.json` dataSources)
- **Routing config** — existing routes and targets

---

## Step 2 — Read Existing Files Relevant to the Feature

Before writing anything, read the files the feature will touch:

```bash
# Views and controllers
cat {{absolute_path}}/webapp/view/{{relevant}}.view.xml
cat {{absolute_path}}/webapp/controller/{{relevant}}.controller.js   # or .ts
cat {{absolute_path}}/webapp/i18n/i18n.properties
```

If the feature involves entity data, also read:

```bash
cat {{absolute_path}}/webapp/localService/mainService/metadata.xml
```

**OData V4 service**: If the feature requires understanding entity relationships, navigation properties, or querying live data — invoke the `odata-v4-reader` agent with the service URL and ask it to return the entity structure, key properties, and relevant associations. Use the returned information to inform the binding paths and `$expand` expressions in Step 5.

Only proceed once you understand the current state of these files.

---

## Step 3 — Research the Approach with the `fiori-frontend-dev` Agent

**Always invoke the `fiori-frontend-dev` agent before planning or writing any code.** The only exception is fixing a linter finding where the replacement is already stated verbatim in the finding detail — everything else requires agent consultation.

Invoke it with:
- The feature to implement (e.g. "add a detail page for the Employee entity")
- Project type (Fiori Freestyle or Elements), UI5 version, app namespace, OData version
- The controls, patterns, or APIs you are considering
- Any specific questions: correct form layout, OData type for a property, navigation pattern, etc.

The agent will consult the UI5 and Fiori MCP servers, validate API visibility and deprecation status, and return a recommended pattern. **Do not proceed to Step 4 until the agent has responded.** If the agent flags a deprecated API or recommends a different control or pattern, adopt it before planning.

For Fiori-specific patterns, also ask the agent to search the docs:

```
fiori-frontend-dev: "Search for the correct pattern for {{feature}} in a Fiori {{Freestyle|Elements}} app on UI5 {{version}}"
```

---

## Step 4 — Plan the Implementation

**Always present the plan to the user and wait for approval before writing any file — no exceptions, even for single-file changes.**

Write out:

- Files to create or modify (view, controller, i18n, routing in manifest)
- Controls to use and their binding paths (validated against metadata.xml)
- OData type to use for each bound property (prefer `sap.ui.model.odata.type.*`)
- New i18n keys needed
- Routing changes (new route + target if adding a new view)

Do not proceed to Step 5 until the user explicitly approves the plan.

---

## Step 5 — Implement

Apply the guidelines from Step 1 strictly while writing code.

### XML Views

- Declare all programmatic API (types, formatters) via `core:require` — never as `sap.x.y.Z` global strings in binding expressions
- Use `{i18n>key}` for all user-visible text — no hard-coded strings
- OData types for formatting:
  - Dates/Times: `sap/ui/model/odata/type/DateTime` or `DateTimeOffset`
  - Decimals: `sap/ui/model/odata/type/Decimal`
  - Integers: `sap/ui/model/odata/type/Int32`
- Forms: always `sap.ui.layout.form.Form` with `ColumnLayout` (2 cols M, 3 cols L, 4 cols XL) — never `SimpleForm`
- No inline `style=` attributes, no inline `<script>` tags

### Controllers (JavaScript)

```js
sap.ui.define([
    "sap/ui/core/mvc/Controller"
    // ... other explicit imports
], (Controller, ...) => {
    "use strict";
    return Controller.extend("{{namespace}}.controller.{{Name}}", {
        onInit() { },
        // handlers ...
    });
});
```

- All dependencies declared in `sap.ui.define` — no global `sap.x.y.Z` access
- No `jQuery.sap.require` or `sap.ui.requireSync`

### Controllers (TypeScript)

- Use ES6 `import` statements
- For event handlers, import the specific typed event (`Button$PressEvent`, etc.) when UI5 ≥ 1.115.0
- Extend `Controller` with `@UI5Define` decorator pattern if the project already uses it

### i18n

- Add every new key to `webapp/i18n/i18n.properties`
- Use comment annotations: `#XTIT:` (title), `#XBUT:` (button), `#XMSG:` (message), `#XCOL:` (column header), `#XPLA:` (placeholder), `#XFLD:` (field label)
- Never add locale-specific files (e.g. `i18n_en.properties`)

### Routing (new views only)

Add a new route and target to `manifest.json` under `sap.ui5/routing`:

```json
"routes": [
  { "name": "{{routeName}}", "pattern": "{{pattern}}", "target": ["{{targetName}}"] }
],
"targets": {
  "{{targetName}}": {
    "viewType": "XML",
    "transition": "slide",
    "path": "{{namespace}}.view",
    "viewName": "{{ViewName}}"
  }
}
```

---

## Step 6 — Validate

After writing every file, run the linter scoped to only the changed files:

```tool
mcp__plugin_ui5_ui5-mcp-server__run_ui5_linter
  projectDir:              "{{absolute_path}}"
  filePatterns:            ["webapp/view/{{changed}}.view.xml", "webapp/controller/{{changed}}.controller.js"]
  fix:                     false
  provideContextInformation: false
```

Then re-run manifest validation if `manifest.json` was changed:

```tool
mcp__plugin_ui5_ui5-mcp-server__run_manifest_validation
  manifestPath: "{{absolute_path}}/webapp/manifest.json"
```

Fix every finding before declaring the feature done. Re-run until clean.

---

## Step 7 — Fiori Elements Annotations

If the feature requires annotations (columns, filters, field groups, value helps, header info), **do not write them directly**. Invoke the `annotation-expert` agent with:

- The annotation file path: `webapp/annotations/annotation.xml`
- The metadata file path: `webapp/localService/mainService/metadata.xml`
- The entity name and the annotation(s) to add (e.g. `UI.LineItem`, `UI.FieldGroup`, `Common.ValueList`)
- Any constraints (which properties to include, which to exclude, sort order)

The `annotation-expert` validates every property path against `metadata.xml` before writing — never bypass it for annotation work, even for simple single-property changes.

**After the annotation-expert completes**, verify the round-trip:

1. Read the written annotation file and confirm every `PropertyPath` or binding referenced in the view (Step 5) has a matching annotation entry.
2. Confirm every annotated property exists in `metadata.xml`.
3. If the agent reported any failures (authorization error fetching metadata, property not found, malformed XML) — do not declare the feature done. Report the specific failure to the user and ask them to resolve it (e.g. update `metadata.xml` manually via the **Service Manager** in SAP Fiori Tools) before re-running the annotation step.

---

## Step 9 — Update Skill Memory

**This step is mandatory. Do not proceed to Step 10 or declare the feature complete until the memory file has been written.**

Update `{{absolute_path}}/.claude/skill-memory/fiori-feature-dev/memory.md` with the full project details and feature outcome:

```markdown
# fiori-feature-dev memory — {{project name}}

## Project
- Path: {{absolute_path}}
- Type: {{Fiori Elements | Freestyle}}
- Namespace: {{namespace}}
- UI5 version: {{version}}
- OData version: {{V2 | V4}}
- Main entity: {{mainEntity}}
- Metadata: {{metadata file path}}

## Completed features
{{List each completed feature with a one-line summary of what was implemented}}

## In-progress feature
{{Feature name and last completed sub-step, or "none"}}

## Pending features
{{Features not yet started}}

## Project-specific patterns
{{Any conventions, workarounds, or decisions discovered during implementation that should be applied consistently — e.g. custom formatter location, reused fragment names, known metadata quirks}}

## Known issues
{{Any unresolved findings, annotation failures, or linter issues left for the user to action}}
```

Write the full memory file now. Do not wait until all planned features are done — update after every completed feature and mark any in-progress feature with its last completed sub-step.

---

## Step 10 — Summary

Report to the user:

- Files created or modified
- Controls and bindings added
- i18n keys added
- Linter result: zero findings or outstanding items
- Any follow-up steps (e.g. annotation work to delegate to `annotation-expert`)
