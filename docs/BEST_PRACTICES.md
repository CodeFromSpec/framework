# Best Practices

Lessons learned from using Code from Spec in practice. These are
not rules — the methodology works without them. They are patterns
that reduce friction and avoid common pitfalls.

---

## Diagnose before fixing

### The problem

A failing test is a disagreement between two generated
readings — the implementation and the test — and either
side can be the wrong one. Each root cause takes a
different repair. A fix applied without knowing the case
tends to reproduce the bug — or to damage the spec: a
clause added to chase a bug the spec never had teaches
nothing and clutters every future generation.

### The practice

1. **Read the failure.** What specifically went wrong? A
   failing assertion, a panic, a compilation error, a bug
   report from use?

2. **Read the generated artifact.** Find the line or logic
   that caused it. Understand what it is doing and why it
   is wrong.

3. **Trace back to the specs — on both sides.** Read the
   spec that produced the artifact and the test spec
   behind the failing test, with the context they inherit.
   Decide which side is wrong before fixing anything. The
   diagnosis lands in one of four cases.

4. **Pin the bug with a test spec.** Before repairing
   anything, write (or update) a test spec that captures
   the correct behavior — the behavior the bug violates.
   Regenerate the test and confirm it fails against the
   current artifact. A red test before the fix proves the
   bug is real and detectable; a green test after the fix
   proves the repair worked. Without this step, you are
   trusting that the fix is correct without a mechanical
   check — the same gap that let the bug in. If the
   diagnosis shows the test spec is the wrong side (case
   one below), this step does not apply — you are
   correcting the test, not adding one.

5. **Repair according to the case.**

   **The test is wrong.** The implementation may be fine.
   If the generated assertion does not follow from its
   test spec, regenerate the test. If the test spec itself
   is wrong — it pins an accident of a previous artifact,
   or an expectation that intent has moved past — correct
   the test spec. Never edit the generated test file, and
   never weaken a test spec just to make the suite pass.

   **The spec never ruled on it.** The generator resolved a
   silence — and resolved it against what you wanted (a
   spec that never said which function to use; generated
   code that picked a platform-dependent one). The repair
   is a decision: state the missing rule in the spec, and
   pin it with a test spec so it is checked mechanically
   from now on (see [TESTING.md](TESTING.md)).

   **The spec ruled and the artifact disobeyed.** The
   repair depends on where the failure was caught. A test
   caught it: nothing to fix — the test did its job;
   regenerate. The same miss returns across regenerations:
   the clause is not landing with this generator — reword
   how the spec says it; the decision itself is fine. It
   slipped past the tests and was found in use: the gap is
   in the test suite — add the test spec that would have
   caught it.

   **The spec ruled wrong.** The artifact obeys the spec
   and the behavior is still wrong — the spec does not say
   what the project needs. No test derived from the spec
   can catch this; the verdict comes from people or
   production. Correct the spec, and review the test specs
   that inherited the error.

6. **Regenerate.** With the repair in place, regeneration
   is targeted rather than hopeful.

### The principle

The diagnosis decides where the repair goes — the test,
the spec, or nowhere but a regeneration. Skipping the
diagnosis means picking the repair blind.

---

## Use external imports intentionally

### The problem

A node needs context from a file outside the spec tree — an API
specification, a database schema, legacy source code. The tempting
approach is to import the entire file via `EXTERNAL/`, regardless
of size. This works but wastes context window and can introduce
noise that causes the generation subagent to hallucinate or focus
on irrelevant details.

### The practice

**Small files — import whole.** If the file is short and
entirely relevant (a `.proto` definition, a JSON contract, a
short config file), import it directly:

```yaml
depends_on:
  - EXTERNAL/proto/payments/v1/transfers.proto
```

**Large files — extract via an intermediate artifact.** When
only part of a large file matters, create a dedicated leaf
node that consumes the whole file via `input: EXTERNAL/` and
generates an intermediate artifact containing only the
relevant extract. The downstream node then consumes that
artifact via `depends_on: ARTIFACT/`. See
[LAYERS.md](LAYERS.md) for the extraction layer pattern.

This keeps the large file out of the downstream chain and lets
the extraction subagent — guided by the node's `# Agent`
section — decide what is relevant.

### The principle

`EXTERNAL/` brings the outside world into the chain. The less
you import, the more focused the generation subagent's context
is. When in doubt, prefer a small intermediate extraction over
a large direct import.

---

## Prefer qualified depends_on

### The problem

When a node declares `depends_on: SPEC/x/y`, the entire
`# Public` section of `SPEC/x/y` participates in the chain
hash. Any change to any part of that section — a new
subsection, a reworded paragraph, a renamed type — makes
every dependent node stale, even if the change is irrelevant
to what the dependent actually uses.

### The practice

When a node only needs a specific subsection, use a qualified
reference:

```yaml
depends_on:
  - SPEC/x/y(interface)
```

This imports only the `## Interface` subsection. Changes to
other subsections (`## Context`, `## Constraints`) do not
cause this node to become stale.

Use unqualified references only when the node genuinely needs
everything in `# Public` — for example, when inheriting all
constraints from a sibling branch.

### The principle

Qualified dependencies reduce the blast radius of changes.
The narrower the dependency, the fewer unnecessary
regenerations.

---

## Route dependencies through the interface

### The problem

When module A consumes module B, the tempting dependency is
B's implementation artifact. Every internal change to B then
makes A stale, even when B's contract did not move — and B's
full source enters A's chain, so A's generation subagent
anchors on internals A was never meant to depend on.

### The practice

Give the module an explicit contract — an authored
`## Interface` subsection imported via a qualified
reference, or a generated interface artifact — and make
every consumer depend on it alone. Prefer the most
mechanically checkable form the language offers. See
[DECOMPOSITION.md](DECOMPOSITION.md) for the forms, the
oracle regimes, and per-language guidance.

### The principle

Consumers depend on the contract, never the implementation.
Staleness propagates at contract granularity, and what the
interface does not state stays free to change.

---

## Harvest shared helpers, do not predict them

### The problem

With one node per file, shared helpers raise a question:
how does a generation subagent know which helpers it may
call? Confinement means an agent sees only its chain — it
cannot browse the package for what exists. The tempting
fix is to author the helper nodes up front, but that
inverts their nature: a helper is a resolution, something
draws reveal a need for, not a design decision the author
can predict on day zero.

### The practice

Treat shared helpers as a lifecycle:

1. **First draws self-contain.** Each node's generation
   gets by on its own — local helpers, inline logic.
   Duplication across sibling files appears. That is the
   cost of independent draws, and it is acceptable.

2. **Harvest.** The author observes the repetition — three
   files carrying the same fetch-with-lock sequence — and
   ratifies it: a helper node is born, its spec written
   from what the draws revealed necessary, its signatures
   stated in `# Public ## Interface`.

3. **Steady state.** The helper is now available
   vocabulary. A new node whose domain touches it declares
   `depends_on: SPEC/…/utils_x(interface)` — not
   predicting helpers, importing vocabulary that already
   exists. The agent decides whether and how to use what
   its chain offers; what it cannot do is depend on what
   it cannot see.

4. **Deduplicate gradually.** The nodes that duplicated
   the logic gain the `depends_on` and regenerate when it
   is worth the draw — not all at once, not urgently.

### The principle

A shared helper is a ratified discovery, not a predicted
design: the helper node is the project's memory of what
the draws taught. Confinement is what makes that memory
reliable — an agent can only call what its chain shows, so
every shared use is a declared, staleness-tracked
dependency instead of an untracked coupling.

---

## Place conventions at the right level

### The problem

A convention placed too high in the tree pollutes every
descendant's chain — even those where it is irrelevant. A
convention placed too low gets repeated across siblings and
risks drifting between copies.

### The practice

**Place each convention at the lowest ancestor that governs
all the leaves that need it.** Go conventions belong at the
top of the implementation subtree, not at the root. Test
patterns belong at the top of the tests subtree. The root
node is the right place only for conventions that genuinely
apply to every leaf in the project (e.g., "artifacts carry
no comments," or a project-wide glossary).

### The principle

A convention's position in the tree determines its blast
radius. The narrower the scope, the fewer unnecessary
regenerations when it changes — and the more focused each
leaf's chain stays.

---

## Start every session with the methodology

### The problem

An agent that works on a Code from Spec project without the
methodology in context will improvise: edit generated files
directly, skip staleness checks, invent conventions. The rules
only govern the session if they are actually loaded — assuming
the agent will fetch or remember them is unreliable.

### The practice

Run `/cfs-init-session` at the start of every session. It reads
the pinned copy of `CODE_FROM_SPEC.md` bundled with the skill
(installed by `cfs-init-repo`) and loads the working guidelines.

If context gets cluttered during a long session, clear and
re-initialize:

```
/clear
/cfs-init-session
```

### The principle

The methodology must be in context before any work begins —
loaded from the local pinned copy, with no network dependency
and no partial reads.

---

## Session lifecycle and cache

### The problem

The cache stores spec chain content from previous
generations. It only serves its purpose — showing the
subagent what changed — if it is rebuilt when the git
state moves, and it raises an obvious question over time:
when should it be pruned?

### The practice

Treat `cfs-init-session` as the boundary between tasks.
The recommended workflow:

1. **Start of task**: `cfs-init-session` — reconstructs
   the cache from the current state. The git state is
   stable (fresh clone, post-merge, or start of new work).
2. **Work**: edit specs, regenerate, test. The cache
   grows as new chains are computed.
3. **End of task**: PR, merge.
4. **Next task**: `cfs-init-session` again.

Pruning is always manual — no step of the workflow prunes
for you. Cache space is cheap, and old entries are not
dead weight: a `git restore` or a reverted merge can make
an old chain current again, and a cache that still holds
its content spares the next generation from running
without history. Run `prune_cache` only when the cache's
size actually becomes a problem.

### The principle

The cache is rebuilt at task boundaries and grows during
work. Old entries cost disk and buy history. Prune by
need, never by schedule.

---

## Names are specs

### The problem

A generation subagent reads every identifier in the chain
as a declaration of intent. A parameter named `sessionHash`
tells the subagent the value is already hashed; a parameter
named `sessionID` tells it the value is a raw identifier
that may need processing. The name decides behavior — no
instruction in `# Agent` compensates for a name that says
the wrong thing.

This applies to every name in the spec: function names,
parameter names, field names, type names, error sentinel
names. Each one is read by the subagent as meaning exactly
what it says.

### The practice

Name everything for what it **is**, not for what will be
done to it or what it came from. A raw session identifier
is `sessionID`, not `sessionHash` (which says it is already
hashed) and not `sessionToken` (which says what carried it).

When reviewing a spec, read the names as a subagent would —
literally, without the context you carry. If a name
suggests a different meaning than the one intended, the
subagent will follow the name.

### The principle

A name in the spec is a contract with the subagent. It
will be taken at face value. A misleading name produces
correct-looking code that does the wrong thing, and no
amount of prose in `# Agent` overrides what the name
already said.

A name that is precise enough for a subagent is precise
enough for a human — both read it literally and expect it
to mean what it says. Getting names right for generation
gets them right for everyone who reads the code afterwards.
