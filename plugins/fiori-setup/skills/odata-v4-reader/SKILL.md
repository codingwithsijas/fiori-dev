---
name: odata-v4-reader
description: Explore an OData V4 service — list entity sets, inspect properties and key fields, trace navigation properties and associations, and retrieve sample data. Use before building a new Fiori app or feature to understand the service structure without having to read raw metadata XML.
---

# OData V4 Reader

Explore and document an OData V4 service. Use this before scaffolding a project or implementing a feature when you need to understand what the service exposes — entity sets, properties, associations, and live data samples.

## When to Use
- Discovering which entity sets a service exposes before scaffolding
- Finding the correct key property name and type for `entityConfig`
- Tracing navigation properties needed for `$expand` in bindings
- Sampling live data to validate property names and formats
- Confirming that a service URL is reachable and responding

---

## Step 0 — Read Skill Memory

Check for an existing memory file for this skill in the project:

```bash
cat .claude/skill-memory/odata-v4-reader/memory.md 2>/dev/null
```

If found, read it in full. If the service has been explored before, present the cached summary to the user and ask whether to re-fetch or use the cached result. Only re-invoke the agent if the user confirms the service may have changed.

If not found, proceed — memory will be created at the end of this session.

---

## Step 1 — Get the Service URL

If the user has not provided a URL, check:

```bash
cat webapp/manifest.json 2>/dev/null | grep -A5 '"dataSources"'
```

Extract the OData V4 service URL from `sap.app/dataSources`. If still not found, ask the user for the service URL before proceeding.

---

## Step 2 — Invoke the `odata-v4-reader` Agent

Delegate all service exploration to the `odata-v4-reader` agent. Pass it:
- The full service URL (including `$metadata` endpoint if known)
- The specific question or entity the user wants to explore
- Any navigation properties or associations to trace

The agent will query the service metadata and return a structured summary. Present its output directly to the user — do not reformat or re-interpret it.

---

## Step 3 — Surface Actionable Next Steps

After presenting the agent's output, call out what the user needs for their next action:

- The **correct entity set name** (case-sensitive) to use in `entityConfig.mainEntity`
- The **key property** to use in routing patterns (e.g. `/{KeyProperty}`)
- Navigation properties needed for `$expand` in bindings
- Any warnings the agent flagged (deprecated properties, missing annotations, auth issues)

Suggest the next command based on what the user is trying to do — e.g.:
- "Run `/project-setup` with entity set `{{EntitySet}}` and key `{{KeyProperty}}`"
- "Run `/fiori-feature-dev` to add a detail page bound to `{{NavProperty}}`"

---

## Step 4 — Optional: Sample Live Data

If the user wants to see real data, ask the agent to fetch a small sample (top 3–5 records). Present the agent's response directly.

---

## Step 5 — Update Skill Memory

Create or update `.claude/skill-memory/odata-v4-reader/memory.md` in the current working directory:

```markdown
# odata-v4-reader memory

## Service
- URL: {{service_url}}
- Last explored: {{date}}

## Entity sets
{{One line per entity set: name, key field(s), draft-enabled yes/no}}

## Navigation properties
{{One line per nav property: source entity → property name → target entity → cardinality}}

## Key findings
{{Anything noteworthy the agent flagged — deprecated properties, missing annotations, auth issues, SAP-specific annotations relevant to Fiori}}

## Used in
{{Project name or path this service is bound to, if known}}
```

Write the memory file even on the first run so future sessions can skip the service fetch when the structure hasn't changed.
