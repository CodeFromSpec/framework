# Token Economics

How Code from Spec manages context and cost — the design
decisions that make AI-driven code generation economically
viable at scale.

---

## The context problem

AI agents have a finite context window. No matter how
large it becomes, a non-trivial system will always
contain more knowledge than fits in a single prompt. The
practical question is not whether the agent can hold the
entire system in context — it cannot — but how the right
context reaches the agent at the right time.

In a traditional AI-assisted workflow, the agent reads
source files, grep results, and conversation history to
assemble context. This is ad hoc and fragile. The agent
does not know what it does not know. It reads what it
finds, infers what it can, and guesses the rest.

Code is a poor context source. It records mechanism, not
intent. When an agent reads a function, it sees what the
function does — not why it does it that way, what
alternatives were considered, or what constraints it
respects. Context assembled from code is voluminous but
shallow.

---

## The chain as context mechanism

The spec tree inverts this. Every node is a unit of
explicit context — intent, constraints, decisions,
interfaces. When a generation subagent receives a chain,
it receives exactly the context it needs: the ancestors'
constraints, the dependencies' interfaces, the external
references, the target's spec. Nothing more, nothing
less.

A typical chain for a leaf node is 3,000–5,000 tokens.
This is a fraction of what the agent would need if it
were reading the full repository. A project with 40 Go
packages might have 50,000+ lines of code. The chain
for any single package is under 200 lines of spec. The
ratio is roughly 10:1 — for the same information
quality, the chain costs an order of magnitude fewer
tokens.

This is context management by construction, not by
effort. The author of a spec node does not think about
"what context will the agent need." They declare
dependencies because the node genuinely depends on them.
The context assembly follows automatically. As the tree
grows, adding hundreds of new nodes does not inflate the
context for existing nodes — each node's chain includes
only what it declared.

---

## Confinement as cost mechanism

The methodology confines the generation subagent: it can
only read the spec chain and write the declared output
file. It cannot browse the repository, read unrelated
code, or fetch external documentation.

This confinement is usually discussed as a correctness
mechanism — preventing the agent from anchoring on stale
code or hallucinated context. But it is equally a cost
mechanism:

- **No exploration tokens.** The agent does not spend
  tokens reading files to figure out what to do. The
  chain tells it.
- **No false starts.** The agent does not generate code
  based on incorrect assumptions assembled from
  repository exploration, then need to regenerate.
- **Predictable cost.** Each generation costs roughly
  `chain_tokens + output_tokens`. Both are bounded by
  the spec size, which is known in advance.

In a traditional "read the repo and implement" workflow,
the agent might read 20,000 tokens of code before
writing 500 lines. With confinement, it reads 4,000
tokens of spec and writes the same 500 lines. The
reading cost is 5x lower, and the output quality is
higher because the context is intent, not mechanism.

---

## Guard nodes amortize rule costs

A guard node is an intermediate node whose `# Public`
content prescribes concrete rules that all descendants
inherit. From a token perspective, guard nodes are
remarkably efficient:

A single rule — "import `lib-go-utils/v2/errors` without
alias" — is perhaps 20 tokens in the ancestor node. It
propagates to every leaf in the subtree. In a project
with 60 leaf nodes under that ancestor, the rule costs
20 tokens per generation (it appears once in each chain)
but prevents an error that would cost thousands of tokens
to diagnose and fix per occurrence.

Without the guard node, the rule would need to be
repeated in each leaf's spec — 60 × 20 = 1,200 tokens
of redundant specification, plus the maintenance burden
of keeping 60 copies in sync. With inheritance, 20
tokens in one place does the work of 1,200.

More importantly, guard nodes prevent the expensive
failure mode: a subagent generates incorrect code, tests
fail, a human diagnoses the issue, fixes the spec,
regenerates. Each iteration of this cycle costs 15,000–
30,000 subagent tokens. A 20-token rule that prevents
the failure across 60 nodes saves potentially 900,000–
1,800,000 tokens in avoided rework.

---

## The no-comments rule

Generated artifacts in Code from Spec contain no
comments except the artifact tag. This is usually
framed as a diff-quality decision — comments create
noise across regenerations. But the token economics
are significant:

**Smaller artifacts.** A Go file without comments is
typically 20–30% shorter. When the subagent reads the
existing artifact (via `load_chain`'s "existing
artifact" section), it reads fewer tokens. When it
writes the output, it writes fewer tokens.

**Less anchoring.** When comments are present, the
subagent reads them as a second source of truth. If a
comment says `// Fetch the account by company ID`, the
subagent anchors on that narrative even if the spec has
changed. Without comments, the subagent anchors only
on code structure, which is less narrative and more
amenable to restructuring when the spec demands it.

**Stable regeneration.** Comments vary between
regenerations — different subagent runs produce
different wording for the same intent. This creates
false diffs and wastes human review time. Without
comments, the only diff is logic changes, which is
what the reviewer actually needs to evaluate.

At scale, the savings compound. In a session that
regenerated 111 artifacts, removing comments saved
roughly 220,000 tokens (estimated 2,000 tokens per
file). Over the lifecycle of a project with frequent
regeneration, this is a material cost reduction.

---

## Batching and failure boundaries

Generating all stale artifacts in one pass is the
naive approach. Batching — generating 10 at a time —
is better for both cost and reliability:

**Rank ordering.** Artifacts have dependencies. A rank-5
artifact may depend on a rank-3 artifact's output. If
rank 3 changes, rank 5's chain hash changes too.
Generating in rank order ensures each artifact sees the
current state of its dependencies.

**Early failure detection.** If a subagent produces bad
output, batching catches it after 10 artifacts instead
of 111. The cost of a bad generation is bounded to the
batch size, not the total.

**Session resilience.** Token limits, network errors,
and session interruptions are realities. After a batch
completes, `validate_specs` shows exactly what remains
stale. The next session — or the next batch in the same
session — picks up where the previous one left off.
No manual bookkeeping is needed; the framework's
hash-based staleness detection is its own progress
tracker.

---

## The cost of tag-only updates

When a spec tree changes, the chain hashes of all
descendant artifacts change — even if the content those
artifacts depend on has not semantically changed. This
means a change to a root-level node (adding a config
field, for example) can make every artifact in the
project stale.

Most of these regenerations are tag-only updates: the
subagent reads the existing code, confirms it matches
the spec, and writes it back with the new hash. The
code is identical; only the artifact tag changes. Yet
each such regeneration still costs ~15,000 tokens
(reading the chain + reading the existing artifact +
writing the output).

This is the framework's largest optimization
opportunity. A system that computes "did the content
that this artifact actually uses change?" — as opposed
to "did the chain hash change?" — could skip the
subagent entirely for tag-only updates and stamp the
new hash directly. For a session where 75 out of 111
regenerations were tag-only, this would save roughly
1,100,000 tokens — over 60% of the generation budget.

---

## Context compounds across sessions

In a traditional workflow, a productive four-hour
session with an AI produces code changes and maybe some
comments. The next session starts over — the agent reads
the code and tries to reconstruct what happened. The
context from the previous session is gone.

In Code from Spec, a productive session produces spec
changes. Those changes are the context. The next
session — with the same agent or a different one —
picks up the spec tree as it stands. Every decision
made in every previous session is present in the tree,
structured and accessible. The context compounds across
sessions, across contributors, across time.

The spec tree can represent more knowledge than any
single context window can hold, while still providing
each agent invocation with precisely the context it
needs. The total knowledge in the tree is unbounded.
The context per generation is bounded and curated. This
is how Code from Spec scales beyond the limits of any
individual AI model.

---

## Summary of token flows

| Phase | Without CFS | With CFS |
|---|---|---|
| Context assembly | Agent reads repo (~20k tokens) | Agent reads chain (~4k tokens) |
| Generation | Code from inferred context | Code from explicit spec |
| Failed generation | Diagnose + re-read + regenerate (~30k) | Fix spec + regenerate (~17k) |
| Convention enforcement | Per-file instructions or hope | Guard node inheritance (20 tokens) |
| Comments | Written + read + anchored on | Absent — spec is the documentation |
| Cross-session context | Lost — reassembled from code | Preserved in spec tree |

The economics are clear: smaller context, higher
precision, lower rework. Each design decision in the
framework — confinement, inheritance, no-comments,
chain assembly — serves both correctness and cost. The
two are not in tension. In Code from Spec, the cheapest
path and the most correct path are the same path.
