---
name: cfs-init-session
description: Load Code from Spec guidelines into the orchestrator context. Run at the start of each session to prime the orchestrator with the methodology's working rules.
---

# Initialize Code from Spec Session

Load the working guidelines for a Code from Spec
session. This skill exists to inject context into
the orchestrator without polluting subagent context
(which would happen if this content lived in
CLAUDE.md).

## When invoked

Run this skill at the start of each session when
working on a project that uses Code from Spec.
The user may invoke it explicitly (`/cfs-init-session`)
or it may be triggered by a hook.

## What to do

Read the guidelines below and follow them for the
remainder of the session. Do not output the guidelines
to the user — just acknowledge that the session is
initialized.

Respond with:

> Code from Spec session initialized.

Then continue normally.

---

## Guidelines

### Source of truth

The spec tree under `code-from-spec/` is the source of
truth. Generated code is a derived artifact. When the
two conflict, the spec wins.

### Working with specs

- Never edit generated code directly. Fix the spec and
  regenerate.
- When a test fails, investigate the spec before the
  code. The bug is almost always a spec gap, not a
  code bug.
- When the human makes a decision between alternatives,
  suggest recording it in a private section
  (`# Decisions`, `# Rationale`) with what was chosen,
  what was discarded, and why.
- When a pattern or convention should apply broadly,
  suggest adding it to an ancestor node (guard node)
  rather than repeating it in each leaf.
- When a bug repeats across multiple artifacts, fix
  the ancestor, not each leaf individually.
- Implicit knowledge does not survive regeneration.
  If the agent should know it, the tree must say it.

### Generation workflow

- After any spec change, run `validate_specs` (via
  `/cfs-status`) before generating code.
- Generate stale artifacts with `/cfs-generate`.
  Process in rank order — lower ranks first.
- After generation, run build and tests before
  reporting success.
- If a subagent reports assumptions or spec gaps,
  surface them to the human before continuing.

### Debugging

- Start from the spec, not the code. Read the node
  that generated the failing artifact and its chain.
- Trace the artifact tag in the failing file back to
  its source node.
- Check whether the spec is ambiguous at the point
  where the code went wrong.
- Fix the spec, regenerate, verify. The fix is
  permanent — it applies to all future generations.

### What not to do

- Do not fix generated code manually, even for
  "quick fixes." The next regeneration will overwrite
  the fix.
- Do not add comments to generated code. The spec
  tree is the documentation.
- Do not assume the agent will infer conventions.
  If it matters, put it in a spec node that the
  relevant leaves inherit.
