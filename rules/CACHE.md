# Cache

Implementation detail of the reference tooling. The cache
stores chain position content and chain structure to
enable delta computation between generations. It is not
part of the framework specification — tools may implement
delta computation by other means or not at all.

This document assumes familiarity with
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md),
[CHAIN_HASH.md](CHAIN_HASH.md), and
[MANIFEST.md](MANIFEST.md).

---

## Location

The cache lives at `code-from-spec/.cache/` and is
gitignored. It contains two subdirectories:

```
code-from-spec/.cache/
├── .content/    ← position content, keyed by content hash
└── .chains/     ← chain structure, keyed by chain hash
```

---

## Content store

`.cache/.content/` stores the processed content of each
chain position — the exact content that participates in
the chain hash (see CHAIN_HASH.md, "Content hash").

Each file is named `.<content-hash>` (dot-prefixed,
27-character base64url hash, no extension). The file
content is the processed text of the position: the
block-extracted `# Public` subsections for spec nodes,
the full file content for artifacts and externals.

Deduplication is automatic — identical content produces
the same hash and is stored once regardless of how many
chains reference it.

---

## Chain store

`.cache/.chains/` stores the structure of each chain at
the time it was computed. Each file is named
`.<chain-hash>` (dot-prefixed, 27-character base64url
hash, no extension).

The file contains the ordered list of positions that
produced the chain hash. Each line has a label and a
content hash:

```
SPEC/payments: d4e5f6g7h8i9j0k1l2m3n4o5p6q
SPEC/payments/fees: g7h8i9j0k1l2m3n4o5p6q7r8s
SPEC/integrations/database: j0k1l2m3n4o5p6q7r8s9t0u
SPEC/payments/fees/calculation (Public): m3n4o5p6q7r8s9t0u1v2w3x
SPEC/payments/fees/calculation (Agent): p6q7r8s9t0u1v2w3x4y5z6a
ARTIFACT/functional/calc (input): s9t0u1v2w3x4y5z6a7b8c9d
```

The label identifies the position: which node, which
section, and the role (ancestor, dependency, target, or
input). The content hash points to the corresponding
file in `.cache/.content/`.

---

## Population

The cache is populated as a side effect of normal
operations. Every time the tooling reads content for a
chain position — during `load_chain`, `validate_specs`,
or `write_file` — it writes the processed content to
`.cache/.content/` and the chain structure to
`.cache/.chains/`.

Over the course of a session, the cache self-completes.

A `reconstruct_cache` operation reads the manifest and
populates the cache from the current state of all files.
It is idempotent — only fills gaps, skipping content
and chain files that already exist. Pruning (see below)
handles cleanup of unreferenced files separately.

---

## Delta computation

When an artifact is stale, the tooling computes a
`disposition` for each chain position by comparing
the old and current chains:

1. Read the old chain hash from the manifest entry.
2. Look up the old chain structure in
   `.cache/.chains/<old-chain-hash>`.
3. Compute the current chain positions (labels and
   content hashes).
4. Compare old and current positions by label:
   - **`unchanged`** — same label, same content hash.
   - **`changed`** — same label, different content hash.
     Old content is read from `.cache/.content/`.
   - **`added`** — label exists in current but not in
     old.
   - **`removed`** — label exists in old but not in
     current. Old content is read from
     `.cache/.content/`.

The disposition is delivered as an attribute on each
`<entry>` in `<previous_constraints>` and on the
`<previous_instructions>` element in the chain XML
(see CODE_FROM_SPEC.md, "Chain assembly"). Entries
with disposition `unchanged` or `added` are
self-closing (no content to deliver). Entries with
disposition `changed` or `removed` include the old
content from the cache.

If the old chain file is not in the cache, delta
computation is not possible. The `<previous_*>`
sections are omitted from the chain entirely.

---

## Graceful degradation

The cache is best-effort infrastructure. Without it,
the framework works — staleness, tamper, and orphan
detection all depend on the manifest, not the cache.
What degrades without cache:

- **Delta computation** — unavailable. The subagent
  receives the full chain without a changes section.
- **Auditing** — the chain that produced an artifact
  is not reconstructible until the cache is
  repopulated.

---

## Concurrency

Cache files are write-once — once created, they are
never modified. This eliminates the need for file
locking on the cache.

- **Writes** must be atomic (write to a temporary file,
  then rename). A cache file either exists completely
  or does not exist.
- **Concurrent writes** of the same hash produce
  identical content. One rename wins; the result is
  correct either way.
- **Reads** always see a complete file or no file.
- **Pruning** deletes only unreferenced files.
  Concurrent deletes of the same file are idempotent.

---

## Pruning

Content files in `.cache/.content/` whose hash is not
referenced by any chain file in `.cache/.chains/` can
be deleted. Chain files in `.cache/.chains/` whose hash
is not referenced by any manifest entry can be deleted.
Pruning is safe to run at any time.
