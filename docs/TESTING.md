# Testing

How to organize test specs so that generated code is checked
by generated tests — and why the way they are organized decides
whether a green suite means anything.

This document assumes familiarity with
[CODE_FROM_SPEC.md](../CODE_FROM_SPEC.md).

---

## Why tests carry more weight here

The methodology assumes an imperfect generation agent. An agent
can hallucinate behavior the spec never asked for, omit a step
the spec requires, misread an unambiguous sentence, or preserve
old behavior the spec has changed — and the code will compile
and the manifest will read clean. The chain hash answers one
question only: was this artifact produced from this
specification. Whether the artifact does what the specification
means is a separate question, and tests are the layer that
answers it.

In hand-written code, tests are a second line of defense behind
the author's judgment — the defensive check a developer adds
from experience protects behavior whether or not a test covers
it. Generated code has no author judgment in it. Every behavior
is protected by a test, or by a deterministic layer (types,
schema constraints, the compiler), or by nothing.

Among everything that checks generated code, the test suite
holds a unique position: it is behavioral where types are
structural, mechanical where review is human, specific to
this project where the compiler is generic — and it runs on
every generation. No other check has all four properties.

The framework does not mandate tests — nothing in the format
requires a node to have them, and projects differ in how much
verification they need. But the methodology's reliability story
depends on them, so the conventions below should be treated as
the default, and departures from them as decisions worth
recording.

---

## Tests are specifications

A test in Code from Spec is not a file someone writes next to
the generated code. It is a generated artifact like any other:
a leaf node whose spec describes behavior — given this
scenario, this expected result — and whose output is a test
file. Test specs are authored, reviewed, versioned, and
regenerated with the same machinery as everything else.

This is what makes fixes durable. When a bug is found, the
correction goes into the spec, and a test spec pins the
expected behavior. A patch in a generated file survives until
the next regeneration; a rule in the spec tree survives every
regeneration.

The same discipline applies in reverse: never fix a failing
test by editing the generated test file, and never weaken a
test spec to make an implementation pass. Diagnose first — see
"Diagnose before fixing" in
[BEST_PRACTICES.md](BEST_PRACTICES.md).

---

## Contract in Public, mechanism in Agent

The format splits a node's content into what other nodes may
see and what only its own generation sees, and the testing
conventions lean on that split.

An implementation node's `# Public` carries the contract:
package name, import path, interface signatures, error values
— what consumers may rely on. Its `# Agent` carries the
mechanism: the step-by-step logic, the algorithm choices, the
implementation guidance. The format makes `# Agent` invisible
to every other node — it is not inherited and cannot be
imported. Anything placed in `# Public`, by contrast, is one
`depends_on` away from any node in the tree.

The convention that follows: **keep `# Public` contract-only.**
Implementation detail in a public section leaks — it becomes
visible to the tests (and to everything else), and it makes
dependents stale on changes that should not concern them.

## The node pair

Give each component two leaf nodes — an implementation node
and a test node — with the test subtree mirroring the
implementation subtree, so the pairing is visible by
inspection: an implementation leaf without a test sibling is a
gap you can see.

```
code-from-spec/golang/
├── implementation/
│   └── fees/
│       └── _node.md      ← # Public: contract · # Agent: logic
│                           output: internal/fees/fees.go
└── test/
    ├── cases/
    │   └── fees/
    │       └── _node.md  ← depends_on the implementation node
    │                       output: internal/fees/fees_test.go
    └── utils/
        └── ...           ← shared test helpers, themselves
                            generated artifacts
```

The test node declares `depends_on` to the implementation node
— which delivers only its `# Public`, the contract — and to
the test-utility nodes it uses. Shared helpers (temp-dir
setup, fixture builders) are their own generated artifacts
under `test/utils/`; the test subtree's **guard node** — a
common ancestor that guards a rule: every leaf under it
inherits the rule and must follow it — should require their
use rather than let each test generation reinvent them.

Place test conventions — framework choices, naming patterns,
setup requirements, how expected values are stated — in the
guard node, so every test leaf inherits them. A convention
that lives there is applied by every test generation,
including tests written months later.

In the test node's `# Agent`, state cases as behavior: a
setup, an action, an expected result, in the domain's terms.

```markdown
#### Rejects a zero amount

Setup:
- No fixtures required.

Actions:
1. Call `CalculateFee(0, false)`.

Expected:
- Returns a validation error.
- Returned fee is 0.
```

---

## Independence, by structure

A passing test is evidence only if the test is an independent
opinion. In ordinary development the independence comes from a
human: the author wrote the code, and a separate act of
judgment wrote the test. Under generation, implementation and
test can come from the same model reading the same spec — and
if the test spec is a paraphrase of the implementation spec,
the two can share the same mistake and agree on it. The suite
is green; the behavior is wrong.

The format guarantees the strongest half of this by itself. A
test node that depends on the implementation node receives
only its `# Public` — the contract. The implementation's
`# Agent`, where the mechanism lives, is not importable by
anything, and its generated source file is not in the test's
chain. The test generation subagent sees what the component
promises and what behavior is expected of it, and cannot see
how the implementation chose to deliver it. The two artifacts
become two independent readings of the same intent, which is
exactly what a second opinion is.

Three conventions complete what the format starts:

**Keep `# Public` contract-only.** The format hides `# Agent`;
it cannot hide what you choose to put in `# Public`. Mechanism
placed there is mechanism the test generation will see.

**Never point a test node at the implementation's artifact.**
`depends_on: ARTIFACT/` of the unit under test would deliver
the generated source into the test's chain, reopening the
correlation channel the structure closed.

**Author test specs as their own account of behavior.** State
scenarios and expected results in the domain's terms — "a
request with amount zero returns a validation error" — not as
a re-narration of the implementation's mechanism. If the
implementation spec changes how, the test spec should not need
to change; it only changes when what is correct changes.

The same principle can extend to the language level: the test
guard node can require black-box testing — in Go, an external
test package that imports the package under test — so that
even the compiled test exercises only the public API.

All of this is checkable by reading the spec tree: if a test
node's chain contains the implementation's mechanism, the
independence is compromised — whether or not anything fails.

One limit no structure closes: implementation and test are
usually generated by models that share most of their
training, and two readers with the same habits can resolve
the same silence the same wrong way — and agree. A green
suite is evidence only on the dimensions where the two
readings could actually diverge. Where both inherited the
same blind spot, green means nothing; that residue is what
human review and real use are for.

---

## No retry loop

The generation subagent never sees a test fail. It
generates from the chain and finishes; tests run later,
and their verdicts return to the human and the
orchestrator — never raw to the subagent as "make this
pass."

This is deliberate. A generator iterating against a fixed
test suite converges on whatever makes the suite green —
hardcoding the expected output is the classic move — and
green-by-search is not correctness. Keeping the subagent
away from the verdict costs convergence speed and buys
something worth more: nothing in the loop is optimizing
against the tests. When a test fails, the fix enters
through the spec — diagnose, correct the spec or the test
spec, regenerate.

---

## What must be tested

Coverage under generation has a specific shape, because the
deterministic layers already hold part of the surface.

Types, schema constraints, and the compiler enforce what they
enforce identically on every generation, with no test
authored: a `CHECK` constraint rejects the invalid row whether
or not the agent remembered the rule; a strong type makes a
category of error unrepresentable. Where an invariant can live
in a type or a constraint, put it there — that is cheaper and
stronger than a test.

Test specs are mandatory exactly where those layers are
silent: meaning carried in prose and convention. A date format
in a `TEXT` column, a status transition rule, a rounding
convention, a business rule that distinguishes two account
kinds — to the compiler these are all just strings and
integers. Nothing deterministic watches them. If no test spec
pins them, they are re-decided by the model on every
regeneration, silently.

Those seams — every place where a convention carries meaning
that no type enforces — are enumerable. Walking the spec tree
and counting them is the honest way to size the test effort a
system actually requires.

---

## Tests migrate with the logic

Test specs are the project's acquired memory: each one exists
because some behavior mattered enough to pin. That memory is
lost in reorganizations unless it is moved deliberately.

When specs are refactored — nodes moved, modules merged or
split — the test specs covering the affected behavior move
with it. The working rule is mechanical: when logic moves, its
tests move; when modules merge, their tests merge; the count
of behavioral assertions after a refactor is at least the
count before. A behavior that had a test in the old structure
and lacks one in the new structure is a regression waiting for
a future generation to introduce it.

---

## Future direction

One of the independence rules ("a test node never depends on
the artifact of the unit it tests") is a property of the
dependency graph, which means tooling could verify it
mechanically rather than by review — and a declared pairing
between implementation and test nodes would let
`validate_specs` report artifacts with no test coverage at
all. Neither exists today; both fit the framework's principle
of enforcing by structure what would otherwise be asked of
discipline.
