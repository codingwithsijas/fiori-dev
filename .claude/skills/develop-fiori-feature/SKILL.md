---
name: develop-fiori-feature
description: Implement any UI feature in a Fiori Freestyle or Fiori Elements project post-setup — views, controllers, OData bindings, routing, i18n, fragments, annotations, and OPA5 tests. Validates every implementation decision against the UI5 and Fiori MCP servers before writing code. Use for all development work after the project scaffold exists.
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
- Writing OPA5 integration tests (uses the standard Fiori Elements V4 OPA test library via UI5 MCP)
- Any UI development work that follows project scaffolding

---

## OPA5 Reference (Fiori Elements V4 — `sap/fe/test`)

> Applies only to Fiori Elements V4 projects using `sap/fe/test` page objects. For Freestyle OPA5, consult the UI5 MCP for the correct patterns.

### Project setup

Each app page needs a page object registered in `opaTests.qunit.js`:

```js
sap.ui.require([
    'sap/fe/test/JourneyRunner',
    'my/namespace/test/integration/pages/MainObjectPage'
], function (JourneyRunner, MainObjectPage) {
    var runner = new JourneyRunner({
        launchUrl: sap.ui.require.toUrl('my/namespace/test') + '/flpSandbox.html#App-action'
    });
    runner.run({ pages: { onTheMainObjectPage: MainObjectPage } }, MyJourney.run);
});
```

Page object definition — `componentId` must match the target `"id"` in `manifest.json` under `sap.ui5.routing.targets`:

```js
sap.ui.define(['sap/fe/test/ObjectPage'], function (ObjectPage) {
    'use strict';
    return new ObjectPage(
        { appId: 'my.namespace', componentId: 'MainObjectPage', contextPath: '/EntitySet' },
        { actions: {}, assertions: {} }
    );
});
```

### Table interactions

Identify a table by its navigation property binding. When a table has a qualifier (e.g. `UI.LineItem#listTeams`), **always pass `qualifier`** — without it `onTable` cannot uniquely resolve the table and silently matches the wrong one:

```js
When.onTheObjectPage.onTable({ property: "_Items" }).iPressRow(0);
When.onTheObjectPage.onTable({ property: "_Items" }).iChangeRow(0, { "Description": "New" });
Then.onTheObjectPage.onTable({ property: "_Items" }).iCheckCells(0, { "Name": "Expected" });
Then.onTheObjectPage.onTable({ property: "_Items" }).iCheckRows(3);
When.onTheObjectPage.onTable({ property: "_Items", qualifier: "listTeams" }).iPressCell(0, "Link");
```

### Resetting mock data between tests

| Method | What it resets |
| --- | --- |
| `iResetTestData()` | `localStorage`, `sessionStorage`, IndexedDB only — does **not** reset mockserver entity data |
| `iResetMockData(oAppInfo)` | Sends `POST $metadata/reload` to the mockserver — clears all in-memory entity sets and reloads from JSON/JS files |

Use `iResetMockData` whenever a previous test has mutated data (create, update, delete).

```js
// Before app starts — use ServiceUri (AppId not yet available)
Given.iResetMockData({
    ServiceUri: '/sap/opu/odata4/sap/MY_SRV/srvd/sap/my_srv/0001'
}).and.iStartMyApp("", { Entity: "KEY001" });

// After app starts — use AppId
Given.iResetMockData({ AppId: 'my.namespace' });
```

> **Critical**: `ServiceUri` must **not** end with `/`. The framework appends `$metadata/reload` directly — a trailing slash produces a double separator, the mockserver route never matches, and `iResetMockData` silently resolves as successful (`.finally()` sets `bSuccess = true` regardless of HTTP status). The OPA chain continues with stale data and the failure is invisible.

### Teardown

Always end every test module with a teardown test. Without it the next `iStartMyApp` reuses the existing iframe and carries over mock server state:

```js
opaTest("Teardown", function (Given, When, Then) {
    Given.iTearDownMyApp();
});
```

### Visibility assertions on table actions

`Then.onThePage.onTable({ property: "..." }).iCheckAction("Label", { visible: false })` passes when either:
- The button's `visible` property is `false`, OR
- The button was not rendered at all (because its `visible` binding resolved to `false` before rendering)

This is the correct assertion for expression-binding-based visibility — no DOM element may exist at all. `{ visible: true }` checks the button is rendered and visible.

### Starting directly on the Object Page

Use `iStartMyApp("", { Entity: "PROJID" })` to launch directly on the Object Page for a specific entity. This avoids the need for List Report filter setup and is faster/more robust when List Report navigation is not being tested:

```js
Given.iResetTestData().and.iStartMyApp("", { Project: "PROJID" });
Then.onTheDetailPage.iSeeThisPage();
When.onTheDetailPage.iEnterFullScreenMode();
When.onTheDetailPage.iGoToSection("Billing");
Then.onTheDetailPage.onTable({ property: "_EntProjProfnlSrvcBillgElmnt" }).iCheckAction("Manage", { visible: false });
Given.iTearDownMyApp();
```

### `requiresSelection` and action visibility tests

If a table action button in `controlConfiguration` does not have `"requiresSelection": true`, the button renders regardless of whether any rows are present. Tests asserting the button's `visible` state do not need child entity mock data for the test project.

**Rule:** When testing a `visible` expression binding on a table action, check whether `requiresSelection` is set. If not set (or `false`), no rows are required and the test is simpler.

### Closing an FCL column

`iCloseThisColumn` is not automatically mixed into `ObjectPage`. It lives in `sap/fe/test/FCLView`. Pass `FCLView` as an additional page definition — `ObjectPage` accepts `..._aAdditionalPageDefinitions` and merges them all in:

```js
sap.ui.define(['sap/fe/test/ObjectPage', 'sap/fe/test/FCLView'], function(ObjectPage, FCLView) {
    'use strict';

    return new ObjectPage(
        { appId: 'my.namespace', componentId: 'MySubObjectPage', contextPath: '/EntitySet/_Nav' },
        { actions: {}, assertions: {} },
        FCLView   // mixes in iCloseThisColumn, iEnterFullScreenMode, iExitFullScreenMode
    );
});
```

```js
// Usage in journey
When.onTheSubObjectPage.iCloseThisColumn();
```

`FCLView` also provides `iEnterFullScreenMode`, `iExitFullScreenMode`, and assertions `iCheckCloseColumnButton`, `iCheckFullScreenButton`.

---

## Fiori Elements V4 Patterns (built-in knowledge)

### `additionalProperties` for programmatically-accessed OData properties

When `autoExpandSelect: true` is set in the OData model settings, the framework only adds properties to `$select` that are referenced in view XML bindings. Properties accessed purely in JavaScript — e.g. inside a custom `enabled` formatter or a `press` handler via `requestObject`/`requestProperty` — are never added automatically.

**Symptom:** `requestObject("SomeProperty")` logs `"Failed to drill-down into .../SomeProperty, invalid segment: SomeProperty"`. The promise resolves with `undefined`, causing silent incorrect behaviour (e.g. `undefined !== "00"` is always `true`, so a button is always enabled regardless of actual value).

**Fix:** Add the property to `additionalProperties` in `manifest.json` for the relevant page target:

```json
"settings": {
  "additionalProperties": ["Customer", "ProcessingStatus"]
}
```

**Rule:** Any time a custom enabled formatter or press handler calls `requestObject`/`requestProperty`/`getProperty` on a field that is not bound in the view XML, add that field to `additionalProperties`.

### `enhanceI18n` is only for overriding Fiori Elements framework texts

`enhanceI18n` (configured per page target in `manifest.json`) is exclusively for overriding standard FE framework text keys (e.g. `C_TRANSACTION_HELPER_CONFIRM_DELETE_WITH_OBJECTTITLE_SINGULAR`).

**Rule:** App-specific texts — custom action labels, error messages, section titles — always go in the application's own `webapp/i18n/i18n.properties`. Never add app-specific keys to the `enhanceI18n` file.

### `this.showMessages()` for errors in custom action handlers

In Fiori Elements V4, custom action handlers have `this` bound to the ExtensionAPI. To surface an error, use the FE message framework rather than `sap.m.MessageBox`:

```js
sap.ui.define([
    "sap/ui/core/message/Message",
    "sap/ui/core/message/MessageType"
], function(Message, MessageType) {
    return {
        myAction: function(oContext) {
            var sMsg = this.getModel("i18n").getResourceBundle().getText("myErrorKey");
            this.showMessages([new Message({ message: sMsg, type: MessageType.Error })]);
        }
    };
});
```

`this.showMessages()` displays the error in the Object Page message strip — consistent with how backend errors appear. It is available on `sap.fe.templates.ObjectPage.ExtensionAPI`.

**Rule:** Use `this.showMessages()` whenever a custom press handler needs to show an error without navigating. Avoid `sap.m.MessageBox`.

### Guard order in custom action handlers: most restrictive checks first

When a custom action handler has multiple early-return guards, order them from most restrictive to least. A stage/status check that blocks the action entirely must come before a check that only changes the mode.

**Rule:** Sequence guards as: absolute blocks first (stage prevents the action entirely) → conditional blocks (mode changes) → mode derivation. Wrong order causes a project in multiple states to hit the wrong error message.

---

## Fiori Elements V4 Expression Bindings (built-in knowledge)

### `%{property}` vs `${property}` in manifest.json expression bindings

In Fiori Elements V4 manifest expression bindings (e.g. `visible` on a `controlConfiguration` action), the binding context resolution differs by prefix:

- **`${property}`** — resolves against the **current control's binding context** (e.g. a table row's entity)
- **`%{property}`** — resolves against the **page's root entity binding context** (the Object Page's main entity)

**Use case:** When a table action needs to read a property from the root entity (e.g. `ProcessingStatus` on a project), use `%{ProcessingStatus}`, not `${ProcessingStatus}`. The sub-entity rows do not have `ProcessingStatus` in their context.

```json
"visible": "{= ${mCustomModel>/isNewSalesOrderAppEnabled} && %{ProcessingStatus} !== '00' }"
```

Here `${mCustomModel>/...}` reads from a named JSON model (works in both contexts), and `%{ProcessingStatus}` reads from the page root entity.

### String-based formatter paths do not work in manifest bindings

`"formatter": "namespace.Formatter.method"` in manifest binding expressions requires the function to be available on the global namespace. AMD modules loaded via `sap.ui.define` are not on the global namespace.

**Symptom:** The formatter is never called; the binding evaluates as if the function doesn't exist.

**Fix:** Use an expression binding instead of a formatter path for dynamic visibility based on model properties.

### Handler function paths in `visible` are evaluated once

A `visible` property set to a handler function path is resolved once when the control renders. It does not react to subsequent model changes (e.g. a stage transition). This makes it unsuitable for dynamic visibility that must update after navigation or data changes.

**Fix:** Use expression bindings (`{= ... }`) for reactive visibility. Expression bindings re-evaluate whenever the bound properties change.

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
