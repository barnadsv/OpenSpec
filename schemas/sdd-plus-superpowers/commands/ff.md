---
name: "OPSX: Fast Forward"
description: "Create a change and generate ALL artifacts in one step (sdd-plus-superpowers)"
category: Workflow
tags: [workflow, fast-forward, experimental]
---

Fast-forward: create a new change with the `sdd-plus-superpowers` schema and generate all artifacts up to `plan.md` in one step.

This is the recommended quick flow when you already have a clear idea of what to build.

**Input**: `/opsx:ff <name>` where `<name>` is a kebab-case change name.

**Steps**

1. **Parse input**

   Extract the change name from the argument string.

   If no name provided, use the **AskUserQuestion tool** to ask:
   > "What's the name for this change? Use kebab-case (e.g., `add-dark-mode`)."

2. **Create the change**

   ```bash
   openspec new change "<name>" --schema sdd-plus-superpowers
   ```

   If a change with that name already exists, ask if the user wants to continue it or pick a new name.

3. **Get the artifact build order**

   ```bash
   openspec status --change "<name>" --json
   ```

   Parse the JSON to get:
   - `applyRequires`: array of artifact IDs needed before implementation
   - `artifacts`: list of all artifacts with status and dependencies

4. **Create artifacts in sequence until apply-ready**

   Use the **TodoWrite tool** to track progress through the artifacts.

   Loop through artifacts in dependency order (artifacts with no pending dependencies first):

   a. **For each artifact that is `ready` (dependencies satisfied)**:
      - Get instructions:
        ```bash
        openspec instructions <artifact-id> --change "<name>" --json
        ```
      - The instructions JSON includes:
        - `context`: project background (constraints for you — do NOT include in output)
        - `rules`: artifact-specific rules (constraints for you — do NOT include in output)
        - `template`: the structure to use for the output file
        - `instruction`: schema-specific guidance for this artifact type
        - `outputPath`: where to write the artifact
        - `dependencies`: completed artifacts to read for context
      - Read any completed dependency files for context
      - **If `instruction` says to invoke a Skill** (e.g., `superpowers:brainstorming`, `superpowers:writing-plans`), invoke it
      - Create the artifact file using `template` as the structure
      - Apply `context` and `rules` as constraints — do NOT copy them into the file
      - Show brief progress: "Created **<artifact-id>**"

   b. **Handle optional artifacts** (e.g., `intent`):
      - Skip optional artifacts unless the user provided input for them (e.g., a PRD or external source)
      - If skipping, announce: "Skipping **<artifact-id>** (optional)"

   c. **Continue until all `applyRequires` artifacts are complete**
      - After creating each artifact, re-run `openspec status --change "<name>" --json`
      - Check if every artifact ID in `applyRequires` has `status: "done"`
      - Stop when all `applyRequires` artifacts are done

   d. **If an artifact requires user input** (unclear context):
      - Use **AskUserQuestion tool** to clarify
      - Then continue with creation

5. **Show final status**

   ```bash
   openspec status --change "<name>"
   ```

**Output**

After completing all artifacts, summarize:

```
## Fast Forward Complete: <name>

**Schema:** sdd-plus-superpowers
**Location:** openspec/changes/<name>/

### Artifacts created
- ✓ brainstorm
- ✓ proposal
- ✓ design
- ✓ specs
- ✓ tasks
- ✓ plan
- ○ intent (skipped — optional)

All artifacts created! Ready for implementation.
Run `/opsx:apply` to start implementing.
```

**Guardrails**
- Always uses the `sdd-plus-superpowers` schema (hardcoded)
- Create ALL artifacts needed for implementation (as defined by `apply.requires`)
- Always read dependency artifacts before creating a new one
- If context is critically unclear, ask the user — but prefer making reasonable decisions to keep momentum
- Verify each artifact file exists after writing before proceeding to next
- `context` and `rules` are constraints for YOU — never copy them into the artifact file
- If an artifact's `instruction` says to invoke a Skill, invoke it — this is what integrates Superpowers into the flow
