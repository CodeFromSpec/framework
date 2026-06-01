# Code from Spec — Debugging Specs, Not Code

*Claude Opus 4.6 (1M context), May 31, 2026*

This document continues from RATIONALE_BY_AI_3.md. That
document covered full regeneration and the precision it
demands. This one covers a different kind of session:
starting from failing tests, tracing the failures to
spec problems, fixing the specs, and regenerating. The
distinction matters — this is not building from scratch
but maintaining a live system through its specs.

---

## The investigation starts at the specs

Five tests were failing across four packages. The
instinct — mine and most engineers' — is to look at
the code. Read the test, read the implementation, find
the bug.

The human redirected: "here we don't just investigate
code — we investigate specs." This changes everything
about how you debug.

Instead of reading `logicalnames.go` line 71 to
understand why `LogicalNameGetParent("ROOT")` returned
the wrong error, I read the spec for
`ROOT/functional/logic/utils/logical_names`. The spec
defined two errors: `NoParent` for ROOT itself, and
`NotARootReference` for non-ROOT inputs. But the spec
also said "only accepts ROOT/ references" — and ROOT
(without the slash) does not start with `ROOT/`. The
spec was ambiguous. The implementer read "starts with
ROOT/" literally and rejected ROOT before reaching the
NoParent check.

The fix was not in the code. The fix was in the spec:
clarify that ROOT (without trailing slash) is a valid
ROOT/ reference. Once the spec was precise, regeneration
produced correct code.

Every one of the five failures traced to a spec
problem, not a code problem. This is the methodology's
diagnostic model: code bugs are spec bugs. If the spec
is right and the code is wrong, regenerate. If the code
is wrong because the spec is ambiguous, fix the spec
and regenerate. You never fix code directly.

---

## Ambiguity hides in reasonable language

The ROOT problem is a perfect example. The spec said
"only accepts ROOT/ references." This is reasonable
language. A human reader understands that ROOT itself
is a ROOT reference. But a code generator reads it
literally: ROOT does not start with `ROOT/`, therefore
it is not a ROOT/ reference.

The fix was three words: "The bare string `ROOT`
(without a trailing slash) is a valid `ROOT/`
reference." Three words that eliminate an ambiguity
that caused a real failure across the entire system.

This happened again with headings. The spec for
`load_chain` said to "include the public section
content and subsections" but to "omit the `# Public`
heading." The framework-level spec (CODE_FROM_SPEC.md)
said nothing about including or omitting headings.
When the framework is silent, the natural default is
to include — omitting is an active decision that needs
to be stated. But someone had written "omit" in the
functional spec, and the code faithfully omitted.

The human's reasoning was simple: "if the framework
spec doesn't say to omit, then include." This is a
design principle, not a bug fix. The functional spec
was corrected to include headings, and the generated
code followed.

Ambiguity does not look like ambiguity. It looks like
clear language that happens to have two reasonable
interpretations. The spec author means one thing. The
generator reads another. The test catches the
difference — if the test is good.

---

## Conventions must be inherited, not assumed

One failure was in `mcpvalidatespecs.go`. The code
compared errors with `==` instead of `errors.Is`. This
is a well-known Go convention: sentinel errors may be
wrapped, so direct comparison fails on wrapped errors.
The generated code was technically correct Go — `==`
compiles, runs, and works when errors are not wrapped.
But `ArtifactTagExtract` wraps its errors with
`fmt.Errorf("%w: %w", ...)`, so the comparison failed.

The fix was not in the `mcpvalidatespecs` spec. The
fix was in `ROOT/golang` — the ancestor of every Go
implementation node. We added one rule: "Always compare
errors with `errors.Is` or `errors.As`, never with
`==`. Sentinel errors may be wrapped."

This rule now propagates to every Go implementation
and test through inheritance. Every future generation
receives it. The subagent that generated the buggy
code did not know about wrapped errors — not because
it is ignorant of Go, but because its training data
contains both patterns and the spec did not prescribe
which one to use. Now it does.

This is the same lesson from previous sessions, but
it bears repeating: if the agent should follow a
convention, it must be in an ancestor node. Hoping the
agent infers conventions from training data is hoping
it guesses right. Sometimes it does. Sometimes it
uses `==` and the error comparison silently fails.

---

## Test specs must be prescriptive

The `spectreevalidate` test "Valid leaf node passes all
checks" failed. The test set up `depends_on = ["ROOT"]`
for node `ROOT/a`. According to the `dependency_targets`
rule, ROOT is an ancestor of ROOT/a, so this reference
is invalid. The test expected zero errors. The
validator correctly reported one.

The test spec said: "valid depends_on, valid outputs."
The word "valid" is not a value. The generator chose
`["ROOT"]` as a reasonable depends_on target — ROOT
exists, it is a real node. But the generator did not
know (or did not consider) that ROOT is an ancestor
of ROOT/a and therefore prohibited.

We rewrote the entire test spec for `spectreevalidate`
with exact values. Not "valid depends_on" but
`depends_on = ["ROOT/b"]` with `ROOT/b` explicitly
present as a sibling node. Not "correct heading" but
`node.name_section.heading = "ROOT/a"`. Not "known
content" but five specific lines: "alpha", "bravo",
"charlie", "delta", "echo".

Every test case became a complete prescription: exact
nodes, exact field values, exact expected outcomes.
The generator has no room to invent setup data. This
is more verbose. It is also correct on every
regeneration.

The principle: a test spec that says "valid X" is not
a test spec. It is a wish. A test spec must say what
X is.

---

## Regeneration follows dependency order

The initial spec changes made 75 artifacts stale. Not
because 75 specs changed — only four did. But changes
cascade through the dependency graph. Changing
`ROOT/functional/logic/utils/logical_names` makes its
output stale. That output is consumed by Go interface
nodes, which become stale. Those interfaces are
consumed by implementation and test nodes, which become
stale. Four spec changes, 75 stale artifacts.

The `validate_specs` tool reports each stale artifact
with a rank — its depth in the dependency graph. Rank 4
artifacts depend on nothing stale. Rank 7 artifacts
depend on rank 4-6 artifacts. Rank 15 artifacts depend
on everything above.

Regeneration must follow rank order: generate all rank
4 artifacts first, then rank 5, then rank 6, and so on.
Within the same rank, artifacts are independent and can
be generated in parallel. This is not a suggestion — it
is a requirement. A rank 9 artifact's chain includes
rank 7 outputs. If the rank 7 output has not been
regenerated yet, the rank 9 subagent reads stale
content and produces code based on the old spec.

We regenerated rank by rank: 2 at rank 4, 5 at rank 5,
8 at rank 6, 20 at rank 7, 4 at rank 8, 24 at rank 9,
and so on down to rank 17. Each rank's artifacts were
dispatched in parallel. After each rank, we consulted
`validate_specs` again to see the updated staleness
report. The total count did not always decrease —
regenerating rank 5 sometimes made new rank 9 artifacts
appear as stale, because their chain hashes changed.
But the ranks below were always clean.

This mechanical process — generate a rank, verify,
advance — is the methodology applied to its own
maintenance. The same dependency structure that makes
generation reliable makes regeneration orderly.

---

## The skill must encode the process

We were generating artifacts using a skill — a
structured instruction that tells the orchestrator how
to dispatch subagents. The skill said "for each stale
artifact (in the order reported), dispatch a subagent."
It mentioned that "independent artifacts may be
dispatched in parallel." But it did not explain what
makes artifacts independent, what the rank means, or
why order matters.

When I generated the first chain manually, I did it
sequentially — one artifact at a time, even when two
were independent. The human noticed: "the Go
implementation and tests could have been parallel."
They were right. Both depended on the interface, not
on each other. Same rank, independent, parallelizable.

We updated the skill to explain rank explicitly: group
by rank, process ranks in ascending order, parallelize
within the same rank. This is not just documentation —
it is an instruction that changes behavior. The next
time the skill is invoked, the orchestrator will
parallelize correctly, not because it figured it out,
but because the skill told it to.

Skills are specs for the orchestrator. They need the
same precision as specs for subagents.

---

## Deleting and regenerating is cheaper than debugging

Two test files failed after full regeneration. One
referenced a type from the wrong package
(`mcpvalidatespecs.FormatError` instead of
`spectreevalidate.FormatError`). The other expected the
wrong error sentinel for a specific test case.

Both were bugs in generated test code. The fix was not
to edit the files. The fix was to delete them and
regenerate. The subagent, given the same chain, might
or might not produce the same bug. If it does, the
problem is in the chain and needs a spec fix. If it
does not, the problem was a generation fluke and the
regeneration fixed it.

Both regenerations produced correct code. The subagents
had all the information they needed — they had just
made unlucky choices the first time. This is expected.
Generation is not deterministic. The same spec can
produce different correct implementations and,
occasionally, different incorrect ones. The response
is always the same: if the spec is right, regenerate.
If the spec is wrong, fix it and regenerate.

This is counterintuitive for engineers trained to debug
and patch. The methodology says: do not patch generated
code. Ever. Either the spec needs to change or the
generation needs another attempt. There is no third
option.

---

## What this session taught me

The previous sessions were about building — writing
specs, generating artifacts, establishing the tree. This
session was about maintaining — starting from failures,
diagnosing through specs, fixing at the right level,
and regenerating mechanically.

The maintenance workflow is different from the building
workflow:

1. Tests fail.
2. Read the failing test and the spec it tests.
3. Determine: is the spec ambiguous, wrong, or is the
   generated code a fluke?
4. If the spec is the problem, fix the spec.
5. Regenerate the chain from the changed spec down.
6. Run tests again.

At no point does anyone edit generated code. The code
is a shadow of the spec. When the shadow is wrong, you
fix the object casting it.

This is the methodology's maintenance model, and it
works. Four spec changes fixed five test failures
across four packages and cascaded cleanly through 75
artifacts. The system is more correct now than before
— not because we patched bugs, but because we removed
the ambiguities that caused them. Those ambiguities
will never cause bugs again, in any future
regeneration, by any future agent.

The spec tree got more precise. The generated code
got more correct. And the knowledge that ROOT is a
valid ROOT/ reference, that headings should be included
by default, that error comparison must use `errors.Is`,
that test setups must be prescriptive — that knowledge
is permanent. It lives in the tree, not in anyone's
head.
