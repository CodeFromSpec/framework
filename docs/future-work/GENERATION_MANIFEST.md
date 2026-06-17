# Generation Manifest

A centralized manifest that records the state of every
generated artifact and the chain that produced it.
Analogous to `go.sum` in Go modules or lockfiles in
package managers.

---

## The problem

Today, staleness is tracked by an artifact tag embedded
in each generated file:

```
code-from-spec: SPEC/x/y@<chain-hash>
```

This works but has costs:

- **Cascade noise.** When a root node changes, every
  artifact that inherits from it becomes stale. After
  regeneration, each file has a different tag hash —
  even if the code did not change. The git diff shows
  hundreds of modified files with one changed line each.
  Review, blame, and bisect become less useful.

- **Agent burden.** The generation subagent must
  remember to include the tag, position it correctly,
  and format it right. This is a source of errors.

- **Mixed concerns.** The generated file contains
  framework metadata alongside application content.

- **Limited information.** The tag says what chain hash
  produced the artifact, but not what changed since the
  last generation. The framework knows *if* something
  is stale, not *what* made it stale.

---

## The proposal

Replace the per-file artifact tag with a centralized
manifest file at `code-from-spec/_manifest/` (or a
single file — naming TBD). The manifest records, for
each artifact:

1. **Artifact name** — the output path.
2. **Artifact hash** — a hash of the generated file's
   content.
3. **Chain hash** — the chain hash at the time of
   generation.

Example:

```
internal/transfers/handler.go:
  artifact: Kx9mP2...
  chain:    Jz3qR7...

internal/transfers/handler_test.go:
  artifact: Wn4bL1...
  chain:    Pm8sT5...
```

---

## What this enables

### Clean git history

When a cascade-only regeneration produces the same code,
the artifact file does not change — only the manifest
entry's chain hash updates. The git diff shows one file
(the manifest) instead of hundreds. Files that did not
actually change stay untouched.

This reinforces codebase maturity: code that has not
changed accumulates trust silently. The manifest absorbs
the churn that today pollutes the git history.

### Tamper detection

Comparing the manifest's artifact hash against the file
on disk detects manual edits to generated files. Today,
if someone edits a generated file, the framework has no
way to know — the artifact tag is still present and the
chain hash still matches. With the manifest, the
artifact hash would diverge.

### Granular staleness

If the manifest stores not a single chain hash but a
hash per chain position, you know exactly what changed:

```
internal/transfers/handler.go:
  artifact: Kx9mP2...
  chain:
    SPEC:                         a1b2c3...
    SPEC/payments:                d4e5f6...
    SPEC/payments/fees:           g7h8i9...
    ARTIFACT/extraction/proto:    j0k1l2...
    SPEC/integrations/db:         m3n4o5...
    SPEC/.../calculation:         p6q7r8...  # public
    SPEC/.../calculation:         s9t0u1...  # agent
    ARTIFACT/functional/fees:     v2w3x4...  # input
```

When an artifact becomes stale, diffing the stored
hashes against the current hashes shows which positions
changed. This answers "why is this stale?" immediately
and enables informed decisions about regeneration
strategy.

### Chain snapshots and delta-aware regeneration

The manifest directory can store snapshots of the chain
positions used in previous generations — not just their
hashes, but their content. This enables a new
regeneration mode:

Instead of the agent receiving only the current chain
and the existing artifact, it receives:

1. **The current chain** — the source of truth.
2. **A delta** — which chain positions changed since
   the last generation, with before and after content.
3. **The existing artifact** — the starting point.

The delta is the directional signal that is missing in
today's regeneration. Currently, the agent must discover
for itself what changed in the spec — while the existing
code sits in the context as an equally plausible source
of truth. With an explicit delta, the agent knows
exactly what changed and where to look. The anchoring
problem described in the "Anchoring on old code" article
is directly addressed: the agent no longer needs to
infer the difference between old and new spec — the
framework computes it and delivers it.

This signal is not ad-hoc. It is derived automatically
from the manifest, is reproducible, and does not require
human intervention. Unlike a directional signal injected
into a prompt (a failing test, an error message), the
delta is versioned and deterministic.

The cost is storage — chain snapshots for every
generation. But specs are small text files, and the
manifest directory can be pruned to keep only the
previous generation's snapshot (the minimum needed
to compute a delta).

### Informed regeneration strategy

With granular staleness and delta information, the
framework can make better decisions about how to
regenerate:

- **Cascade-only** (ancestor changed, target node
  unchanged): the delta shows that the target's own
  spec did not change. Incremental regeneration with
  the existing artifact is likely safe — the anchoring
  risk is low because there is nothing new for the
  agent to miss in the target spec.

- **Target spec changed**: the delta shows what
  changed in the target node. The agent receives the
  change explicitly. The risk of anchoring is reduced
  because the directional signal is present.

- **Agent error** (spec is correct, code is wrong):
  the chain hashes match but the code does not pass
  tests. The manifest's artifact hash confirms the
  file was generated, not manually edited. Incremental
  regeneration with the existing artifact plus error
  feedback is the right tool.

---

## Relationship to other proposals

- **Artifact pinning.** A pinned artifact would be
  recorded in the manifest with a `pinned: true` flag.
  The manifest already tracks everything needed — the
  pinning mechanism becomes a manifest annotation rather
  than a separate concept.

- **Variability analysis.** Generating twice and
  diffing requires knowing what the chain was in each
  generation. The manifest with chain snapshots provides
  this naturally.

---

## Open questions

- **Single file vs. directory?** A single manifest file
  is simpler to diff in git. A directory allows storing
  chain snapshots alongside hashes. Could be both: a
  manifest index file plus a snapshots directory.

- **Pruning strategy.** How many previous snapshots to
  keep? One (for delta computation) is the minimum.
  More enables richer analysis but costs storage.

- **Migration.** Existing projects have artifact tags
  in generated files. The transition would need to
  support both modes temporarily, or require a
  one-time full regeneration.

- **Naming.** `_manifest/`, `_state/`, `_generation/`
  — the name should convey that this is framework
  state, not user content.
