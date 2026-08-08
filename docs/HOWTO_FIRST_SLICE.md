# How-To: Your First Vertical Slice

Take a spec tree from empty to a built, running artifact, and
learn the loop you will repeat for everything else.

This document assumes familiarity with
[CODE_FROM_SPEC.md](../CODE_FROM_SPEC.md): nodes, the `Public` /
`Agent` / `Private` sections, chains, and staleness. It is the
first walk through the loop; the deeper documents —
[DECOMPOSITION.md](DECOMPOSITION.md),
[BEST_PRACTICES.md](BEST_PRACTICES.md),
[TESTING.md](TESTING.md) — take over where this one hands off.

You will need a repository with the tooling installed, so
`validate_specs`, chain loading, and artifact writing are
available, plus whatever your artifacts compile or run with.

The running example is the first screen of a React single-page
app, but nothing about the loop is specific to React. Read the
steps as general and the boxed examples as one illustration.

---

## Slice thin, not flat

The instinct on a new tree is to build a layer — write every
data node, then every screen node, then wire them together.
Resist it. Your first pass should be a **thin vertical slice**:
the smallest thing that touches the whole chain end to end,
from the root that carries your conventions down to a leaf that
writes a file you can run.

A thin slice proves the machinery — chain assembly, generation,
staleness, your build — on something trivial, where a mistake
costs minutes. A flat layer proves nothing until the very end,
where a mistake costs a day.

> **Example.** The first slice in the sample project was not
> "the data layer" or "all the screens". It was a theme
> stylesheet, an app frame with a placeholder heading inside it,
> a route table, and the entrypoint that mounts them. Five
> leaves, one of which only rendered the text `Hello world`.
> Useless as product; exact as a proof that a spec could become
> a running page.

---

## Step 1 — Choose the slice

Pick the shortest path from your root conventions to something
that runs. It should cross every kind of node you will lean on
later — a root for shared context, perhaps an intermediate
branch, and at least one generating leaf — while carrying no
real complexity in any of them.

A good first slice is:

- **Runnable.** The leaf produces something your build compiles
  or your app renders, so the end of the loop is a real
  pass/fail, not a document you eyeball.
- **Trivial in content.** A placeholder heading, a constant, an
  empty frame. You are testing the loop, not the design.
- **Representative in shape.** If the project will have
  data-access modules feeding screens, give the slice one thin
  module and one thin screen, so the dependency wiring is
  exercised once, small.

Leave out error states, styling polish, and edge cases.
Everything in the first slice is something you will regenerate a
few times while finding your footing; keep it cheap to
regenerate.

> **Example.** The sample slice left the placeholder text in
> English (`Hello world`) though the product is Portuguese — a
> visible marker that nothing real was on screen yet. A scaffold
> should look like a scaffold.

---

## Step 2 — Write the minimum nodes

Every node speaks to three audiences, and each section is for
one of them:

- **`Public`** is a contract for *other nodes*. It is inherited
  by descendants and imported by nodes that depend on this one.
  Put here only what another node legitimately needs to know:
  types, signatures, the tokens or constants it may consume, the
  constraints it must respect.
- **`Agent`** is instruction for the *generator* of this leaf,
  and no one else. Step-by-step logic, the exact thing to emit,
  the pattern to follow.
- **`Private`** is for *you and the orchestrator*: the decisions
  taken and the alternatives discarded, the open questions.

The most common beginner error is crossing these wires — putting
generation instructions in `Public`, or a contract other nodes
need in `Agent` where they can never see it. The test is
audience: *who needs this fact?* If it is another node, it is
`Public`. If it is the generator writing this one file, it is
`Agent`. If it is a human deciding, it is `Private`.

Keep the root lean. Its `Public` reaches the chain of every
descendant, so a sentence added there is paid for in every
generation. Anything true of only one branch belongs in that
branch, not the root.

When a leaf consumes another node, route the dependency through
the **interface**, not the implementation — `imports:
SPEC/…/x(interface)` rather than the artifact — so an internal
change to `x` does not needlessly restale this one. The cases
where you *do* want the artifact (and the generation ordering it
buys) are covered in [DECOMPOSITION.md](DECOMPOSITION.md).

> **Example.** The theme node split cleanly three ways.
> `Public`: "consume the colours through these Tailwind
> utilities; the sidebar width is `--sidebar-width`, referenced,
> not copied." `Agent`: "emit a `:root` block mapping these
> variables to these palette values." `Private`: "why the theme
> lives in its own file and not the one the UI library
> rewrites." Each fact sat where its reader would look.

---

## Step 3 — Validate before you generate

Run `validate_specs`. If it reports **format errors**, stop and
fix them — a malformed tree cannot be generated, and the errors
are cheap to read.

On a clean tree the report is only **staleness**: on a fresh
slice, every artifact is `missing`, which simply means "not
generated yet". Read the **rank** column while you are here. It
is the generation order: rank reflects dependency depth, so a
node's rank is one past the highest rank it depends on. You will
generate in ascending rank.

Do this before every generation session, not just the first. A
clean `validate_specs` is the signal that the tree is coherent
enough to act on.

---

## Step 4 — Generate by rank, validate between ranks

Generate one artifact per subagent, lowest rank first. Artifacts
of the **same rank are independent** — dispatch them together.
Wait for a rank to finish before starting the next.

Between ranks, **run `validate_specs` again**. This is not
optional tidiness. Generating rank *N* changes artifact content,
which restales the rank *N+1* nodes that depend on it; the
intermediate validation is what surfaces them. Skip it and you
generate the next rank against stale inputs.

One discipline holds the whole method together here: **do not
add hints or corrections to the subagent's prompt.** The
subagent generates from the chain alone. If it produced the
wrong thing last time, the fix belongs in the spec — a prompt
patch is invisible to the next generation, unversioned, and
absent from the hash. It will not reproduce, which means it is
not a fix.

---

## Step 5 — Read the diff

A generation is not done when it writes a file. It is done when
you have read what it wrote.

You are not checking whether the subagent produced plausible
code; it almost always does. You are checking whether it had to
**decide anything the spec left open**. Every such decision is a
spec gap wearing the disguise of finished code — and the
compiler will not flag it, because the code is well-formed. It
is simply not what anyone chose.

Two signals repay the reading:

- **A value or name that appears twice.** If the generator
  emitted a number, a class, or a string that also lives
  elsewhere, ask which is the source of truth. If the spec did
  not say, the generator guessed, and the two copies will drift.
- **Something invented to fill a silence.** A token, a helper,
  an element produced because the generator needed *something*
  and the chain supplied nothing. It may even be correct. But
  nobody chose it, so nothing keeps it correct.

> **Example — the value that appeared twice.** The theme node
> declared `--sidebar-width: 280px`; the frame node, told to
> make the sidebar "280px wide", emitted `w-[280px]`. Both
> correct, both `280`, two sources. The build was green.
> Nothing was wrong except that a later change to one would
> silently not reach the other. The gap was not in the code — it
> was that the spec never said *how* to consume the constant.

> **Example — the invented token.** A screen was told to style a
> warning "with the warning token". The theme never defined one.
> The generator wrote `text-warning`, a class that maps to
> nothing, so the text rendered with no colour — and the build
> passed. A silent visual bug no compiler would catch, surfaced
> only by reading the diff.

Reviewing the diff is worth doing whether or not you have
automated tests. Where you have them, a **test spec** turns the
gap you just found into a mechanical check that fails next time
instead of relying on your eye; see [TESTING.md](TESTING.md).
Where you have none, the diff is the whole of your safety net,
and reviewing it is not a courtesy you can skip.

---

## Step 6 — Fix the spec, not the artifact

Neither example above is a bug you fix in the file. The next
regeneration would overwrite the edit, which means a fix that is
not in the spec does not exist. Each gap is the artifact telling
you the specification is incomplete; the repair is to complete
it.

Turn the gap into a spec change at the right node. The duplicated
constant becomes a rule in the theme's `Public`: "consume this by
reference, never by copying the value." The missing token becomes
two edits — define it in the theme, *and* point the consumer at
it — because a token nothing references and a reference to a token
that does not exist are the same gap seen from two ends.

Then regenerate, and watch the cascade restale exactly the
consumers the change reached and no others. That precision is the
staleness machinery confirming your mental model: edit a
contract, and its dependents move; edit prose that no chain
carries, and nothing moves.

When the *subagent itself* reports an assumption — "the spec did
not specify X, so I chose Y" — treat it the same way, but do not
grade it yourself. Surface it to the human, who decides whether Y
is acceptable or reveals a gap. [BEST_PRACTICES.md](BEST_PRACTICES.md)
sets out how to diagnose which side is wrong before you touch
anything; a fix applied to the wrong side reproduces the bug or
damages the spec.

---

## Step 7 — Build or run

The diff and the build catch different failures, and neither
catches the third kind.

- The **diff** catches gaps of intent — decisions the spec left
  open.
- The **build** catches type and reference errors — the code
  does not fit together.
- Only **running it** catches behaviour — the code fits together
  and does the wrong thing.

State to yourself what each pass proved and what it did not. In
the sample slice, the compiler confirmed the warning class was
syntactically fine; only the browser showed that it rendered no
colour. A green build is a necessary signal, never a sufficient
one.

This is the end of the loop, and the shape of every loop after
it: choose, write, validate, generate, read, fix, run.

---

## Pitfalls

- **Editing the generated file.** The next regeneration erases
  it. If the behaviour matters, it goes in the spec.
- **Meta-language in specs.** "This is a leaf node that generates
  an artifact" tells the generator nothing and the reader what
  they already know. Describe the node's purpose, not its
  position in the taxonomy.
- **Predicting shared pieces.** Do not build a component library
  up front. Let a shared node appear when duplication is actually
  observed — a wrong abstraction costs more than the duplication
  it was meant to save.
- **Skipping the between-rank validation.** It is what keeps a
  generation session consistent; without it, freshly staled
  artifacts are generated against stale inputs.
- **Growing the root.** Every clause in the root's `Public` is
  inherited by every generation forever. Push branch-specific
  content down to the branch.

---

## What's next

Once the loop is muscle memory, the guides that build on it:

- **Harvesting a shared node** — when a component or helper earns
  its own node, and how to lift it out of the screens that
  already duplicate it, instead of predicting it.
- **Mocking a boundary** — shaping a data node so the mock and
  the eventual real implementation share one interface, and the
  swap is only the body. See also [LAYERS.md](LAYERS.md).
- **Vendored territory** — keeping code you do not own out of the
  tree without losing sight of it: nothing declares `output`
  there, and a documentation node describes what exists.
