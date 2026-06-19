# Layer Mapping

Status: idea, not designed. Captured from a brainstorm on
2026-06-12.

---

## The problem

A layer that transforms another layer's artifacts (see
[LAYERS.md](../LAYERS.md)) costs O(n) mirror nodes that contain
no decisions. A `golang/` layer that implements every artifact
of a `functional/` layer needs one leaf per functional leaf,
and each is pure plumbing:

```yaml
---
input: ARTIFACT/functional/transfers
output: internal/transfers/transfers.go
---
```

Everything intelligent about the layer lives in its
intermediate nodes (the Go conventions, inherited by every
leaf). The leaves are mechanical mirrors of the upstream
tree's shape. Adding a functional node means remembering to
add its mirror; forgetting produces silent gaps.

The wish: a layer that *maps over* another layer —
`golang = map(implement, functional)` — so that a new
functional leaf automatically implies a new implementation
artifact. A plug-and-play layer: guard nodes plus a mapping
declaration, zero per-leaf plumbing.

---

## Option A — virtual nodes (declarative fan-out)

A template node declares an iteration:

```yaml
for_each: SPEC/functional/**      # every leaf with output
input: ARTIFACT/{match}
output: internal/{leaf}/{leaf}.go
```

The framework expands it into implied nodes, one per match,
mirroring the upstream structure. The template's `# Agent`
applies to every instance; the layer's intermediate nodes
provide conventions via inheritance, as usual.

**Tension:** breaks the axiom that a node's position in the
filesystem is its position in the hierarchy. Implied nodes
exist in the tree but not on disk — not visible in the file
explorer, not versioned individually, and manifest entries
reference nodes that cannot be opened. Power bought with opacity, in
a framework whose core thesis is "everything visible,
everything declared."

**Assessment: rejected.** Invisible nodes contradict the
framework's soul.

---

## Option B — materialization by tooling (recommended first step)

A skill or tool command (`cfs-sync-layer`) reads the template
and **writes real `_node.md` files** to disk. Nodes are real,
versioned, visible, auditable. The sync detects new upstream
leaves (creates the mirror) and removed ones (reports
orphans). Framework semantics change *nothing* — same
precedent as resolving repeated `depends_on` with authoring
tooling rather than inherited imports.

A conceptually pleasing variant: the template is itself a leaf
node whose generated artifact is the set of `_node.md` files —
specs generating specs. Real complication: generation creates
nodes that `validate_specs` only sees on the next pass
(two-phase fixpoint).

**Assessment: cheapest experiment.** Costs one skill, zero
spec changes. Try it on a real project and let usage answer
the open questions below.

---

## Option C — mapping as a first-class concept (possible destination)

The top of a layer declares the mapping:

```yaml
maps: SPEC/functional
```

The mirroring rule becomes framework semantics: every upstream
leaf with `output` implies a leaf here, with automatic `input`
and a templated `output` path. **Override by
materialization:** if a real node directory exists at the
mirrored position, it replaces the implied node.

The common case is zero-maintenance; the special case (an
instance needing an extra `depends_on` or a different
`# Agent`) is a normal node, visible, in the expected
position.

**Assessment: the real answer to plug-and-play — but the
largest semantic addition since the tree itself. Do not
specify it before Option B has proven the pattern in
practice.**

---

## The acid-test questions

Any design must answer these:

1. **Heterogeneity.** When 5 of 30 instances need something
   different, the template must NOT grow conditionals —
   template logic is the death of legibility. The answer is
   override by materialization: the moment an instance
   diverges, it becomes a real node. (Works in all options.)

2. **Path templating.** `output: internal/{leaf}/{leaf}.go`
   requires a substitution mini-language. Keep the scope
   closed and tiny (`{path}`, `{leaf}`, little or nothing
   more) or it becomes a template engine.

3. **Orphans.** Upstream leaf removed → mirror disappears →
   the generated artifact remains on disk with a tag pointing
   at a node that no longer exists. `validate_specs` reports
   `missing` (node without artifact); this needs the inverse
   (artifact without node). See
   [ORPHAN_DETECTION.md](ORPHAN_DETECTION.md) — useful
   independently of this feature and tracked on its own.

4. **Composite staleness.** Template changed → all instances
   stale. One upstream artifact changed → one instance stale.
   This falls out naturally from the chain hash in every
   option — a sign the idea is compatible with the core.

---

## Recommendation

Start with Option B (a sync skill, framework untouched).
Promote to Option C only after the materialized pattern has
proven itself on a real project and the acid-test questions
have empirical answers. Option A is rejected.
