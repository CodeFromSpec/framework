---
name: cfs-init-session
description: Load Code from Spec guidelines into the orchestrator context. Run at the start of each session.
---

# Initialize Code from Spec Session

## When invoked

Run this skill at the start of each session when
working on a project that uses Code from Spec.

## What to do

1. Read `code-from-spec/_rules/CODE_FROM_SPEC.md`.
   This is the methodology specification — understand
   it and follow it for the remainder of the session.

2. Read the guidelines below.

3. Acknowledge:

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
- When the human makes a decision between real
  alternatives (not trivial naming or formatting — the
  kind where a future reader would ask "why this and
  not that?"), record it under `## Decisions` in the `# Private`
  section of the relevant node. Include what
  was chosen, what was considered and discarded, and
  why. Do not ask permission for this — it is part of
  the normal workflow, like updating a spec after a
  bug fix.
- When a pattern or convention should apply broadly,
  suggest adding it to a parent `_node.md` so all
  descendants inherit it, rather than repeating it in
  each spec that needs it.
- When a bug repeats across multiple generated files,
  fix the parent spec that they all inherit from, not
  each individual spec.
- Implicit knowledge does not survive regeneration.
  If the generated code should follow a rule, the spec
  tree must state it.

### Generation workflow

- After any spec change, run `/cfs-status` before
  generating code.
- Generate stale artifacts with `/cfs-generate`.
- After generation, run build and tests only when the
  human asks — do not run them automatically.
- If a subagent reports assumptions or spec gaps,
  stop and surface them to the human before continuing.
  Each assumption is a potential spec gap. Never
  proceed past assumptions without discussion.
- Validate between ranks. This is mandatory, not an
  optimization to skip.
- Do not add hints, corrections, or extra context to
  the subagent prompt. The prompt template is fixed.
  If the subagent produces wrong output, the fix goes
  in the spec — not in an ad-hoc prompt addition that
  bypasses the chain.
- When a spec's behavior changes (not just a hash
  cascade), delete the existing artifact before
  regenerating. The existing artifact anchors the
  subagent on old patterns and can cause spec changes
  to be silently ignored.
- Do not delete files without the human's confirmation.
- Do not start generation without the human's approval.

### Debugging

- Start from the spec, not the code. Find the
  `code-from-spec:` tag in the failing file to
  identify which spec produced it. Read that spec
  and the context it inherits.
- Check whether the spec is ambiguous at the point
  where the code went wrong.
- Fix the spec, regenerate, verify. The fix is
  permanent — it applies to all future generations.
- Never blame the subagent. If the subagent produces
  wrong output, investigate what it received in the
  chain before attempting to regenerate. The subagent
  works from the chain alone — if the chain is wrong
  or incomplete, the output will be wrong.
- When the same error repeats after regeneration,
  investigate the chain content (what the subagent
  actually sees) rather than retrying. Create a
  diagnostic node that dumps the load_chain output
  if needed.
- When an existing artifact was present and the
  subagent did not apply a spec change, check whether
  the subagent anchored on the existing artifact.
  Delete the artifact and regenerate from scratch.

### What not to do

- Do not fix generated code manually, even for
  "quick fixes." The next regeneration will overwrite
  the fix. The one exception is self-referential bugs
  where the tool's own code prevents correct
  regeneration — document these and fix them in the
  spec as soon as possible.
- Do not add comments to generated code. The spec
  tree is the documentation.
- Do not assume the generated code will follow a
  convention unless the spec states it. If it matters,
  put it in a spec that the relevant files inherit.
- Do not use CLAUDE.md for Code from Spec rules.
  CLAUDE.md is loaded by subagents and will
  contaminate the generation process. Orchestrator
  guidelines belong in this session skill, not in
  files that subagents can see.
