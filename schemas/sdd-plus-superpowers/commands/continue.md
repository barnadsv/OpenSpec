---
name: "OPSX: Continue"
description: "Continue a change by creating the next pending artifact"
category: Workflow
tags: [workflow, continue, experimental]
---

Continue a change by creating the next pending artifact in the pipeline.

**Input**: Optionally specify a change name (e.g., `/opsx:continue add-auth`). If omitted, infer from context or prompt.

**Steps**

1. **Select the change**

   If a name is provided, use it. Otherwise:
   - Infer from conversation context if the user mentioned a change
   - Auto-select if only one active change exists
   - If ambiguous, run `openspec list --json` and use the **AskUserQuestion tool** to let the user select

   Always announce: "Continuing change: **<name>**"

2. **Get current status**

   ```bash
   openspec status --change "<name>" --json
   ```

   Parse the JSON to understand:
   - `schemaName`: the workflow being used
   - `artifacts`: list with status and dependencies
   - `applyRequires`: what's needed before apply

3. **Find the next artifact to create**

   From the artifacts list, find the first artifact where:
   - `status` is NOT `done`
   - All entries in `requires` have `status: "done"` (dependencies satisfied)
   - If `optional: true` and no upstream data feeds it, ask the user whether to create or skip it

   **If no artifact is pending** (all done or all blocked):
   - If all `applyRequires` are satisfied: "All artifacts ready! Run `/opsx:apply` to implement."
   - If blocked: show which artifacts are blocked and what they need

4. **Create the artifact**

   Get instructions for the selected artifact:
   ```bash
   openspec instructions <artifact-id> --change "<name>" --json
   ```

   The instructions JSON includes:
   - `context`: project background (constraints for you — do NOT include in output)
   - `rules`: artifact-specific rules (constraints for you — do NOT include in output)
   - `template`: the structure to use for the output file
   - `instruction`: schema-specific guidance for this artifact type
   - `outputPath`: where to write the artifact
   - `dependencies`: completed artifacts to read for context

   Process:
   a. Read all completed dependency files for context
   b. Follow the `instruction` field — if it says to invoke a Skill (e.g., `superpowers:brainstorming`), do so
   c. Create the artifact file using `template` as the structure
   d. Apply `context` and `rules` as constraints — do NOT copy them into the file

5. **Show updated status and next steps**

   ```bash
   openspec status --change "<name>"
   ```

   - Show which artifact was created
   - Show what's next in the pipeline
   - Suggest `/opsx:continue` again for the next one, or `/opsx:apply` if ready

**Output**

```
## Created: <artifact-id>

**Change:** <name>
**Artifact:** <artifact-id> → <outputPath>

### Next in pipeline
- <next-artifact> (ready)
- <another-artifact> (blocked by: <dependency>)

Run `/opsx:continue` for the next artifact, or `/opsx:apply` when ready.
```

**Guardrails**
- Create only ONE artifact per invocation — the user controls the pace
- If an artifact's `instruction` says to invoke a Skill, invoke it
- If the artifact requires user input (unclear context), use **AskUserQuestion** to clarify
- Always read dependency artifacts before creating a new one
- `context` and `rules` are constraints for YOU — never copy them into the artifact file
- If all artifacts are done, do not create anything — just suggest `/opsx:apply`
