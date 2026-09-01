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

## Write fixes as decisions, not corrections

### The problem

A fix is written with the defect in view: the old code,
the failing test, the whole session that diagnosed it.
Written from that position, the natural wording is
oppositional — "do not use X", "instead of the previous
approach", "no longer does Y". Every future reader of
the spec — a generation subagent months from now, a
teammate, a new model — has none of that context. A
clause worded against a state that no longer exists is
unintelligible to them: it forbids something they cannot
see, opposes an approach that is nowhere in the chain.

AI assistance makes this failure mode the default, not
the exception. An AI editing a spec writes in opposition
to what it sees, and what it sees — the artifact, the
diagnosis, the conversation — is not in the spec.

There is a deeper cost. Each such clause is written in
the coordinate system of the artifact that happened to
be generated. A specification defines a region; the
artifact is one draw from it. Accumulate corrections
against that one draw, and the spec stops encoding
intent and starts encoding the accidents of the draw —
scar tissue around a particular artifact. A future
regeneration, or a different generator, inherits
constraints that intent never demanded.

### The practice

**Word every fix as a statement of what must be true**,
in the domain's terms. "Monetary amounts are integers in
cents" — not "replace the float from the previous
version". The clause must be self-contained: readable
by someone who never saw the bug, the old artifact, or
the session that fixed it.

**Apply the resample test before adding the clause**:
would this clause still be wanted if the artifact were
regenerated from scratch and had never had this bug?

- *Yes* — it is a real decision, discovered through the
  bug but independent of it. State it positively and
  keep it.
- *No, it only makes sense against the current
  artifact* — it does not belong in `# Public` or
  `# Agent`. Record the incident under `## Decisions`
  in `# Private`, where it informs humans without
  entering any chain.

**Record provenance.** When a clause is born from an
incident, add a line under `## Decisions` in `# Private`
saying which incident produced it. The clause alone
cannot tell a future reviewer whether it encodes intent
or a scar; the provenance can. `# Private` enters no
chain and no hash, so this costs nothing in
regenerations.

A warning sign to watch for in review: a spec
accumulating specific prohibitions that only make sense
to someone who saw yesterday's artifact. Intent tends to
be stated affirmatively; scars inherit the shape of the
defect they corrected.

### The principle

The spec accumulates the destination and the rules of
the road; course corrections are consumables of the
loop, not heritage. A clause earns its place by being
true under any artifact intent would accept — not by
having fixed the one that existed when it was written.

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
imports:
  - EXTERNAL/proto/payments/v1/transfers.proto
```

**Large files — extract via an intermediate artifact.** When
only part of a large file matters, create a dedicated leaf
node that consumes the whole file via `input: EXTERNAL/` and
generates an intermediate artifact containing only the
relevant extract. The downstream node then consumes that
artifact via `imports: ARTIFACT/`. See
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

## Prefer qualified imports

### The problem

When a node declares `imports: SPEC/x/y`, the entire
`# Public` section of `SPEC/x/y` participates in the chain
hash. Any change to any part of that section — a new
subsection, a reworded paragraph, a renamed type — makes
every dependent node stale, even if the change is irrelevant
to what the dependent actually uses.

### The practice

When a node only needs a specific subsection, use a qualified
reference:

```yaml
imports:
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
   `imports: SPEC/…/utils_x(interface)` — not
   predicting helpers, importing vocabulary that already
   exists. The agent decides whether and how to use what
   its chain offers; what it cannot do is depend on what
   it cannot see.

4. **Deduplicate gradually.** The nodes that duplicated
   the logic gain the `imports` entry and regenerate when it
   is worth the draw — not all at once, not urgently.

### The principle

A shared helper is a ratified discovery, not a predicted
design: the helper node is the project's memory of what
the draws taught. Confinement is what makes that memory
reliable — an agent can only call what its chain shows, so
every shared use is a declared, staleness-tracked
dependency instead of an untracked coupling.

---

## Move in small steps

### The problem

It is tempting to batch spec work: author the new module,
the signature change, the tests, and the consumers all at
once, then regenerate everything in one pass. Generation
handles this fine — the agents do not care how much changed.

The person reviewing the result does. Every regenerated
line still has to be read by someone, and reading is where
the real cost of this methodology lives. Generating code is
fast and cheap; a human reading a diff is slow and
expensive. A big batch produces one huge diff, and a subtle
mistake buried in a thousand regenerated lines looks just
as plausible as its correct neighbors. The bigger the diff,
the worse the reading — and the reading is what you are
actually paying for.

### The practice

Change one thing at a time. Edit the spec for a single
decision, regenerate, read the diff, build and test — then
move to the next decision.

Let the staleness cascade work for you instead of trying to
anticipate it. When a change ripples through dependents,
regenerate rank by rank and read as you go. Each diff stays
small enough to answer at a glance: "the only change here
is the renamed parameter." That is a review you can
actually trust.

Working this way has two side benefits:

- **Gaps show up at the right moment.** You read each
  dependent's spec just before regenerating it — right next
  to the change that made it stale. A stale assumption in
  some forgotten consumer is easy to catch there, and easy
  to miss in one big up-front sweep.

- **Failures point at their cause.** With one decision per
  step, a failing build or test points directly at the
  change that broke it. Nothing to bisect. In a big batch,
  the same failure points at the whole lot, and now you are
  investigating instead of fixing.

Batching within a rank is still fine: artifacts of equal
rank are independent, and reading three sibling diffs
together costs little. What you want to avoid is stacking
several *decisions* into one regeneration — that is what
turns a glance into an investigation.

One honest caveat: this practice matters in proportion to
how much of the checking is human reading. A component
whose behavior is fully pinned by exhaustive tests gets its
confidence from the suite, not from the diff — batching
there is cheap. Everywhere else, the reading is the check,
and small steps are what keep it affordable.

### The principle

Generation is fast; review is scarce. A small diff is a
cheap reading, and the reading is the real bottleneck —
so shape the work around the reader. Each step small enough
to actually read, each mistake caught in the step that
introduced it. The sequence feels slower and finishes
faster than one big step followed by one big review.

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
