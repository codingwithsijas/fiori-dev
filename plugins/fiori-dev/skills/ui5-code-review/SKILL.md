---
name: ui5-code-review
description: SAP Fiori / UI5 code review skill. Reviews staged changes or specified files against UI5 best practices, detects private/protected API usage, deprecated APIs, CSP violations, incorrect binding patterns, and form anti-patterns. Validates every finding against the UI5 MCP API reference. Presents a findings report and asks for confirmation before applying any fix.
---

# UI5 Code Review

Review SAP Fiori / UI5 code for correctness, best-practice compliance, and private/deprecated API usage. Every finding is validated against the live UI5 API reference before being reported. No file is modified without explicit user approval.

---

## Step 0 — Read Skill Memory

Check for an existing memory file for this skill in the project:

```bash
cat {{project_root}}/.claude/skill-memory/ui5-code-review/memory.md 2>/dev/null
```

If found, read it in full. Use it to:
- Understand which files or patterns have been reviewed before and what findings were accepted or skipped
- Avoid re-reporting findings the user has explicitly chosen not to fix
- Apply any project-specific review notes recorded in previous sessions

If not found, proceed — memory will be created at the end of this session.

---

## Step 1 — Determine Scope

If the user passed a file path or glob as an argument, use that. Otherwise default to:
1. Git-staged files: `git diff --cached --name-only`
2. If nothing staged: all files changed vs. the base branch: `git diff origin/HEAD --name-only`
3. If not a git repo: ask the user to specify files.

Filter to UI5-relevant extensions only: `.ts`, `.js`, `.xml`, `.json` (manifest only), `.html`.

---

## Step 2 — Collect Project Context

Run in parallel:

```tool
mcp__plugin_ui5_ui5-mcp-server__get_project_info
  projectDir: "{{project_root}}"
```

```tool
mcp__plugin_ui5_ui5-mcp-server__get_guidelines
```

From `get_project_info`, extract:
- **UI5 framework version** — determines which TypeScript event types are available (≥ 1.115.0)
- **Framework** — SAPUI5 vs OpenUI5 (affects available libraries)
- **Project type** — detected from manifest (Fiori Elements vs Freestyle)

---

## Step 3 — Run UI5 Linter

```tool
mcp__plugin_ui5_ui5-mcp-server__run_ui5_linter
  projectDir:              "{{project_root}}"
  filePatterns:            ["{{file1}}", "{{file2}}", ...]   // scoped to Step 1 files
  fix:                     false                              // never auto-fix before review
  provideContextInformation: true
```

Record all linter findings — they feed directly into the findings report in Step 6.

---

## Step 4 — Validate API Usage via `fiori-frontend-dev` Agent

Do not call `get_api_reference` directly for changed files. Invoke the `fiori-frontend-dev` agent to validate API usage and Fiori/UI5 pattern compliance — it will consult both the UI5 and Fiori MCP servers and return findings consistent with current standards:

```agent
fiori-frontend-dev
```

Pass it:
- The list of changed files and their content
- Project type (Fiori Elements or Freestyle), UI5 version, OData version
- Ask it to check: deprecated APIs, private/restricted access, incorrect binding patterns, form anti-patterns, CSP violations, and TypeScript event type usage

Merge the agent's findings with the linter output from Step 3. Do not run separate `get_api_reference` calls for symbols the agent has already evaluated.

For any Fiori Elements–specific pattern questions not covered by the agent, also search the docs:

```tool
mcp__fiori-mcp-server__search_docs
  query: "{{symbol or pattern in question}}"
```

---

## Step 4B — Review Annotation Files (Fiori Elements projects)

If any changed files include `annotation.xml` (under `webapp/annotations/` or `webapp/localService/`), invoke the `annotation-expert` agent to review them:

- Pass it the annotation file path and the corresponding `webapp/localService/metadata.xml`
- Ask it to check for:
  - **Vocabulary prefix errors** — wrong or missing namespace declaration (e.g. `UI.` without `xmlns:UI="..."`)
  - **Property path mismatches** — any `PropertyPath` or `Path` value that does not exist in `metadata.xml`
  - **Entity type mismatches** — annotations targeting an entity type not present in the metadata
  - **Structural errors** — malformed `Record`, `Collection`, or `Apply` expressions
  - **Annotation target errors** — `Target` attributes pointing to non-existent entity sets or properties

Record all annotation findings in the Step 6 report under a dedicated **ANNOTATION** severity bucket (treat property/entity mismatches as CRITICAL, structural/namespace issues as WARNING).

If no annotation files are in scope, skip this step.

---

## Step 5 — Review Checklist

Evaluate every changed file against all categories below. Only raise a finding when a violation is confirmed.

### 5A — Private & Restricted API
- Any method/property with visibility `private` or `restricted` accessed from outside its declared module
- Framework internals accessed via `_` prefix (e.g. `oControl._oPopup`, `oModel._mChangedEntities`)
- `sap.ui.getCore()` — deprecated since UI5 1.119; replace with `Component.getRouterFor()`, `this.getOwnerComponent()`, or direct model access
- Direct DOM manipulation via `$()` or `getDomRef()` when a UI5 API equivalent exists

### 5B — Deprecated API
- Any symbol flagged `deprecated` by the linter or API reference
- Report the deprecated symbol, the UI5 version it was deprecated in, and the official replacement with a code example

### 5C — Module Loading
- Global namespace access: `new sap.m.Button()` instead of declared dependency
- `jQuery.sap.require()` — synchronous, forbidden
- `sap.ui.requireSync()` — synchronous, forbidden
- Missing `sap.ui.define` / ES6 `import` wrapper for controllers, types, formatters

### 5D — Data Binding
- Custom formatter used where an OData type (`sap.ui.model.odata.type.*`) covers the use case
- Two-way binding implemented with a formatter (use a custom `SimpleType` instead)
- Hard-coded values in views that should be model-bound

### 5E — TypeScript Event Types (UI5 ≥ 1.115.0)
- Generic `Event` from `sap/ui/base/Event` used where a control-specific `<Control>$<Event>Event` type is available
- Missing import for the typed event (e.g. `Button$PressEvent`, `Table$RowSelectionChangeEvent`)

### 5F — Forms
- `sap.ui.layout.form.SimpleForm` used — must be replaced with `Form` + `ColumnLayout` unless the user explicitly requested SimpleForm
- `Form` without a `ColumnLayout` — missing responsive layout declaration

### 5G — CSP
- Inline `<script>` tags
- Inline `<style>` tags
- Inline `style="..."` attributes on elements
- `eval()` or `new Function(...)` in JS/TS files

### 5H — i18n
- Hard-coded user-visible strings in views or controllers (not bound to `{i18n>...}`)
- Direct edits to localized files (`i18n_de.properties`, `i18n_fr.properties`, etc.)

### 5I — Manifest
- `minUI5Version` missing or below the version actually used
- Missing `sap.app.i18n` declaration
- OData V4 model declared without `"synchronizationMode": "None"` and `"operationMode": "Server"`

### 5J — Component Initialization
- Manual `new Component(...)` calls in `index.html` — use `ComponentSupport` instead
- `data-sap-ui-async="false"` — must be `true`

---

## Step 6 — Build Findings Report

Group findings by severity. For each finding:

```
### [CRITICAL|WARNING|INFO] <Short title>

**File**: path/to/file.ts  **Line**: N
**Rule**: <category from 5A–5J>
**Visibility** (if API issue): private | restricted | deprecated

**What was found**:
<exact code snippet from the file>

**Why it is a problem**:
<one sentence — cite the UI5 guideline or API reference>

**Recommended fix**:
<corrected code snippet>

**Reference**: <UI5 doc page title or API symbol returned by get_api_reference>
```

Severity mapping:
- **CRITICAL**: private API, `eval`, inline scripts, synchronous loading, annotation property/entity path mismatches
- **WARNING**: deprecated API, SimpleForm, missing typed event, CSP violation, wrong binding, annotation structural/namespace issues
- **INFO**: i18n, manifest tweaks, style suggestions

End the report with a summary table:

| Severity | Count |
|---|---|
| CRITICAL | N |
| WARNING | N |
| INFO | N |
| ANNOTATION | N |
| **Total** | **N** |

If zero findings: state "No issues found — code meets UI5 best practices." and stop.

---

## Step 7 — Ask Before Fixing

After presenting the report, ask:

> **Would you like me to apply the fixes above?**
> - **All** — apply every finding
> - **Critical only** — apply CRITICAL findings only
> - **Select** — list the finding numbers you want applied (e.g. "1, 3, 5")
> - **None** — review only, no changes

**Do not edit any file until the user responds.**

---

## Step 8 — Apply Approved Fixes

For each approved finding:
1. Edit the file using the minimum change needed — do not refactor surrounding code.
2. Re-run the linter on the patched file to confirm the finding is resolved:
   ```tool
   mcp__plugin_ui5_ui5-mcp-server__run_ui5_linter
     projectDir:   "{{project_root}}"
     filePatterns: ["{{patched_file}}"]
     fix:          false
   ```
3. If a new finding is introduced by the fix, report it immediately before continuing.
4. After all fixes are applied, run a final linter pass over all patched files and confirm zero findings remain.

---

## Step 9 — Update Skill Memory

Create or update `{{project_root}}/.claude/skill-memory/ui5-code-review/memory.md`:

```markdown
# ui5-code-review memory — {{project name}}

## Project
- Path: {{project_root}}
- Type: {{Fiori Elements | Freestyle}}
- UI5 version: {{version}}
- Last reviewed: {{date}}

## Skipped findings
{{List any findings the user explicitly chose not to fix, so they are not re-reported}}
- {{file}} — {{finding title}} — reason: {{user's reason or "user declined"}}

## Recurring patterns
{{Anti-patterns seen repeatedly across multiple review sessions — flag these as high priority in future reviews}}

## Resolved issues
{{Summary of findings fixed per session}}
```

Write the memory file even on the first run. Update it after every completed review session.

---

## Step 10 — Final Summary

Report:
- Files reviewed
- Findings found / fixed / skipped
- Any findings the user chose not to fix (so they are not forgotten)
- Reminder: **restart Claude Code** if MCP servers were reconfigured during the session
