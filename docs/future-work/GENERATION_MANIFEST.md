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

Replace the per-file artifact tag with two framework
artifacts: a committed `.manifest` file and a
gitignored `.cache/` directory. Both live under
`code-from-spec/`.

The `.manifest` centralizes artifact tracking. The
`.cache/` stores previous content for delta
computation. See **Structure** below for details.

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

The manifest stores per-position content hashes for
each artifact (see **Structure**). When an artifact
becomes stale, comparing old positions against current
positions shows exactly which changed. This answers
"why is this stale?" immediately and enables informed
decisions about regeneration strategy.

### Delta-aware regeneration

Instead of the agent receiving only the current chain
and the existing artifact, it can receive:

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

The delta is derived automatically from the manifest
positions and the `.cache/`. For each position whose
hash changed, the framework reads the old content
(from cache, keyed by old hash) and the new content
(from cache, keyed by new hash) and presents the diff.
The signal is reproducible and requires no human
intervention.

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

## Structure: `.manifest` + `.cache/`

Two artifacts, different lifecycles:

**`.manifest`** — committed to git. One entry per
artifact, four fields:

```
SPEC/payments/fees/calculation/implementation:
  output: internal/fees/calculation.go
  artifact: Kx9mP2...
  chain: Jz3qR7...
```

- **Key** — the logical name of the spec node that
  generates the artifact. Unique identifier.
- **output** — the file path where the artifact lives
  on disk (relative to project root).
- **artifact** — hash of the file content at the time
  of generation or last `accept`.
- **chain** — the chain hash at the time of generation.

These four fields enable all framework checks:

- **Staleness**: chain in manifest != current chain
  hash of the node → stale, needs regeneration.
- **Tamper**: artifact in manifest != hash of file on
  disk → someone edited the file manually.
- **Orphan**: key in manifest references a node that
  no longer exists in the spec tree → artifact has
  no spec, can be deleted.

The `.manifest` replaces the per-file artifact tag.
Generated files no longer contain
`code-from-spec: SPEC/x/y@hash` — that information
lives exclusively in the manifest. Artifacts become
clean application code with no framework metadata.

The `.manifest` must be versioned because it **is**
the staleness state. If it disappears, the framework
cannot know which artifacts are current — it would
have to assume everything is stale and regenerate
the entire tree.

The manifest also stores the chain positions for
each artifact — the ordered list of content hashes
that produced the chain hash:

```
SPEC/payments/fees/calculation/implementation:
  output: internal/fees/calculation.go
  artifact: Kx9mP2...
  chain: Jz3qR7...
  positions:
    - a1b2c3...   # SPEC (Public)
    - d4e5f6...   # SPEC/payments (Public)
    - g7h8i9...   # SPEC/payments/fees (Public)
    - j0k1l2...   # ARTIFACT/extraction/proto
    - m3n4o5...   # SPEC/external/database (Public)
    - p6q7r8...   # target Public
    - s9t0u1...   # target Agent
```

Each hash in the positions list corresponds to a
file in the `.cache/`. The chain that produced the
artifact is fully reconstructible.

This enables:
- **Delta computation**: compare old positions vs
  new, read both versions from cache, generate
  per-position diffs for the subagent.
- **Auditing**: "what exactly did the subagent see
  when it generated this artifact?" — the answer is
  in the cache, reconstructible from the manifest.
- **Debugging**: artifact has a bug → read the
  positions from the cache → see exactly the context
  that produced the error.

**`.cache/`** — gitignored. Stores the processed
content of each chain position — the exact content
that participates in the chain hash. This includes:

- **Specs**: the Public section content.
- **Artifacts** (referenced via `ARTIFACT/`): the
  artifact content without the artifact tag.
- **Externals**: the full file content.

Each file is named `.<sha1>` (dot-prefixed hash, no
extension). Deduplication is automatic — identical
content is stored once regardless of how many chains
reference it. The dot prefix signals infrastructure
(like `.git/objects`) and prevents editors/IDEs from
indexing cache content.

### How the cache is populated

The cache is populated by every MCP operation that
reads or writes content:

- **`reconstruct_cache`** — new MCP tool. Reads the
  `.manifest`, and for each position hash of each
  artifact, checks if the hash exists in the cache.
  If not, reads the current file, extracts the
  public content that participates in the chain hash,
  and caches it. This is the bootstrap operation —
  run it after cloning a repo (which has the
  `.manifest` from git but no `.cache/`) or after
  the cache is lost. Idempotent — only fills gaps.

- **`load_chain`** — every time the MCP reads a file
  to assemble a chain, it caches the processed
  content. This happens organically during normal
  work. If the human edits a spec, the previous
  content is already cached (from `reconstruct_cache`
  or a prior `load_chain`), and the new content is
  cached on the next `load_chain`. Both versions
  coexist — delta computation works.

- **`write_file`** — after writing an artifact and
  updating the manifest, the artifact content is
  cached. Future `load_chain` calls that reference
  this artifact via `ARTIFACT/` will find it in the
  cache.

- **`accept`** — after accepting a manual edit, the
  new file content is cached under its hash.

### The typical workflow

1. **Clone repo from GitHub**: has `.manifest`
   (committed), no `.cache/` (gitignored).
2. **`reconstruct_cache`**: reads manifest, populates
   cache from current file state. Now the cache is
   the "before" snapshot for any future changes.
3. **Edit specs**: human modifies spec files.
4. **`load_chain`**: MCP reads modified specs, caches
   new content. Cache now has both "before" and
   "after" versions.
5. **Generation**: framework computes delta from
   cache (old positions vs new positions), delivers
   it to subagent alongside the full chain.

### Graceful degradation

The cache is best-effort infrastructure. Without it,
the framework works — staleness, tamper, and orphan
detection all depend on the `.manifest`, not the
cache. What degrades without cache:

- **Delta computation**: unavailable. The subagent
  receives the full chain without a changes section,
  exactly like today.
- **Auditing**: the chain that produced an artifact
  is not reconstructible until the cache is
  repopulated.

Each operation that touches a file contributes to
the cache. Over the course of a session, the cache
self-completes. `reconstruct_cache` accelerates this
to a single upfront step.

### Pruning

Delete cache files whose hash is not referenced by
any position in the manifest. Safe to run at any
time — unreferenced content is by definition no
longer needed for delta computation or auditing.

## Orphan detection

The `.manifest` enables automatic orphan detection.
When `validate_specs` runs, it crosses manifest
entries against existing spec nodes. If a manifest
entry references a node that no longer exists in the
tree, the artifact is an orphan — its spec was
deleted but the generated file remains on disk. The
framework reports orphans and offers to delete them.

Today, orphan detection depends on human memory.
In a real session, two job modules were deleted from
the spec tree but their generated files stayed on
disk until someone thought to check.

## Manual edits and the accept flow

### The problem today

With per-file artifact tags, the framework cannot
detect manual edits. The tag stays in the file, the
chain hash still matches, and the framework reports
"up to date." The edit is invisible — whether it was
a legitimate cosmetic fix or an accidental logic
change.

### How the manifest changes this

The manifest stores the artifact hash (hash of the
file content at generation time). When `validate_specs`
runs, it hashes the file on disk and compares:

- **Hashes match**: artifact is as generated.
- **Hashes diverge**: someone edited the file.

The framework reports the divergence as **tampered**
— distinct from stale (chain changed) and orphan
(node deleted).

### The accept command

Not all manual edits are mistakes. A human or
orchestrator may edit a generated file for legitimate
reasons: fixing a linter warning, adjusting
formatting, correcting a cosmetic issue that does
not warrant full regeneration.

The flow:

1. Human edits the generated file directly.
2. Framework detects tamper (artifact hash diverges).
3. Human runs `cfs accept <file>`.
4. The manifest updates the artifact hash to match
   the current file on disk. The chain hash is
   unchanged.
5. The artifact is now **current** again — accepted
   as-is.

On the next regeneration of that node, the subagent
receives the accepted (edited) file as existing
artifact. It may preserve the edit or revert it.
If the edit matters (e.g. a linter rule) and the
subagent reverts it, the linter catches it again.
The loop is self-healing.

### When to accept vs. when to regenerate

`accept` is for edits that do not change behavior:
linter compliance, formatting, renaming a local
variable. The human takes responsibility for the
edit being cosmetic.

For logic changes, `accept` is the wrong tool. The
spec should change and the artifact should be
regenerated. Accepting a logic change bypasses the
spec-as-source-of-truth principle — the code and
the spec diverge silently.

The decision is always the human's. The framework
provides the mechanism; the human provides the
judgment.

### Artifact tags no longer needed

With the manifest, the subagent no longer writes an
artifact tag in the generated file. The `load_chain`
response no longer includes a `chain_hash:` line for
the subagent to embed. The MCP handles all manifest
bookkeeping in `write_file` — computing the content
hash, recording it alongside the chain hash, and
updating the manifest atomically.

The subagent generates pure application code. The
framework metadata lives entirely in the manifest.
This eliminates a class of generation errors (wrong
hash, missing tag, malformed tag) and reduces the
subagent's cognitive load.

## Cascade review

When an ancestor changes, all descendants become
stale. Most may not need regeneration — the ancestor
change may not affect them. But determining this
automatically is not possible: any content in the
chain can influence generation, and only the
subagent knows how.

The framework can facilitate a human decision. With
the `.cache/` storing previous content, the framework
computes the delta for each stale artifact: which
chain positions changed, and what the diff is. A
command like `cfs cascade-review` shows this
information and lets the human decide per artifact:
regenerate or accept (bump chain hash in manifest
without regenerating).

This is a risky call — a change that looks cosmetic
may have introduced or resolved an ambiguity that
changes the subagent's interpretation. The safe
default is always regenerate. The efficient option
is giving the human enough information to make an
informed skip. The framework supports both; the human
chooses.

## Initialization and migration

### New projects

A new project starts with no `.manifest`. Every
artifact is stale by definition — the framework has
no record of what was generated from what. The first
`cfs-generate` run creates the `.manifest` from
scratch as each artifact is generated.

### Manifest loss

If the `.manifest` is deleted, the framework loses
all staleness state. Every artifact becomes stale
and must be regenerated. This is the cost of losing
the manifest — equivalent to losing `go.sum` and
having to re-verify every dependency. The manifest
must be committed to git to prevent accidental loss.

### Migration from artifact tags

Existing projects have `code-from-spec: SPEC/x/y@hash`
tags embedded in generated files. The migration path:

1. A `cfs manifest init` command scans all generated
   files for artifact tags.
2. For each tag found, it creates a manifest entry:
   key from the logical name in the tag, output from
   the file path, artifact hash computed from the
   current file content, chain hash from the tag.
3. The `.manifest` is written and committed.
4. From this point, new generations write to the
   manifest and omit the tag from generated files.
5. The old tags remain in files as inert comments
   until those files are regenerated — at which point
   they disappear naturally.

This avoids a full regeneration on migration. The
manifest is bootstrapped from existing tags, and the
project transitions incrementally.

## Open questions

- **Concurrency.** Multiple subagents run in parallel,
  calling `load_chain` and `write_file` concurrently.
  The MCP must use file locking: shared lock during
  `load_chain` (allows concurrent reads, blocks
  writes), exclusive lock during `write_file`
  (blocks everything else). The lock scope covers the
  entire `load_chain` duration (not just the manifest
  read) to prevent reading a partially-written
  artifact. Both Linux (`flock`) and Windows
  (`LockFileEx`) support shared/exclusive file locks
  natively. Go libraries like `github.com/gofrs/flock`
  abstract both platforms.

- **Git discard.** When the user discards changes via
  git (`checkout`, `restore`), the manifest and
  artifacts may go out of sync if only one is
  discarded. The framework detects this as tamper
  (artifact hash diverges) or staleness (chain hash
  diverges). The cache may have lost entries that are
  now relevant again — `reconstruct_cache` repopulates.
  Not perfect, but self-correcting: any inconsistency
  is detected, and the worst case is regeneration
  without delta (today's behavior).

- **Location.** `.manifest` at
  `code-from-spec/.manifest`, `.cache/` at
  `code-from-spec/.cache/`. Both use dot-prefix,
  consistent with a proposed framework-wide rule:
  any directory or file starting with `.` anywhere
  in the spec tree is ignored by the framework.
  This replaces the v4 convention of `_`-prefixed
  directories only at the top level.

- **Manifest format.** YAML as shown in examples, or
  a custom line-oriented format for easier diffing
  and merging. The manifest will be committed and
  appear in PRs — the format should be easy to read
  in a diff view.

- **Manifest format.** YAML as shown in examples, or
  a custom line-oriented format for easier diffing
  and merging. The manifest will be committed and
  appear in PRs — the format should be easy to read
  in a diff view.
