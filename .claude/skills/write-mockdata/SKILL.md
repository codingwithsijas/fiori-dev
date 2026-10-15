---
name: write-mockdata
description: Write or update SAP FE Mockserver mockdata files — JSON datasets, JS hook files (addEntry, onAfterRead, updateEntry, removeEntry, executeAction, onDraftPrepare), getReferentialConstraints, EntityContainer unbound actions, and cross-service communication. Use after project scaffolding to set up test data for OPA tests or manual exploration.
---

# Write Mockdata

Write and maintain `sap-fe-mockserver` (`@sap-ux/ui5-middleware-fe-mockserver`) mockdata for an SAP Fiori / UI5 project. Produces JSON datasets and JS hook files validated against `metadata.xml`.

> **Scope**: This skill applies only to **UI5 Tooling projects** that use `sap-fe-mockserver` as a middleware (configured in `ui5-mock.yaml` or `ui5.yaml`). It does not apply to CAP-native mocking (`cds mock`) or other mock strategies.

## When to Use

- Creating initial JSON datasets for entity sets
- Writing JS hook files (`addEntry`, `onAfterRead`, `updateEntry`, `removeEntry`, `executeAction`, `onDraftPrepare`)
- Adding `getReferentialConstraints` to supply missing navigation join keys
- Implementing unbound actions in `EntityContainer.js`
- Extending mockdata to cover OPA test scenarios
- Configuring cross-service mockdata communication

---

## Mockserver Reference (built-in knowledge)

### File naming rules

| File | Named after | Purpose |
| --- | --- | --- |
| `EntitySetName.json` | EntitySet | Static initial dataset |
| `EntitySetName.js` | EntitySet | Hook functions (`addEntry`, `onAfterRead`, etc.) |
| `EntityTypeName.js` | EntityType | `getReferentialConstraints` only |
| `EntitySetName-100.json` | EntitySet + tenant | Tenant-specific data (sap-client=100) |
| `EntityContainer.js` | Fixed name | Unbound actions / FunctionImports |

### JSON data rules

- All key properties and `IsActiveEntity` must be present on every record.
- Every property referenced in any `$select` must exist on the record.
- Do **not** embed navigation expansions inline in JSON (the mockserver resolves `$expand` via referential constraints). The only exception is attaching a child object to `mockEntry` inside `addEntry` to shape the create response.
- `__FieldControl` values: `1` = read-only, `3` = optional, `7` = mandatory.
- **UUID generation**: The variant segment of a UUID must be hex-only (`[0-9a-f]`). Use `c057` (or any valid 4-hex-char value) — never a word like `cust`. Example: `42010aee-c646-1edd-c057-000000000001`.
- **`replace_all` guard**: When using `replace_all` to add a field, first check whether the field already exists on the record. Either (a) skip the initial targeted single-record edit and rely solely on `replace_all`, or (b) guard the `replace_all` pattern so it only matches records that do **not** already have the field — this prevents duplicate JSON keys.

### Hook rules

- Export only the hooks you need; unimplemented hooks fall back to mockserver defaults.
- Always end overridden hooks with `this.base.<hook>.apply(this, arguments)` — except `onAfterRead` and `executeAction`, which return a value.
- In `onAfterRead`, always filter the `data` parameter. Never `require()` the JSON file and filter it directly — that bypasses key predicates, tenant isolation, and mockserver resolution. The only valid exception is a value help that must return its full static list.
- `throwError(message, statusCode?, messageData?, isSAPMessage?, headers?)` — call `this.throwError(...)` inside any hook to return an error response immediately.

### `odataRequest.queryPath` structure

Each URL path segment becomes a `queryPath` entry with shape `{ path: string, keys: Record<string, any> }`:

| Index | Corresponds to |
| --- | --- |
| `[0]` | Root entity set (e.g. `Project`) |
| `[1]` | First navigation (e.g. `_Element`) |
| `[2]` | Second navigation (e.g. `_Team`) |

`keys` is populated when the URL contains a key predicate.

### `base` API reference

Accessible as `this.base.xxx` inside any hook:

| Method | Signature | Description |
| --- | --- | --- |
| `addEntry` | `(mockEntry, odataRequest?) => void` | Add a record |
| `updateEntry` | `(keyValues, newData, odataRequest?) => void` | Update a record by key |
| `removeEntry` | `(keyValues, odataRequest?) => void` | Delete a record by key |
| `getEntityInterface` | `(entityName, serviceAlias?) => Promise<interface>` | Get interface for an entity set |

Entity interface methods (returned by `getEntityInterface`):

| Method | Description |
| --- | --- |
| `addEntry(mockEntry)` | Add a new record |
| `updateEntry(keyValues, patchData)` | Update — merges, preserves existing fields |
| `removeEntry(keyValues)` | Delete a record |
| `getAllEntries()` | Return all records |
| `fetchEntries(keyValues)` | Return records matching key values |
| `hasEntry(keyValues)` | Check if a record exists |
| `getEmptyObject()` | Return an empty template object for the entity |
| `generateKey(property)` | Generate a new GUID or key value |

### `getReferentialConstraints` rules

- Place this function in the **EntityType** JS file (not the EntitySet file).
- Always include a `default` case returning `navigationDetail.referentialConstraint`.
- `sourceProperty` is on the source (parent) entity; `targetProperty` is on the target (child/related) entity.
- Multiple constraints in one array are ANDed together.

---

## Step 0 — Read Skill Memory

```bash
cat {{absolute_path}}/.claude/skill-memory/write-mockdata/memory.md 2>/dev/null
```

If not found, create the skeleton immediately:

```markdown
# write-mockdata memory — (project name TBD)

## Project
(populated after Step 1)

## Mockdata location
(populated after Step 1 — use repo-relative path)

## Completed mockdata files
none

## In-progress
none

## Project-specific patterns
(populated as discovered)

## Known issues
none
```

Write the skeleton to `{{absolute_path}}/.claude/skill-memory/write-mockdata/memory.md`.

---

## Step 1 — Locate Mockdata Configuration

Read the project's mockserver config to determine where mockdata files live:

```bash
# Check for ui5-mock.yaml (preferred)
cat {{absolute_path}}/ui5-mock.yaml 2>/dev/null

# Fall back to ui5.yaml if no mock yaml
cat {{absolute_path}}/ui5.yaml 2>/dev/null

# Confirm the mockdata directory exists
ls {{absolute_path}}/webapp/localService/ 2>/dev/null
```

From the config, extract:
- **`mockdataRootPath`** (Option A `config.json`) or **`mockdataPath`** (Option B inline `services`) — this is where JSON and JS files go.
- **`metadataXmlPath`** / **`metadataPath`** — use to validate property names.
- **Service aliases** (Option B) — required for cross-service `getEntityInterface` calls.

If neither config exists, ask the user to point you to the mockdata folder before proceeding.

---

## Step 2 — Read metadata.xml

Read the metadata file identified in Step 1:

```bash
cat {{absolute_path}}/{{metadataXmlPath}}
```

From metadata, extract and hold in working memory:
- **EntitySet names** → determines JSON / hook file names.
- **EntityType names** → determines `getReferentialConstraints` file names.
- **Key properties** for each EntityType → every JSON record must include all keys.
- **Navigation properties** and any `<ReferentialConstraint>` elements → tells you which navigations already have join keys defined and which need `getReferentialConstraints`.
- **Property types** (e.g. `Edm.Guid`, `Edm.String`, `Edm.Boolean`) → used when generating keys or default values.

---

## Step 3 — Inspect Existing Mockdata Files

```bash
ls {{absolute_path}}/{{mockdataRootPath}}/ 2>/dev/null
```

Read any existing JSON or JS file that the feature will touch or extend. Understand the current shape before adding or changing anything.

---

## Step 4 — Plan

**Always present the plan to the user and wait for approval before writing any file.**

Write out:

- JSON files to create or update (one per entity set, with all required key properties listed)
- JS hook files to create or update and which hooks each will export
- `getReferentialConstraints` files needed (where metadata is missing `<ReferentialConstraint>`)
- Cross-service dependencies (if any hook reads from another service alias)
- OPA test scenarios the mockdata will support (if provided by the user)

Do not proceed to Step 5 until the user explicitly approves the plan.

---

## Step 5 — Implement

### JSON datasets

Write `{{EntitySetName}}.json` files under `{{mockdataRootPath}}`:

```json
[
    {
        "MyEntityUUID": "aaaabbbb-0000-1111-2222-ccccddddeeee",
        "IsActiveEntity": true,
        "HasActiveEntity": false,
        "__EntityControl": { "Updatable": true, "Deletable": true },
        "__FieldControl": { "MyProperty": 3 }
    }
]
```

Checklist before writing each JSON file:
- [ ] All key properties present on every record
- [ ] `IsActiveEntity` present if the entity supports draft
- [ ] No inline navigation expansions (no `"_Child": { ... }` unless inside `addEntry mockEntry`)
- [ ] Properties match names in `metadata.xml` exactly (case-sensitive)

### EntitySet JS hooks

Write `{{EntitySetName}}.js` under `{{mockdataRootPath}}`:

```js
"use strict";

// addEntry — set defaults, populate computed fields, create child entities
async function addEntry(mockEntry, odataRequest) {
    if (mockEntry.HasActiveEntity === false && mockEntry.IsActiveEntity === false) {
        mockEntry.__EntityControl = { Deletable: true, Updatable: true };
        // set defaults here
    }
    this.base.addEntry.apply(this, arguments);
}

// onAfterRead — discriminator filtering only; always filter the data parameter
async function onAfterRead(data, odataRequest) {
    const segment = odataRequest.queryPath[2];
    if (!segment) return data;
    switch (segment.path) {
        case "_TypeA": return data.filter(item => item.Type === "A");
        default:       return data;
    }
}

// updateEntry — apply side-effects
function updateEntry(keyValues, newData, odataRequest) {
    this.base.updateEntry.apply(this, arguments);
}

// removeEntry — clean up related data
async function removeEntry(mockEntry) {
    this.base.removeEntry.apply(this, arguments);
}

// executeAction — handle bound actions
function executeAction(actionDefinition, actionData, keys, odataRequest) {
    switch (actionDefinition.name) {
        case 'myAction':
            return { result: 'ok' };
        default:
            this.throwError('Not implemented', 501, {
                error: { message: `Action "${actionDefinition.name}" not mocked` }
            });
    }
}

// onDraftPrepare — inject SAP__Messages for draft validation warnings/errors
async function onDraftPrepare(actionDefinition, responseData, keys, odataRequest) {
    responseData['SAP__Messages'] = [{
        code: '001',
        message: 'Validation warning',
        numericSeverity: 3,
        target: '/MyEntity(ID=1,IsActiveEntity=false)/MyField',
        transition: true
    }];
}

module.exports = { addEntry, onAfterRead, updateEntry, removeEntry, executeAction };
```

Export only the hooks actually needed — remove unused ones from both the function definitions and `module.exports`.

### Shared Entity Set Pattern (Discriminator)

When multiple navigation properties on a parent all resolve to the same child entity set, use all three mechanisms together:

1. **`getReferentialConstraints`** on the parent EntityType — provides the join key for every navigation.
2. **`onAfterRead`** on the child EntitySet JS file — filters by the discriminator property.
3. JSON data contains a discriminator field that matches the switch cases.

```
ParentEntity (ParentUUID)
  ├─ _TypeA ─┐
  ├─ _TypeB  ├── all resolve to SharedEntity via getReferentialConstraints
  └─ _TypeC ─┘
                  ↓
       onAfterRead filters by SharedEntity.DemandType
```

Parent EntityType constraints file:

```js
// ParentEntityType.js
function getReferentialConstraints(navigationDetail) {
    switch (navigationDetail.name) {
        case '_TypeA':
        case '_TypeB':
        case '_TypeC':
            return [{ sourceProperty: "ParentUUID", targetProperty: "ReferencedObjectUUID" }];
        default:
            return navigationDetail.referentialConstraint;
    }
}
module.exports = { getReferentialConstraints };
```

Child EntitySet discriminator filter:

```js
// SharedEntity.js
async function onAfterRead(data, odataRequest) {
    const segment = odataRequest.queryPath[2];
    switch (segment.path) {
        case "_TypeA": return data.filter(item => item.DemandType === "A");
        case "_TypeB": return data.filter(item => item.DemandType === "B");
        case "_TypeC": return data.filter(item => item.DemandType === "C");
        default:       return data;
    }
}
module.exports = { onAfterRead };
```

### getReferentialConstraints

Write `{{EntityTypeName}}.js` under `{{mockdataRootPath}}` when a navigation property is missing `<ReferentialConstraint>` in metadata:

```js
"use strict";

function getReferentialConstraints(navigationDetail) {
    switch (navigationDetail.name) {
        case '_ChildA':
        case '_ChildB':
            return [{ sourceProperty: "ParentUUID", targetProperty: "ReferencedUUID" }];
        default:
            return navigationDetail.referentialConstraint;
    }
}

module.exports = { getReferentialConstraints };
```

### EntityContainer.js — unbound actions

```js
"use strict";

module.exports = {
    executeAction(actionDefinition, actionData, keys) {
        switch (actionDefinition.name) {
            case 'MyUnboundAction':
                return { status: 'done' };
            default:
                this.throwError('Not implemented', 501);
        }
    }
};
```

### Cross-service calls

When a hook needs data from another service (requires aliases in `ui5-mock.yaml`):

```js
const otherEntity = await this.base.getEntityInterface('OtherEntitySet', 'otherService');
if (otherEntity) {
    const entries = await otherEntity.getAllEntries();
    // use entries
}
```

---

## Step 6 — Verify

After writing all files, verify structural correctness:

```bash
# Confirm all expected files are present
ls {{absolute_path}}/{{mockdataRootPath}}/

# Validate JSON syntax for every written .json file
node -e "JSON.parse(require('fs').readFileSync('{{absolute_path}}/{{mockdataRootPath}}/{{EntitySet}}.json','utf8'))" \
  && echo "JSON valid" || echo "JSON INVALID"

# Check JS hook files for syntax errors
node --check {{absolute_path}}/{{mockdataRootPath}}/{{EntitySet}}.js \
  && echo "JS valid" || echo "JS INVALID"
```

Fix any syntax errors before declaring work done.

Cross-check JSON records against metadata:
- Every key property extracted in Step 2 appears in every record.
- No property name is misspelled (compare against metadata property names exactly).

---

## Step 7 — Update Skill Memory *(mandatory — always runs)*

Update `{{absolute_path}}/.claude/skill-memory/write-mockdata/memory.md`:

```markdown
# write-mockdata memory — {{project name}}

## Project
- Path: .
- OData version: {{V2 | V4}}
- Metadata: {{repo-relative path to metadata.xml, e.g. webapp/localService/metadata.xml}}

## Mockdata location
- {{repo-relative path to mockdata folder, e.g. webapp/localService/mockdata/}}

## Completed mockdata files
{{List each file written with a one-line note on purpose}}

## In-progress
{{File name and last completed sub-step, or "none"}}

## Project-specific patterns
{{E.g. which navigations needed getReferentialConstraints, any discriminator fields, known key shapes}}

## Known issues
{{Unresolved problems — e.g. navigation property missing from metadata, property name mismatch}}
```

---

## Step 8 — Summary

Report to the user:

- Files created or updated (full paths)
- Entity sets covered and number of records in each JSON file
- Hooks implemented per JS file
- `getReferentialConstraints` files added (and which navigations they cover)
- OPA test scenarios enabled
- Validation result: JSON and JS syntax clean, or outstanding issues
- Any follow-up steps (e.g. add service alias to `ui5-mock.yaml` for cross-service calls)
