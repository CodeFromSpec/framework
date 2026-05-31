# Code from Spec — Rationale

Code from Spec is a methodology where code is a generated
artifact, not the source of truth. Specifications are the
source of truth. To change behavior, you change the
specifications and regenerate the code.

AI is the enabler, not the point. The disruption is in
who participates, how knowledge flows, and where
accountability lives.

---

## The problem

Software is written by people who hold context in their
heads. The engineer receives requirements, translates
them into code, and in that translation makes hundreds
of decisions that are never recorded. When the engineer
leaves, the decisions leave too.

Code expresses mechanism, not intent. You can read code
and understand what it does. You cannot read it and
understand why, what alternatives were considered, or
what constraints it silently respects.

The industry built compensating mechanisms: comments,
wikis, ADRs, onboarding docs. None work at scale because
they exist separately from the system. They describe a
system that changes independently. They drift. The team
stops trusting them. The knowledge returns to people's
heads.

---

## Why specifications failed before

The 1970s and 1980s produced rigorous methods for
capturing domain knowledge before writing code. They
failed not because they were wrong but because they were
expensive. Maintaining a specification in sync with
evolving code required constant manual effort. The spec
drifted. The cost exceeded the benefit.

The industry responded with agility: shorter cycles,
working software over documentation. This was rational.
If specifications cannot be kept current, get feedback
faster instead.

Agile solved the bottleneck by removing the spec. The
knowledge became invisible — encoded in code that only
the programmer could read.

---

## AI changes the economics

AI inverts the cost structure. Code generation is cheap.
The scarce resource is no longer writing code — it is
knowing what to write.

When code is generated from spec, synchronization is
automatic by construction. The spec does not drift from
the code because the code is derived from the spec. The
argument that killed formal specification in the 1980s
no longer applies.

---

## How it works

Specifications are organized as a tree. Each node adds
precision to its parent — high-level intent at the root,
implementation detail at the leaves. Only leaf nodes
generate artifacts.

An orchestrator dispatches a generation subagent for each
stale artifact. The subagent receives the **chain** — the
ordered set of ancestor constraints, dependency
interfaces, external references, and the target node's
specification. The chain is the complete context. Nothing
outside it is needed.

The subagent produces one of two results: generated
artifacts, or a findings report identifying what is
ambiguous or missing in the specification. Both outcomes
are equally valid.

---

## Why it can be trusted

AI is not infallible. The methodology compensates through
structure:

**Small scope.** Each leaf generates a small, focused
piece of code. The subagent is not asked to generate an
entire system.

**Complete context.** The subagent reads the full chain.
Every constraint from every ancestor is present. It does
not guess — it reads.

**Tests as guardrails.** Every leaf has a sibling test
node. Tests describe expected behavior, not
implementation. The generated code must pass them.

**Build verification.** Every regeneration ends with
build + test. If the code doesn't compile or pass, the
spec is traced back and corrected.

This is not blind trust. It is a framework that uses AI's
strengths (fast generation from precise context) while
compensating for its weaknesses (hallucination,
inconsistency) with structural guardrails.

---

## Precision, not documentation

The word "specification" suggests a document. In
practice, a spec node is a machine component that must
fit precisely with every other part.

Precision means: every error has a formal name. Every
function name is chosen once and used identically across
every layer. Every record field has an explicit type.
Every test case prescribes the setup that produces the
expected result, not just the expected result.

When a spec says "file unreadable," different agents
produce different sentinel names, different wrapping
patterns, different test assertions. When a spec says
`FileUnreadable`, every agent produces
`ErrFileUnreadable`. The difference between prose and
formal names is the difference between approximate and
exact generation.

This precision is expensive to achieve. It is paid once.
Every subsequent regeneration benefits.

---

## The real work

Spec authoring is harder and more iterative than it
appears. A spec that seems clear to a human may be
ambiguous to an agent. The agent makes a reasonable
choice — and it is wrong. The tests fail. The team
diagnoses, discovers the ambiguity, adds a constraint,
and regenerates. A single leaf may go through ten
iterations before the spec reliably produces correct
code.

This is not failure. It is the methodology working.
Each iteration makes the spec more precise, the
generated code more predictable, and the team's
understanding of the domain more explicit. The knowledge
that was implicit becomes explicit in the spec. That
explicit knowledge is the asset.

The ultimate test is full regeneration: delete every
artifact and regenerate from scratch. If the result is a
working system, the specs are sufficient. If not, the
failures point directly to spec gaps.

---

## Tests as accumulated knowledge

Test nodes describe what to verify, not how to implement.
But generated test code often contains knowledge beyond
the spec: specific values that triggered a production
bug, sequences that exposed a race condition, assertions
that catch subtle regressions.

This knowledge accumulates. Every bug adds a scenario.
Every edge case adds a verification. The test file is a
living record of the system's failure modes.

Before regenerating a test file, review it for knowledge
that lives only in code. Migrate that knowledge to the
test spec. Then regenerate. This is the cost of the
methodology — and the mechanism by which the spec tree
absorbs the organization's learning.

---

## Software as a collaborative product

Code from Spec makes every contributor a direct author —
not by making everyone a programmer, but by giving each
domain expert a medium they can read and evaluate.

- A compliance officer contributes regulatory constraints.
- A product manager contributes business rules.
- A legal team member contributes contract
  interpretations.
- An infosec engineer contributes security constraints.
- A software engineer contributes technical constraints.

Consider a company building a credit card system. In
traditional development, the engineering team builds
the settlement logic and shows it to accounting a week
before production. Accounting discovers that
provisioning uses the wrong base amount, that journal
entries don't follow the chart of accounts, that
reconciliation assumes calendar days instead of business
days. These are not edge cases — they are fundamental
rules that the engineering team never knew.

With Code from Spec, the accounting team writes the
spec nodes for provisioning, journal entries, and
reconciliation. The engineer contributes the technical
constraints: concurrency, idempotency, error recovery.
Neither overwrites the other. Both combine in the tree,
and the agent produces code that satisfies both.

Guard nodes at intermediate levels enforce constraints
that all descendants must respect — security policies,
error handling standards, architectural patterns. A
contributor who adds a new endpoint cannot accidentally
bypass security requirements. The constraints are above
them in the tree. The agent reads them. The generated
code respects them.

The spec makes quality observable. When the compliance
officer reviews a spec node, they can tell whether the
rules are right — before any code is generated. The
incentive realigns: correctness is no longer invisible,
and shortcuts are no longer undetectable.

---

## AI as spec co-author

The same AI that generates code helps write specs. A
non-technical contributor describes behavior in natural
language; the agent structures it into a valid spec node.

In practice, AI participates in every phase: reviewing
specs for inconsistencies, proposing alternatives,
diagnosing generation failures, tracing bugs to spec
gaps. But AI does not make design decisions. The human
asks "should this feature exist?" and "would a newcomer
understand this structure?" — questions that require
judgment about purpose, not just consistency.

The productive model is: AI handles volume (reviewing
every node, checking every reference, generating every
artifact) and the human handles judgment (naming,
organization, feature scope, simplification).

---

## The cost of change

Changing a business rule costs the same whether it is
day one or year three: update the spec, regenerate the
code. The accounting team that discovers an error in
reconciliation logic corrects the spec, and the code is
regenerated.

Changes cascade widely but mechanically. Changing a
fundamental type may touch dozens of files, but every
change at every cascade point requires no creative
judgment — it is mechanical regeneration. The blast
radius is large in files touched, small in decisions
needed.

---

## The spec as organizational asset

A spec tree that grows with the system is a complete,
versionable representation of organizational knowledge.
Every decision recorded. Every constraint traceable.
Every behavior attributable to a spec node with an
author and a version.

It is not documentation — documentation drifts. The spec
tree is the system; code is its shadow. It is not code —
code expresses mechanism. The spec tree expresses intent.

The asset compounds. A team that uses Code from Spec for
a year has a spec tree that reflects a year of learning.
The software becomes more correct not because the
engineers got better, but because the domain knowledge
got more explicit.

---

## Auditability

Every generated file carries an artifact tag linking it
to the spec that produced it. The chain hash is a
fingerprint of everything that contributed to the
generation. The git history of the spec directory is a
complete audit trail.

In regulated environments, this is compliance by
construction. An auditor traces any behavior back to a
spec node, to the person who authored it, to the version
that introduced it.

---

## Context management

AI agents have finite context windows. The spec tree
solves this by construction: each node's chain includes
only what it declared via `depends_on` and inheritance.
Adding hundreds of nodes to the tree does not inflate
the context for existing nodes.

The spec tree accumulates context across sessions. A
productive session produces spec changes. The next
session picks up the tree as it stands. Every decision
from every previous session is present, structured and
accessible.

The total knowledge in the tree is unbounded. The
context per generation is bounded and curated. This is
how Code from Spec scales beyond the limits of any
individual AI model.

---

## The transition

Trust is built through evidence, not optimism.

**Phase 1**: Humans review specs and generated code.
Every regeneration is inspected.

**Phase 2**: Humans review specs thoroughly but examine
generated code by sampling. Tests provide confidence.

**Phase 3**: Humans review specs only. Code is verified
by tests and CI.

**Phase 4**: Humans review specs only at the production
deployment boundary. Everything else is automated.

Each transition is earned by evidence. Trust can regress
— a serious bug should make the team return to a
previous phase.

---

## The endgame (a vision)

Today, repositories contain both specs and generated
code. This is a transitional state — teams need to
inspect generated output while trust is being built.

The logical conclusion: if code is derived from specs,
it does not need to be versioned. A repository that
contains only the spec tree and test specs. The CI
pipeline generates code, runs tests, and deploys. No
one reviews code. No one merges code. The entire team
works in specs — the artifact they all understand.

This vision is not yet realized. Every step toward it
— more precise specs, better test coverage, more
reliable agents — makes the methodology more valuable.

---

## Caveats

**AI is the weakest link.** Agents hallucinate, ignore
instructions, and rationalize skipping rules. The
structural guardrails exist precisely because the agent
cannot be trusted on its own. Trust the framework, not
the agent.

**The organizational shift is political.** The
methodology enables domain experts to contribute
directly. It does not cause them to. The cause is
leadership, training, and cultural investment.

**Not for everything.** Prototypes, throwaway code, and
trivial systems do not benefit. Code from Spec is
designed for systems where the cost of getting it wrong
exceeds the cost of specifying it precisely.

**Implicit knowledge is invisible knowledge.** Every
pattern, convention, or technique that should be followed
must be explicit in the spec tree. If it is not written
down, it will not be followed consistently. This is the
methodology's core cost — and its core value: knowledge
written once is knowledge that will never be lost.
