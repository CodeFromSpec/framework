---
name: cfs-init-session
description: Load Code from Spec guidelines into the orchestrator context. Run at the start of each session.
---

# Initialize Code from Spec Session

## When invoked

Run this skill at the start of each session when
working on a project that uses Code from Spec.

## What to do

1. Read `CODE_FROM_SPEC.md` from this skill's directory.
   This is the methodology specification — understand
   it and follow it for the remainder of the session.

2. If the `reconstruct_cache` tool is available (via
   the framework-mcp MCP server), call it. This
   rebuilds the cache from the current state of the
   repository.

3. If an `AGENTS.md` file exists at the repository
   root, read it. It contains project-specific
   instructions that apply for the remainder of the
   session.

4. Check for a `CLAUDE.md` file only at the repository
   root — do not search the whole repository, which is
   slower and unnecessary. If it exists, warn the human:
   `CLAUDE.md` is loaded automatically by subagents,
   including generation subagents, and contaminates their
   context — which violates the Code from Spec confinement
   rules. Advise the human to move any project instructions
   into `AGENTS.md` at the repository root, which only the
   orchestrator reads (see step 3). If no `CLAUDE.md` exists
   at the root, say nothing about it — do not mention its
   absence or otherwise reference `CLAUDE.md`.

5. Read the guidelines below.

6. Acknowledge:

   > Code from Spec v7 session initialized.

   Then continue normally.

---

## Guidelines

### Authority

The spec tree under `code-from-spec/` carries the
project's decisions. A conflict between spec and
generated code is won by the spec.

### Working with specs

- Never edit generated code directly. Fix the spec and
  regenerate.
- When a test fails, investigate the spec before the
  code. The bug is almost always a spec gap, not a
  code bug.
- When writing a fix into a spec, state what must be
  true, in the domain's terms — never a correction of
  the current artifact or of the conversation ("do not
  use X", "instead of the previous approach", "no
  longer does Y"). You are writing with the old code
  and this session in context; every future reader of
  the spec — human or generation subagent — has
  neither, and a clause worded against a state that no
  longer exists is unintelligible to them. This is a
  natural failure mode for an AI editing a spec:
  it writes in opposition to what it sees, and what it
  sees is not in the spec.
- Before adding such a clause, apply the resample
  test: would this clause still be wanted if the
  artifact were regenerated from scratch and had never
  had this bug? If yes, it is a real decision — state
  it positively. If it only makes sense against the
  current artifact, it does not belong in `# Public`
  or `# Agent`; record the incident under
  `## Decisions` in `# Private` instead.
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
- Never classify a subagent assumption as "reasonable"
  on your own. Present the subagent's exact text to
  the human. The human decides whether it is
  acceptable or reveals a spec gap.
- Collect all assumptions from a batch before
  advancing to the next rank. Do not accumulate
  them silently across batches.
- Validate between ranks. This is mandatory, not an
  optimization to skip.
- Do not add hints, corrections, or extra context to
  the subagent prompt. The prompt template is fixed.
  If the subagent produces wrong output, the fix goes
  in the spec — not in an ad-hoc prompt addition that
  bypasses the chain.
- Do not delete files without the human's confirmation.
- Do not start generation without the human's approval.

### Debugging

- Start from the spec, not the code. Use the manifest
  to identify which spec produced the failing file.
  Read that spec and the context it inherits.
- Check whether the spec is ambiguous at the point
  where the code went wrong.
- Fix the spec, regenerate, verify. The fix is
  permanent — it applies to all future generations.
- Never blame the subagent. If the subagent produces
  wrong output, investigate what it received in the
  chain before attempting to regenerate. The subagent
  works from the chain alone — if the chain is wrong
  or incomplete, the output will be wrong.
- Before diagnosing the root cause of a test failure,
  present the data to the human instead of concluding
  alone. Wrong diagnoses lead to unnecessary
  regenerations and spec changes that don't address
  the real problem.
- When the same error repeats after regeneration,
  investigate the chain content (what the subagent
  actually sees) rather than retrying. Create a
  diagnostic node that dumps the load_chain output
  if needed.

### What not to do

- Do not fix generated code manually, even for
  "quick fixes." The next regeneration will overwrite
  the fix. 
- Do not add comments to generated code. The spec
  tree is the documentation.
- Do not assume the generated code will follow a
  convention unless the spec states it. If it matters,
  put it in a spec that the relevant files inherit.
- Do not use CLAUDE.md for Code from Spec rules or
  project instructions. CLAUDE.md is loaded by
  subagents and will contaminate the generation
  process. Orchestrator guidelines belong in this
  session skill, and project-specific instructions
  belong in `AGENTS.md` at the repository root — not in
  files that subagents can see.
- Do not call `prune_orphans` without the human's
  explicit request. Pruning deletes artifacts from
  disk — a destructive, irreversible action that may
  remove files the human still needs (e.g. an artifact
  that was moved and needs regeneration at its new
  path, not deletion).
