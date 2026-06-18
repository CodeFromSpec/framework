# Agent Failure Modes

The methodology depends on AI agents. Agents are not
infallible. This document catalogs known failure modes
and the mitigations in place or planned.

---

## Literal interpretation of ambiguous specs

Spec says "only accepts ROOT/ references." Agent reads
this literally and rejects `ROOT` (without trailing
slash). Spec says "include the public section content."
Agent omits the `# Public` heading because the spec
did not say to include headings.

Every ambiguity that seems clear to a human has two
reasonable interpretations for an agent.

*Mitigation:* Iterative precision. Each failure traces
to a spec gap. Fix the spec, regenerate. The gap is
closed permanently.

---

## Test scenarios silently omitted

The spec listed a test case. The agent did not generate
it and did not report the omission. The gap was found
during manual review.

*Mitigation:* The artifact generation skill requires
the subagent to report assumptions and gaps. But this
depends on the agent noticing the omission — which is
the same agent that omitted it. Defense in depth: test
specs should be prescriptive enough that omissions
cause compilation errors (e.g., a test that references
a specific setup value will fail to compile if the
setup is missing).

---

## Cosmetic variation between regenerations

Variable names, struct organization, helper function
grouping differ between regenerations of the same spec.
Pollutes git diffs and makes review harder.

*Mitigation adopted:* No comments in generated code
(comments are the most variable part). Prescribe names
in the spec — function names, error names, record
names, field names. The more the spec prescribes, the
less the agent invents.

*Mitigation planned:* Code formatter pass after
generation to normalize whitespace and import order.

---

## Over-engineering

Agent adds abstractions, error handling, features, or
patterns not requested by the spec.

*Mitigation:* Subagent rules say "write straightforward
code." But this is a soft constraint. The real defense
is tests — over-engineered code that passes the same
tests as simple code is harmless, just noisy.

---

## Hallucinated imports or packages

Agent invents an import path that does not exist.

*Mitigation:* Build verification catches this
immediately. The regeneration cycle always ends with
build + test.

---

## Under-specification

Spec is too vague. Agent fills gaps with reasonable but
unpredictable choices. Different runs produce different
implementations.

*Mitigation:* Variability analysis — review generated
code, classify each variable aspect as "accept" or
"prescribe," tighten the spec. This is ongoing work,
not a one-time fix. Specs get more precise over time.

---

## Context window exceeded

With hundreds of nodes, the chain from root to a deep
leaf may exceed the agent's context window. The agent
would miss constraints from ancestor nodes.

*Mitigation:* The chain mechanism naturally limits
context to what the node declared. Keep specs concise.
Remove redundancy aggressively. In practice, even deep
chains stay well within current context limits (1M
tokens) for individual node generation.
