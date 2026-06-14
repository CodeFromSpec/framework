# Best Practices

Lessons learned from using Code from Spec in practice. These are
not rules — the methodology works without them. They are patterns
that reduce friction and avoid common pitfalls.

---

## Diagnose before regenerating

### The problem

When generated artifacts fail tests, the instinct is to regenerate
immediately — fix the spec, dispatch the subagent, hope it works
this time. This often produces the same bug or a different one,
because the root cause was never understood.

The spec might be correct and the subagent might have made a
reasonable but wrong implementation choice. Regenerating from the
same spec can produce the same wrong choice, or a different wrong
choice, burning tokens without progress.

### The practice

When tests fail after artifact generation, stop and diagnose:

1. **Read the failing test output.** What specifically failed?
   An assertion, a panic, a compilation error?

2. **Read the generated artifact.** Find the line or logic that
   caused the failure. Understand what it is doing and why it's
   wrong.

3. **Trace back to the spec.** Is the spec ambiguous? Missing a
   constraint? Prescribing something that doesn't work? Or is
   the spec correct and the subagent simply implemented it
   incorrectly?

4. **Fix the spec if needed.** If the spec is the problem,
   correct it. Be specific — add the constraint, prescribe the
   approach, clarify the ambiguity. A vague spec fix produces
   vague output.

5. **Regenerate.** Now that you understand the problem and the
   spec addresses it, regeneration is targeted rather than
   hopeful.

### A real example

In one session, generated code used a standard library function
(`filepath.Match`) that behaved differently across operating
systems. Tests passed on Windows but failed on Linux. The spec
was not wrong — it simply didn't prescribe which function to
use, so the subagent chose one that seemed reasonable.

The first instinct was to regenerate. Instead, the team
investigated: they read the test output, traced the failure to
the function's platform-dependent behavior, identified that the
spec needed to prescribe a platform-independent alternative
(`path.Match`), updated the spec, and regenerated. The fix was
permanent.

Had they regenerated without diagnosing, the subagent might have
chosen the same function again — or a different one with its own
problems.

### The principle

Regeneration is not debugging. The subagent generates artifacts
from the spec it receives. If the spec doesn't address the
problem, no amount of regeneration will fix it. Diagnosis is the
step that turns a failing test into a better spec.

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
affect this node's chain hash.

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
`code-from-spec/_rules/CODE_FROM_SPEC.md` (the pinned copy
installed by `cfs-init-repo`) and loads the working guidelines.

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
