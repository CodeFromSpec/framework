# Specialized Generation Subagents

Status: idea, not designed. Captured on 2026-06-18.

---

## The insight

Artifact generation has three distinct modes depending
on what context is available:

1. **From scratch** — no existing artifact. The subagent
   generates freely from the spec chain. No anchoring
   risk, no need for instructions about preserving
   existing code.

2. **With existing artifact, without delta** — the
   subagent receives the existing code as a reference.
   It must make minimal changes while avoiding anchoring
   on outdated patterns. This is the hardest mode — the
   most context with the least directional signal.

3. **With existing artifact and delta** — the subagent
   receives the existing code and an explicit description
   of what changed in the spec. The task is narrower:
   apply the specific changes, preserve everything else.

Each mode is a different cognitive task with different
risks and different optimal prompting strategies.

## The problem with a single subagent

Today, one subagent definition handles all three modes.
Its prompt must cover all cases: instructions for
generating from scratch, warnings about anchoring when
an existing artifact is present, and guidance for using
a delta when available. These concerns dilute each other
— the subagent receives anchoring warnings when there is
no existing artifact, or delta instructions when there
is no delta.

## Proposed approach

Three subagent definitions, each optimized for its mode.
The orchestrator selects the appropriate subagent based
on what `load_chain` returns:

- No `--- existing artifact ---` section → from-scratch
  subagent.
- Existing artifact present, no delta available →
  incremental subagent (no delta).
- Existing artifact present, delta available →
  delta-aware subagent.

Each subagent's prompt is focused on its specific task
without noise from the other modes.
