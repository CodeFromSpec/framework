# Variability Analysis

Variability analysis is the process of identifying where
a spec is under-specified by observing variation in the
generated output. It is the primary mechanism by which
specs converge toward precision.

---

## The problem

A spec written in natural language is inherently less
precise than a formal grammar. When two regenerations of
the same spec produce different code, the differences
reveal points where the spec left room for interpretation.
Some differences are cosmetic (variable names, import
order) and can be accepted. Others are behavioral (error
handling strategy, query structure) and must be resolved
by tightening the spec.

---

## Manual process (current)

1. Generate code from a spec.
2. Regenerate (different session or after clearing
   artifacts).
3. Diff the two outputs.
4. For each difference, classify:
   - **Cosmetic** — accept. The spec does not need to
     prescribe this.
   - **Behavioral** — prescribe. Add a concrete rule to
     the spec that eliminates the variation.
5. Regenerate. Repeat until stable.

Each cycle makes the spec more precise. The spec
converges toward a point where regeneration is
deterministic — not because the LLM became deterministic,
but because the spec left no room for interpretation on
points that matter.

---

## Automated process (planned)

An agent-assisted workflow that:

1. Generates code from a spec twice (different
   sessions).
2. Diffs the two outputs.
3. Classifies each difference as cosmetic or
   behavioral.
4. Reports behavioral differences as spec gaps.

This could run periodically to monitor spec precision
and identify nodes that need tighter prescription.

---

## Relationship to convergence

Variability analysis is the diagnostic tool.
Convergence is the outcome. Each round of analysis
produces spec improvements. Each improvement reduces
variability. The trajectory is monotonic: specs only
get more precise, never less.

This is analogous to undefined behavior in programming
language specifications. When C was young, `i = i++`
was undefined — the spec was under-specified. The
standards committee eventually prescribed behavior.
The difference is that in Code from Spec, this
convergence happens continuously, guided by observed
output rather than committee deliberation.
