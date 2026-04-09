---
name: "OPSX: New"
description: "Create a new OpenSpec change with optional schema selection"
category: Workflow
tags: [workflow, create, experimental]
---

Create a new OpenSpec change.

**Input**: `/opsx:new <name> [--schema <schema-name>]`

- `<name>`: kebab-case name for the change (e.g., `add-user-auth`)
- `--schema`: optional schema override (default: project's `config.yaml` schema)

**Steps**

1. **Parse input**

   Extract the change name and optional `--schema` flag from the argument string.

   If no name provided, use the **AskUserQuestion tool** to ask:
   > "What's the name for this change? Use kebab-case (e.g., `add-dark-mode`)."

2. **Create the change**

   ```bash
   openspec new change "<name>"
   ```

   If `--schema` was provided:
   ```bash
   openspec new change "<name>" --schema "<schema>"
   ```

3. **Check for schema-specific commands**

   After creating the change, determine the schema being used:
   ```bash
   openspec status --change "<name>" --json
   ```

   Check if the schema has extra commands to install:
   ```bash
   ls openspec/schemas/<schemaName>/commands/*.md 2>/dev/null
   ```

   **If commands exist but are NOT yet in `.claude/commands/opsx/`:**
   - Inform the user:
     > "The schema **<schemaName>** includes extra commands (`ff`, `continue`, etc.) that aren't installed yet.
     > Run `cp openspec/schemas/<schemaName>/commands/*.md .claude/commands/opsx/` to install them."
   - Do NOT block — proceed to step 4.

4. **Show status and guide next steps**

   ```bash
   openspec status --change "<name>"
   ```

   Display the artifact pipeline for this schema and suggest next actions:
   - `/opsx:continue` to create artifacts one at a time
   - `/opsx:ff <name>` to fast-forward (create all artifacts at once)
   - `/opsx:explore <name>` to think through the idea first

**Guardrails**
- If a change with that name already exists, ask if the user wants to continue it or pick a new name
- Always use kebab-case for change names
- Do not create any artifacts — this command only scaffolds the change directory
