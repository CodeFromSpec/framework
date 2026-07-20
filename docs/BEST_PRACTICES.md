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

4. **Repair according to the case.**

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

5. **Regenerate.** With the repair in place, regeneration
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
