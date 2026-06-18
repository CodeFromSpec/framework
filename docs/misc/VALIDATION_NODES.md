# Validation Nodes

Status: idea, not designed. Captured from a brainstorm on
2026-06-13.

---

## The concept

A **validation node** is an ordinary leaf node whose generated
artifact is a list of problems instead of code:

```yaml
---
input: ARTIFACT/X            # the artifact to validate
depends_on:                  # the context to validate against
  - ARTIFACT/...
output: .../problems.md      # the list of problems found
---
```

Its `# Agent` section holds the validation criteria (a coverage
check, an omission checklist, ambiguity hunting). A generation
subagent reads X plus the chain and produces the problems list.

It introduces no new framework primitive — it is generation
where the artifact happens to be a critique rather than code.

## What it buys

- **Validation becomes staleness-tracked.** The problems list
  has a chain hash. When X changes, the validation node goes
  stale — the framework itself tells you the validation is out
  of date and must be re-run. Today `cfs-spec-review` is a skill
  you must remember to invoke; as a node, the staleness machine
  reminds you.
- **Reuses the generation mechanism.** No new concept; a leaf
  with `input` + `output` + `# Agent`.
- **Confinement applies for free.** The subagent receives
  exactly X plus the chain, so its findings are grounded.

## The core tension and its resolution

Every other artifact is meant to be a stable function of its
chain — code variation is cosmetic. A list of problems is a
**judgment**: the agent may find five problems one run and four
the next, and the difference is substantive, not cosmetic. A
judgment-shaped thing is going into a slot designed for
derivation-shaped things.

This is contained by one rule:

> **Validation nodes are terminal. Nothing may `input` or
> `depends_on` the output of a validation node.**

A validation artifact is consumed only by humans or CI, never
by another generation. Its non-determinism therefore never
propagates into another chain — it is always a dead end in the
dependency graph, so it cannot poison the chain invariant
anywhere else.

## Semantic caveat

A validation node being stale means "X changed, re-check." A
validation node being current means "checked at this version" —
**not** "approved." An empty, current problems list says the
last run found nothing, not that X is correct. The tooling must
present it that way.

## Where it sits: a taxonomy of validation

- **Mechanical validation** (format errors, circular
  references, staleness) — stays in the `validate_specs` tool.
  Deterministic checks, not nodes.
- **Judgment validation** (omissions, coverage, ambiguity) —
  becomes a validation node. Needs an LLM, and benefits from
  staleness-tracking ("re-judge when the input changes").

The dividing line is "is this a check or a judgment?" — and it
decides where the validation lives.

## Relationship to cfs-spec-review

The existing `cfs-spec-review` subagent already classifies
aspects as Clear / Ambiguous / Omitted / Inconsistent, and can
already receive a full chain (for a functional node: the domain
as `input`, the spec as target, the generated pseudocode as
existing artifact). A validation node is essentially the
node-form of that review: the same confined analysis, but
persisted as a stale-tracked artifact rather than invoked
ad hoc.

A natural specialization is a **cross-layer omission hunter**:
a validation node over a pseudocode artifact whose `depends_on`
is the domain it came from, tasked with finding domain concepts
the pseudocode fails to cover, plus a checklist of generic edge
cases (zero, negative, empty, maximum, concurrent, on-error).
It catches coverage gaps and generic edges; it cannot catch
domain knowledge nobody wrote down (that is outside the chain,
and surfaces later as a bug — the immune-system loop).

## Application: unit tests for pseudocode

A validation node is the natural tool for **unit-testing
pseudocode** — verifying logic before any code exists.

Pseudocode is not executable, but an LLM can *trace* it. A
validation node whose `# Agent` holds test cases — inputs and
expected outcomes stated in domain language ("purchase of 100
with limit 50 -> denied") — traces the pseudocode against each
case and reports which cases it fails to satisfy. The omission
"doesn't handle zero amount" surfaces as a failed pseudocode
unit test, at the functional layer, the cheapest place to fix
it. Verification shifts left, before code generation.

Two properties make this fit the methodology:

- **The layperson authors the test cases.** They are
  acceptance criteria in business terms — exactly what a domain
  expert can write and validate.
- **The test intent is authored once and used twice.** The same
  domain-language cases drive the pseudocode validation node
  (early, by reasoning) and later become the spec for the
  executable unit tests at the code layer (late, by execution).
  Test knowledge flows down the pipeline like everything else.

Honest limit: this relies on the LLM tracing the pseudocode
correctly — it is reasoning, not execution, and can be wrong.
It is an early, cheap, fallible filter — a smoke detector, not
a proof. The deterministic, executable unit tests still happen
at the code layer.

## Open questions

- **Versioning.** Should a problems list be committed (audit
  trail: "what problems existed at commit X") or treated as
  transient CI output, like a test result? The endgame view
  ("code as build artifact") suggests transient; auditability
  suggests committed. Possibly project-dependent.
- **Granularity.** One validation node per artifact keeps
  chains small and findings focused. A whole-tree validator
  would have an enormous chain — avoid.
