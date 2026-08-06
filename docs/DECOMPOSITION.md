# Decomposition

How to split software into modules and route dependencies
between them.

This document assumes familiarity with
[CODE_FROM_SPEC.md](../CODE_FROM_SPEC.md).

[LAYERS.md](LAYERS.md) is the companion document: layers cut
the pipeline vertically, between stages of refinement;
decomposition cuts the system horizontally, between modules.
Both use the same mechanism — an artifact acting as a context
firewall — on orthogonal axes.

---

## Modules

A **module** is a subtree of spec nodes that produces one
component of the software: its implementation and its
contract. The subtree's intermediate nodes hold what the
module's leaves share; its leaves generate the artifacts.

A module boundary is real to the extent that dependencies
cross it through the contract rather than the
implementation. The rest of this document is about making
that hold.

---

## Route dependencies through the interface

When module A consumes module B, the tempting dependency is
`depends_on: ARTIFACT/…/b` — B's implementation artifact.
This has two costs:

- **Staleness at the wrong granularity.** Every internal
  change to B makes A stale, even when B's contract did not
  move. Regenerations cascade for changes that could not
  affect A.

- **Internals leak into A's chain.** A's generation
  subagent sees B's full source — and anchors on details A
  was never meant to depend on. That is Hyrum's Law
  reproduced inside the pipeline: what is visible gets
  depended on.

Instead, give the module an explicit contract — an
**interface** — and make every consumer depend on it alone:

- Internal changes to B leave the interface untouched, so
  consumers do not become stale. Staleness propagates at
  contract granularity.

- What the interface does not state is structurally
  invisible. B's internals can be regenerated or
  restructured freely, with no lock-in.

- The interface is the narrow, cheap place to spend human
  review: reading a contract takes minutes; reading a
  module does not.

The next two sections present the two decisions that shape
an interface: who decides its content, and what checks it.

---

## Who decides the interface

The interface itself is a piece of the software, and faces
the same question every piece faces: does it live at the
spec level or at the artifact level?

**Authored — a `## Interface` subsection.** The module's
root node carries the contract under `# Public`:

```markdown
# Public

## Interface

- `CreateTransfer(ctx, req) (*Transfer, error)` — ...
- `ErrInsufficientFunds` — returned when ...
```

The module's own leaves inherit it automatically.
Consumers in other branches import it with a qualified
reference:

```yaml
depends_on:
  - SPEC/implementation/transfers(interface)
```

The interface is a **decision**: the author determines
signatures, types, and error contracts, and the text aims
the generation of the implementation and of every consumer.
Changes to other subsections (`## Context`,
`## Constraints`) do not make consumers stale.

**Generated — an interface leaf node.** A dedicated leaf
node whose spec states what the module must expose
semantically, and whose generation resolves the exact
shape — names, idiomatic signatures. Consumers declare
`depends_on: ARTIFACT/…/interface`. The interface is a
**resolution**: the draw decides the form, and the artifact
records and stabilizes it across regenerations.

**Choosing.** Author the interface where the contract is
what is being decided: public APIs, seams between teams,
boundaries that external consumers will freeze anyway.
Generate it where the semantics matter and the form is
convention — internal modules whose exact shape the
generator resolves well.

---

## Who checks the interface

An interface only holds the boundary if something checks
that both sides conform to it. Languages offer three
regimes, in decreasing strength:

**Native compilable form.** The interface is generated as
real code — a C header, a Java `interface`, a TypeScript
`.d.ts`. The compiler becomes the seam's oracle: an
implementation that diverges does not build, a consumer
that diverges does not build. Conformance is mechanical,
free, on every draw.

**Optional checker.** The language has a native form whose
oracle is opt-in — Python (`.pyi` stubs or `Protocol`
checked by mypy/pyright), JavaScript (JSDoc typedefs
checked by `tsc` with `@ts-check`). Generate the native
form anyway; the pattern is a good reason to adopt the
checker.

**No native form.** The language has no separate interface
artifact (Go, Rust). The contract is an exact document —
the authored `## Interface` subsection, or a generated
`.md` — stating exported types, signatures, and error
contracts. No compiler checks it directly. The oracle is
indirect: implementation and consumers are generated from
the same text, so a divergence between the two readings
breaks the consumer's build.

The rule across all three: **generate the interface in the
most mechanically checkable form the language offers; an
exact document is what remains when it offers none.**

---

## Combining the two

Where the language has a native form, the strongest design
combines an authored contract with a generated native
interface: the decision lives in the spec, and the compiler
enforces it.

```
code-from-spec/
└── implementation/
    ├── _node.md            ← language, conventions
    ├── transfers/
    │   ├── _node.md        ← # Public ## Interface (authored)
    │   ├── interface/
    │   │   └── _node.md    ← output: include/transfers.h
    │   └── core/
    │       └── _node.md    ← output: src/transfers.c
    │                          depends_on:
    │                            ARTIFACT/implementation/transfers/interface
    └── reports/
        └── _node.md        ← output: src/reports.c
                               depends_on:
                                 ARTIFACT/implementation/transfers/interface
```

The `transfers/_node.md` root holds the authored contract.
The `interface/` leaf inherits it and transcribes it into
the native form — a cheap, near-deterministic draw. The
implementation leaf and every consumer depend on the
resulting artifact: they receive the exact compiled truth,
and the `ARTIFACT/` reference orders generation correctly.

In a language with no native form, the transcription node
disappears: the authored `## Interface` already is the
exact document, and consumers use the qualified `SPEC/`
reference directly.

---

## Per-language guidance

Languages sit on a spectrum, by how much of the pattern
they build in:

1. **The language is the pattern.** OCaml (`.mli`), Ada
   (`.ads`/`.adb`): an authored interface file the compiler
   enforces, with invisibility by omission. C sits here by
   convention — the `.h` is the interface artifact, checked
   wherever it is included. These languages are evidence
   that the separation is sound: they made it a language
   design decision, validated over decades.

2. **Native construct, mandatory oracle.** Java, C#,
   Kotlin, Swift, TypeScript: `interface` types, `.d.ts`,
   protocols. Generate interface files; the compiler
   enforces both sides. C# adds a structural wall
   (`internal` scopes to the assembly); Kotlin's explicit
   API mode makes the compiler demand that the public
   surface be deliberate.

3. **Optional oracle.** Python (`.pyi`/`Protocol` + mypy or
   pyright), JavaScript (JSDoc + `@ts-check`). The native
   form exists; the check is opt-in. Adopt it.

4. **No separate form.** Go, Rust: the exported surface is
   implicit in the source. Use an exact contract document
   and the two-readings oracle.

One caveat rather than a group: **C++**. The header looks
like C's, but templates and inline code force
implementation into it — the interface artifact carries
internals by necessity, and every consumer's chain receives
them. The seam is structurally wide in template-heavy code.
C++20 modules restore the separation where they can be
adopted.

---

## Boundaries need an enforcer

A module boundary is real only if something in the
software enforces it. Several leaves generating files into
the same Go package are separated in the spec tree and
united in the namespace: every file sees every other
file's unexported identifiers, so one draw can call a
helper another draw generated — coupling no spec declared
and no staleness tracks. The seam exists in the tree and
not in the software.

When a boundary matters, make it coincide with one the
language enforces — in Go, the package. A library of heavy
routines becomes one module per routine, each its own
package, with a facade package as the public surface and
the sub-packages under `internal/` so external consumers
cannot reach them: the internal decomposition is free to
change because nothing outside can depend on it. Domain
types shared between the routines get a small module of
their own that the others import — they cannot live in the
facade without creating an import cycle.

---

## Granularity within a module

Below the module level, the same question recurs: do the
module's functions share one leaf, or does each get its
own? The criterion is the module criterion one level down.

**Functions that share private representation** — internal
types, helpers, invariants — are one region. Keep them in
one leaf. Split across leaves, the shared dimensions cross
the seams: helpers duplicated or drifting between draws,
surface coherence that is nobody's job.

**Functions related only through the module's surface**
can each be a leaf under the module's root node. Each
inherits the shared contract and conventions, generates
its own file, and regenerates alone when its own spec
changes.

Within a module these leaf seams do not need an enforcer —
the files share the package namespace, and that is
tolerable exactly because the module is one region of
trust, covered as a whole by its tests. But the softness
has a consequence: if one leaf's generated code comes to
lean on another's unexported helper, regenerating one can
break the other without making it stale. The module's test
suite, which runs whichever file changed, is what catches
it.

Helpers shared between sibling leaves get leaf nodes of
their own, with their signatures stated in
`# Public ## Interface` and imported by qualified
`depends_on` — but they are harvested from observed
duplication, never designed up front. See "Harvest shared
helpers, do not predict them" in
[BEST_PRACTICES.md](BEST_PRACTICES.md).

A large leaf is also cheaper than it looks. Regeneration
is not a blind rewrite: with the cache available, the
chain marks exactly what changed, the existing artifact
anchors everything else, and the diff stays the size of
the change. When in doubt, start coarse and split when it
hurts — splitting a grown node later costs a spec
refactor, while merging leaves that were cut too fine
costs more, because the inconsistency between their draws
has already been generated and tested. When splitting,
move the test specs with the logic — see "Tests migrate
with the logic" in [TESTING.md](TESTING.md).

---

## The spec as a boundary detector

Node granularity is not file granularity. The node is the
unit of the draw — its content enters every chain — of
staleness, and of inheritance. And description granularity
is coupled to generation granularity: only leaves
generate, so the only way to give a routine its own spec
is to give it its own leaf. Splitting the description is
splitting the draw.

That makes legibility pressure on a spec a legitimate
decomposition signal — and the spec itself the cheapest
boundary detector available. Try to split the description:

- **The parts come out self-contained** — each routine
  describable without referencing the others' internals.
  That is evidence of a real seam the description was
  hiding. Cut there.

- **The parts would reference each other's internals
  constantly.** The length is honest: the region is
  genuinely large, and splitting it would draw a false
  boundary in the spec before drawing one in the code.
  Organize the node with `##` subsections, or compress
  upstream context through a layer (see
  [LAYERS.md](LAYERS.md)), and keep one leaf.

This is the horizontal analogue of the layer-boundary test
in LAYERS.md: there, a downstream layer reaching back for
upstream context reveals a wrong boundary; here, a
description resisting the cut reveals the same thing.

---

## One node, one file

Each leaf generates at most one artifact, at a path the
spec declares. The constraint is deliberate. The obvious
objections were weighed, and each dissolved on
examination:

- **"The generator should decide the file layout."** File
  boundaries are load-bearing far more often than they
  look. In Go, a file's name can carry build semantics
  (`_test.go`, platform suffixes) and a directory changes
  visibility (`internal/`). In Python and JavaScript, the
  import path is public API — a consumer can see it, so
  Hyrum's Law applies to it. In Java, directory and file
  name are the package and the class. A dimension that is
  this often part of the contract belongs to a decision
  recorded in the spec tree — versioned, reviewable,
  stable across draws — not to a resolution recorded only
  in a manifest.

- **"A single file grows too large to read."** If the
  description splits cleanly, split the node — the
  boundary detector above at work — and the node split
  buys what a file split cannot: declared dependencies,
  staleness at the right grain, helpers that can be
  harvested. If the description resists the split, the
  region is genuinely one, and splitting the file would
  scatter it without shrinking it: the reader still reads
  all of it, in more places.

- **"Some languages force multiple files per unit."**
  Fewer than they appear to. Java forces a file only per
  *public* top-level class — and a public class is
  surface, an authored decision that deserves its own
  node anyway; internal classes can share its file as
  package-private or nested types. Go's platform variants
  (`x_linux.go`, `x_windows.go`) are separately
  describable by construction — one node per platform is
  a clean split, and supporting a platform is a decision,
  not a resolution.

What the constraint buys in exchange: the output
namespace is static. Every artifact has one declared
path, so collisions between nodes are detectable by
validation before anything generates, ownership of every
file is unambiguous, and the manifest stays one line per
artifact. Letting a draw choose file names would trade
all of that for a freedom the cases above show is rarely
free to give.

---

## Modules in the spec tree

A module's subtree lives in the implementation branch. Its
tests do not live inside it — they live in the mirrored
test subtree, following [TESTING.md](TESTING.md):

```
code-from-spec/golang/
├── implementation/
│   └── transfers/
│       ├── _node.md        ← # Public ## Interface
│       ├── interface/
│       │   └── _node.md
│       └── core/
│           └── _node.md
└── test/
    └── cases/
        └── transfers/
            └── _node.md    ← depends_on:
                                SPEC/golang/implementation/transfers(interface)
```

Keeping tests out of the module subtree is what the test
conventions require: test conventions live in the test
subtree's guard node, inherited by every test leaf — a
test nested inside the module would inherit the
implementation branch's craft instead. And the module
pattern hands the test node its ideal source: the
qualified reference delivers exactly the authored
contract, nothing of the mechanism.

Mirror the project's filesystem in the tree only where the
level carries a rule. `implementation/internal/` earns its
place if its node states the conventions of internal
packages — not importable from outside, designed for the
facade rather than for external consumers. A level with
nothing to say is a `_node.md` of pure ceremony, polluting
every descendant's chain: placement on disk is the
`output` path's job; the tree's structure is inheritance.

---

## Choosing per module

None of this is an allegiance. Each module faces the
decision once: is its interface authored or generated, and
in which form is it checked? The answer follows the same
economics as everything else — how settled the contract is,
how load-bearing the seam is, how well the generator
resolves the language's conventions. Re-ask the question
when those change.
