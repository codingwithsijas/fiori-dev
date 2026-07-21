---
name: develop-fiori-feature
description: Implement any UI feature in a Fiori Freestyle or Fiori Elements project post-setup — views, controllers, OData bindings, routing, i18n, fragments, and annotations. Validates every implementation decision against the UI5 and Fiori MCP servers before writing code. Use for all development work after the project scaffold exists.
---

# Develop Fiori Feature

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

## Protocol Guarantee

These two steps are **non-negotiable** regardless of how the skill was invoked (direct slash command, inline request, or sub-task):

- **Step 6** (linter + manifest validation) — must run after **every** file write, including single-line edits
- **Step 9** (memory update) — must run after **every** completed feature

The assistant must not declare a feature done until both have run and passed.

---

## Step 0 — Application Analysis and Skill Memory

### 0a — Read or Regenerate application.md

The file `{{absolute_path}}/.claude/skill-memory/develop-fiori-feature/application.md` is a cached analysis of the project structure. It is the primary source of truth for project shape and must be loaded before anything else.

**Check whether the file exists and is still fresh:**

```bash
# 1. Does the file exist?
ls {{absolute_path}}/.claude/skill-memory/develop-fiori-feature/application.md 2>/dev/null

# 2. When was it last written? (file modification timestamp)
stat -f "%Sm" -t "%Y-%m-%d %H:%M:%S" \
  {{absolute_path}}/.claude/skill-memory/develop-fiori-feature/application.md 2>/dev/null

# 3. Have any webapp/** or backend files changed since then?
git -C {{absolute_path}} log --oneline --after="$(stat -f "%Sm" -t "%Y-%m-%dT%H:%M:%S" \
  {{absolute_path}}/.claude/skill-memory/develop-fiori-feature/application.md 2>/dev/null || echo '1970-01-01T00:00:00')" \
  -- webapp/ pom.xml package.json 2>/dev/null | head -5
```

**Regenerate if any of these are true:**
- The file does not exist
- The `git log` command above returns one or more commits (source files changed since last analysis)

**If the file is fresh (exists + no new commits), read it and skip to Step 0b.**

---

**To regenerate, run the following in parallel:**

```bash
# Project identity
cat {{absolute_path}}/webapp/manifest.json

# App structure
find {{absolute_path}}/webapp/view       -name "*.view.xml"    | sort
find {{absolute_path}}/webapp/controller -name "*.controller.*" | sort
find {{absolute_path}}/webapp/fragment   -name "*.fragment.xml" 2>/dev/null | sort
find {{absolute_path}}/webapp/util       -name "*.*"            2>/dev/null | sort
find {{absolute_path}}/webapp/model      -name "*.*"            2>/dev/null | sort
find {{absolute_path}}/webapp/formatter  -name "*.*"            2>/dev/null | sort

# Component base class → project type
grep -m1 "UIComponent\|AppComponent\|sap/suite/ui/generic" \
  {{absolute_path}}/webapp/Component.js 2>/dev/null \
  || grep -m1 "UIComponent\|AppComponent\|sap/suite/ui/generic" \
     {{absolute_path}}/webapp/Component.ts 2>/dev/null

# OData version and service paths
grep -A5 '"dataSources"' {{absolute_path}}/webapp/manifest.json | head -20
```

```bash
# Backend dependencies — detect project type first
if [ -f {{absolute_path}}/pom.xml ]; then
  # RAP / CAP Java project
  grep -E "<artifactId>|<groupId>|<version>" {{absolute_path}}/pom.xml \
    | grep -v "^--$" | head -60
else
  # CAP Node.js / pure UI5 project
  cat {{absolute_path}}/package.json
fi
```

**From the gathered data, write `application.md`** using this exact structure:

```markdown
# Application Analysis — {{project name}}
_Last analysed: {{ISO date}}_

## Identity
- **Artifact ID / App ID**: {{sap.app.id from manifest}}
- **Application name**: {{sap.app.title from manifest}}
- **Namespace**: {{sap.ui5 namespace}}
- **UI5 version**: {{sap.ui5.minUI5Version or framework version}}
- **Framework**: {{SAPUI5 | OpenUI5}}

## Template
- **Type**: {{Fiori Freestyle | Fiori Elements V2 | Fiori Elements V4}}
  - Freestyle: Component base = `sap/ui/core/UIComponent`
  - Elements V2: Component base = `sap/suite/ui/generic/template/lib/AppComponent` OR OData V2 + `sap.ui.generic.app` in manifest
  - Elements V4: OData V4 + `sap/fe/core/AppComponent` OR `sap.fe.templates` in manifest

## OData Service
- **Version**: {{V2 | V4}}
- **Service path**: {{dataSource uri from manifest}}
- **Metadata path**: {{localUri from manifest}}

## Structure

### Views
{{list each .view.xml file — filename only, one per line}}

### Controllers
{{list each .controller.js/.ts file — filename only, one per line}}

### Fragments
{{list each .fragment.xml file, or "none"}}

### Utilities / Formatters / Models
{{list files under util/, formatter/, model/ — filename only, or "none"}}

## Libraries

### UI5 Libraries (from manifest sap.ui5.dependencies.libs)
{{list each library name, one per line}}

### External / Backend Dependencies
{{If pom.xml found — list groupId:artifactId for each <dependency> block}}
{{If package.json found — list all keys from "dependencies" and "devDependencies"}}
```

Write the file to `{{absolute_path}}/.claude/skill-memory/develop-fiori-feature/application.md`.

---

### 0b — Read Skill Memory

Check for `{{absolute_path}}/.claude/skill-memory/develop-fiori-feature/memory.md`:

```bash
cat {{absolute_path}}/.claude/skill-memory/develop-fiori-feature/memory.md 2>/dev/null
```

If found, read it in full — use it to resume in-progress features and apply known project patterns.

If not found, **create the skeleton now**:

```markdown
# develop-fiori-feature memory — (project name TBD)

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

Write the skeleton immediately. It will be filled in after Step 1 and updated after every feature.

---

## Step 1 — Load Project Context from application.md

`application.md` (written or refreshed in Step 0) is the authoritative project context. Extract and hold in working memory:

- **UI5 version** — for API and TypeScript event type decisions (typed events ≥ 1.115.0)
- **Framework** — SAPUI5 vs OpenUI5
- **Project type** — Fiori Elements V2, Elements V4, or Freestyle
- **App namespace** — required for all `sap.ui.define` module paths
- **OData version** — V2 or V4
- **Routing config** — read from `manifest.json` if not already captured in `application.md`
- **Existing views, controllers, fragments** — use the lists from `application.md` to know what already exists before planning

Also load the UI5 guidelines (run in parallel with reading `application.md`):

```tool
mcp__plugin_ui5_ui5-mcp-server__get_guidelines
```

Do not re-scan `manifest.json` or the file system for anything already recorded in `application.md`.

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

## Step 6 — Validate *(mandatory after every file write — no exceptions)*

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

## Step 9 — Update Skill Memory *(mandatory — always runs, no exceptions)*

**This step is mandatory regardless of invocation method. Do not proceed to Step 10 or declare the feature complete until the memory file has been written.**

Update `{{absolute_path}}/.claude/skill-memory/develop-fiori-feature/memory.md` with the full project details and feature outcome:

```markdown
# develop-fiori-feature memory — {{project name}}

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
