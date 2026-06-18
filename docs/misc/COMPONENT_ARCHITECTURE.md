# Component-Based Architecture as the Natural Fit

Status: conceptual, captured from a brainstorm on 2026-06-13.
Article material. The claim: component-based architecture (CBA)
is not merely compatible with Code from Spec — the spec tree's
primitives are CBA's primitives, and the methodology's
economics actively select for good component decomposition.

---

## 1. The spec tree's primitives *are* CBA's primitives

Not compatible — close to isomorphic:

- **A node with `# Public`** = a component with an interface.
  The `# Public` is literally "what others may use"; the rest
  is hidden. Information hiding is built into the node.
- **`depends_on`** = a declared, explicit component dependency.
  The dependency graph *is* the wiring diagram — and it is the
  source of truth (the artifact won't generate otherwise),
  unlike import statements scattered through code.
- **The tree** = the composition hierarchy. Intermediates
  compose; leaves are atoms.
- **Confinement** = enforced encapsulation. A generation
  subagent sees only the chain — its own spec plus its
  dependencies' interfaces — and physically cannot reach into
  another component's internals. Encapsulation is not a
  convention; it is what is (and isn't) in the chain.
- **One leaf, one artifact** = the atomic component.

Building with Code from Spec *is* doing CBA, named or not,
because the tree forces encapsulation, explicit dependencies,
and composition. The framework-mcp tree is the proof: pure CBA
— interface / implementation / test per component, composed via
`depends_on` on interfaces.

## 2. Same problem, two consumers

CBA and CFS answer the same question — tame complexity through
decomposition + information hiding — for two different
consumers:

- **CBA** tames it for the **human** (reason about one
  component at a time).
- **CFS** tames it for the **AI's context** (generate with a
  bounded chain).

The insight: **good component boundaries are good context
boundaries — the same cut.** A well-decomposed system (low
coupling, high cohesion, clean interfaces) produces small clean
chains; a tangled one produces tangled chains. Cohesion and
coupling for humans = context boundaries for the AI.

## 3. The economics reward CBA (and punish its absence)

Because the interface is what propagates in chains, a fat
interface = fat chains for every consumer = more staleness;
tangled dependencies = huge chains. The methodology has CBA's
values **baked into its cost structure**: the cheapest-to-
generate architecture is the well-componentized one. ("Prefer
qualified depends_on" is the fine-grained instance.) In
traditional development, keeping interfaces lean takes
vigilance; here, not-componentizing *hurts* — in tokens and in
staleness. The incentive self-aligns. Same "cheapest path =
most correct path" theme, applied to architecture.

## 4. What this means for the practitioner

The engineer's irreducible judgment is **where the component
boundaries go** — the AI generates a component, but it does not
decide with good judgment that the component *should exist* and
*where it begins and ends*. That boundary-drawing is exactly
the decomposition skill experienced engineers carry, and the
methodology elevates it (the rationale's "engineer does more
engineering").

Clarification: "component" here is not OO — not a class
hierarchy. The framework-mcp is Go, barely OO, and fully
component-decomposed (packages with interfaces). What maps is
the **decomposition instinct** ("encapsulated unit with an
interface"), language- and paradigm-agnostic.

## 5. The honest nuance: architectures that fight the grain

CBA fits because the tree *is* a static component decomposition.
Architectures that resist decomposition into interface-bearing
units fight the grain:

- the monolithic "big ball of mud" (one giant node, no clean
  `# Public`);
- heavily cross-cutting designs (aspects woven through
  everything — possible via a `depends_on`'d concern tree, e.g.
  security, but it is the awkward case);
- runtime-dynamic composition (the static tree describes it but
  does not naturally mirror it).

And over-fragmentation is a risk for tiny systems ("one leaf,
one artifact" can splinter them) — which is where
[LAYER_MAPPING.md](LAYER_MAPPING.md) would help.

## Two kinds of component

From the framework-mcp tree: `os/`, `parsing/`, `utils/` are
**generic infrastructure** (file IO, text normalization —
nothing tool-specific), reusable shelf bricks; `chain/`,
`spec_tree/`, `mcp_tools/` are **domain-specific**. The
infrastructure kind is the candidate for off-the-shelf,
engineer-vetted technical layers; the domain kind is bespoke,
born from the problem.
