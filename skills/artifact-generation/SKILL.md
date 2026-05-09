---
name: artifact-generation
description: Generates or regenerates artifacts from the Code from Spec tree. Use when the staleness-check tool reports stale artifacts, or when the user asks to generate or regenerate.
---

# Artifact Generation

Generate artifacts for all stale items reported by the
staleness-check tool.

## When invoked

Run this skill when the user asks to generate artifacts,
regenerate files, or when the staleness-check tool reports stale
items.

## Prerequisites

1. Verify the staleness-check binary exists
   (`tools/staleness-check.exe` on Windows,
   `tools/staleness-check` elsewhere). If not, tell the user it
   is missing and stop.

2. Verify the subagent-mcp binary exists
   (`tools/subagent-mcp.exe` on Windows,
   `tools/subagent-mcp` elsewhere). If not, tell the user it is
   missing and stop.

## Algorithm

1. Run the staleness-check tool and collect all stale items.
2. If no items are stale, report that everything is up to date
   and stop.
3. Group items by `node` — each unique logical name is one
   generation task.
4. For each stale node (in the order reported), dispatch a
   `code-from-spec-code-generation` subagent with the following
   prompt:

   > You are a confined artifact generation subagent.
   > Your only task is to generate the artifact(s) for the node
   > `<logical-name>`.
   >
   > Steps:
   > 1. Call `load_chain` with logical_name `<logical-name>` to
   >    receive the complete spec chain.
   > 2. Read the chain carefully. Identify the target node's
   >    `# Public` and `# Agent` sections, the constraints from
   >    ancestor nodes, and any dependency content.
   > 3. For each artifact declared in the node's `artifacts` list,
   >    generate the complete file content. The artifact tag must
   >    appear as early in the file as the format allows:
   >    `code-from-spec: <logical-name>@<hash>`
   >    where `<hash>` is the chain hash provided by `load_chain`.
   > 4. Call `write_file` once per artifact, passing the logical
   >    name, the relative file path, and the complete content.
   > 5. If the spec has gaps or contradictions that prevent
   >    generation, do not guess — report the problem clearly
   >    instead of writing a file.
   > 6. After generating, list any assumptions you made where the
   >    spec was silent or ambiguous. Label this section
   >    `## Assumptions`. Include: format choices, column/field
   >    mappings you inferred, interpretations of ambiguous
   >    wording. If there are none, omit the section.
   >
   > Do not read any file not provided by `load_chain`. Do not
   > call any tool other than `load_chain` and `write_file`.

5. After all subagents complete, run the staleness-check tool
   again. Report the remaining stale items (if any) to the user.

## Rules

- Dispatch one subagent per node logical name, not per artifact.
- Independent nodes may be dispatched in parallel (single message
  with multiple Agent tool calls).
- Never edit generated artifacts manually — always regenerate via
  a subagent.
- After each subagent completes, check its output for an
  `## Assumptions` section or any language indicating the spec
  was ambiguous, silent, or required interpretation (e.g., "the
  spec does not specify", "chose", "assumed", "not defined").
  Collect all such items and present them to the user **before**
  reporting success. These are potential spec gaps that need
  confirmation.
- If a subagent reports a spec gap that prevented generation,
  surface it to the user. Do not attempt to fill the gap by
  reading the codebase yourself.
- After generation, do not automatically run build or tests
  unless the user asks — report what was generated and let the
  user decide.
