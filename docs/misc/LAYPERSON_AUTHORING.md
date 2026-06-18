# Layperson Authoring: from domain knowledge to software

Status: conceptual, captured from a long brainstorm on
2026-06-13. Article material and methodology guidance, not yet
designed into the framework. Covers three linked threads: the
authoring path, the pseudocode accessibility gradient, and
domain elicitation (the DDD lineage).

---

## 1. The path: problem first, solution second

A domain expert can describe a system at high level without any
technical detail: *"an authorizer that receives a transaction
from a POS terminal and authorizes based on available balance."*
This lives entirely in the plane of ideas.

Two phases, in order:

- **Phase 1 — the problem, conceptual and complete.** Business
  rules in detail, exceptions, edge cases. Solution-agnostic —
  no database, no language, no UI yet. The domain expert is the
  primary author. Crucially, this phase is **itself a
  refinement hierarchy**: a one-line concept at the top
  ("authorize based on balance") down to minute rules at the
  leaves — all without leaving the plane of ideas. It is an
  artifact with standalone value: the problem understood,
  readable and validatable top to bottom before any software
  exists.

- **Phase 2 — the solution shape.** Only after the problem is
  mapped does an engineer decide: is there a frontend? a
  database? integrations? This is engineering judgment, informed
  by the complete problem, and it is **the hinge** — it
  determines which technical subtrees will exist. It is the
  "engineer does more engineering" of the rationale: the noble
  work is designing the solution shape, not typing code.

The data model / schema is NOT the early hinge (an earlier
draft of this thinking wrongly placed it there). It is a
*product* of phase 2, not the bridge. The bridge is the
transition from "problem understood" to "solution shape
chosen."

**The 5-step path:** problem → solution decision → architecture
→ connection → generation. Steps 1 and 5 (the generate/test/fix
loop) are proven by the testimonials. **Step 2 (solution
decision) is the least-mapped part** — the charnel point worth
developing.

### Open nuance: tree vs mind map

The domain in an expert's head is not a strict tree — it is a
web (mind map): "available limit" is defined via "pending
transactions" which relate to "authorization" which consumes
"available limit." A strict hierarchy does not capture sideways
and circular relations.

But the spec tree is not a strict tree either: it is hierarchy
(inheritance) **plus** cross-links (`depends_on`). So
specifying the domain is the act of **rooting the mind map** —
choosing a backbone hierarchy and turning the remaining
associations into explicit links. That conversion is real work,
not automatic. Possible framing: mind map for thinking, tree +
edges for specifying.

---

## 2. The pseudocode accessibility gradient ("the wall is a ramp")

Pseudocode is easy to read, hard to write (it is the
semester-1 skill: decomposing a problem into precise steps).
The read/write asymmetry is exactly where AI fits: **AI writes
(the hard part), the layperson reads and validates (the easy
part).** The "layperson can't author pseudocode" limit
dissolves — they don't author, they recognize.

The deeper insight: the technical wall is not a cliff, it is a
**ramp**. Between domain rules in natural language and real
code there is not one pseudocode layer but a **gradient**, each
level enriched with more technical machinery (begin
transaction, lock the row, commit, rollback) yet still
narratable to a layperson — because each technical concept
carries a business justification ("lock so two purchases don't
spend the same last $10").

Critically, validation **splits** across this gradient — it is
not one thing:

- **Reading the narrative** — preserved deep into technical
  levels.
- **Validating business intent** ("two purchases must not
  double-spend") — the layperson's job.
- **Validating technical correctness** ("is row locking the
  right mechanism?") — the engineer's job.

So the enriched pseudocode is a **shared artifact validated by
two readers on different concerns** — the meeting point of
distributed authorship. Stronger than "the layperson
validates": the layperson and the engineer validate the same
document, each for their layer of concern.

Each enrichment level is a layer in the framework sense
(consumes the prior, adds a class of concern, produces the
next). For a simple technical system the functional layer is
one stage; for a domain-rich + technically-demanding system
(card authorizer) it can be a stack: logic → +persistence →
+concurrency → ...

Honest limits: the ramp ends where business justification runs
out (B-tree index choice, optimistic-locking retry — genuinely
technical, not narratable). And it is a capability, not a
mandate — most systems collapse it to one or two levels; the
gradient is the tool for when you *want* the domain expert to
validate technical-but-business-meaningful decisions (fintech:
locking, idempotency, settlement timing).

### AI fills the authoring gap — as an active interviewer

The AI does not just transcribe; it **probes for omissions**
("what on a tie? expired but with limit? zero amount?"). Without
probing, reading-validation degenerates into rubber-stamping
(the site's "faithfully implements an ambiguous request"
failure). The omission taxonomy and the validation-node /
pseudocode-unit-test mechanisms are in
[VALIDATION_NODES.md](VALIDATION_NODES.md).

---

## 3. Domain elicitation and the DDD lineage

### Entities are latent in the layperson's own words

From *"card number, amount, store... deny if no limit, blocked,
or expired"* the entities fall out: **Purchase** (card, amount,
store), **Card** (available limit, blocked, expiry), **Store**.
Phase 2's data modeling is not a fresh elicitation — it is the
agent **reading back the nouns the layperson already used**,
organized in business terms, for the layperson to validate and
complete ("you forgot the card also has a daily limit"). The
expert validates because entity and attribute are business
nouns, not technical structures. Where modeling hits a real
decision, the agent frames it as a *business* question ("is the
available limit a number you store, or the total minus pending
purchases?").

### Node granularity heuristic for domain rules

Rules that **define each other, change for the same reason, and
describe the same business concept** go in the same node.
Shared vocabulary rises to the ancestor (the glossary /
ubiquitous language). When in doubt, merge — splitting later is
cheap. The split signal is objective: staleness crossing
boundaries that should not exist, or two different owners
editing the same file.

Important correction: do **not** use "implemented by the same
component" as a grouping criterion at this stage. That leaks
implementation into the domain and inverts the methodology's
arrow (domain is the source; component is derived from it). The
criteria must be entirely intra-domain. Composition across
concepts is handled later by `depends_on`, so the domain need
not anticipate implementation.

### The DDD lineage

Code from Spec is, in effect, Domain-Driven Design with the
maintenance cost removed:

- The **glossary in an ancestor's `# Public`** that descendants
  inherit *is* DDD's ubiquitous language.
- Cohesive rule groupings *are* bounded contexts / aggregates
  emerging.
- DDD's diagnosis was right (the shared language is the asset);
  its economics failed (keeping the language synced with code
  was manual and expensive). AI pays that bill — the spec
  cannot drift because code is derived from it.

Industry read on DDD: ubiquitous language and bounded contexts
survived as near-consensus; the tactical patterns got a
reputation for ceremony ("cargo cult DDD"). DDD is
respected-in-principle, tired-in-practice. So **use the lineage
as recognizable affinity, do not brand as DDD** — "if you did
DDD, you'll recognize the ubiquitous language here, now synced
by construction." Same move the rationale already makes about
1980s formal methods: a second, more recent example of the same
thesis.

---

## Where this should land

The methodology guidance (the path, the elicitation method, the
granularity heuristic) is a candidate for BEST_PRACTICES or a
future GETTING_STARTED guide. The narrative framings (wall is a
ramp, DDD lineage) are candidates for site articles, mined the
way the confinement and context-management articles were.
