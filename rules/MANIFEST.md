# Manifest

The manifest tracks the state of every generated artifact.

This document assumes familiarity with
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md).

---

## Location

The manifest is a single file at
`code-from-spec/.manifest`. It is committed to version
control.

If the manifest is lost, the framework has no record of
what was generated — all artifacts are treated as stale.

---

## Format

One entry per artifact. Each entry has four fields:

```
SPEC/payments/fees/calculation:
  output: internal/fees/calculation.go
  artifact: Kx9mP2vB7wY2tHsJ8dFak4Xz9pQ
  chain: Jz3qR7nL5cW1gT4yK8mDfAx0vBe
```

- **Key** — the logical name of the leaf node. Unique
  identifier for the entry.
- **output** — the artifact file path, relative to the
  project root.
- **artifact** — hash of the file content at the time of
  generation.
- **chain** — the chain hash at the time of generation.

All hashes use the same algorithm and encoding defined
in CHAIN_HASH.md (SHA-1, base64url, 27 characters).

---

## Artifact status

The `validate_specs` tool reports the status of each
artifact. Four states exist:

### Current

The chain hash in the manifest matches the current chain
hash of the node, and the artifact hash in the manifest
matches the hash of the file on disk. The artifact is
up to date.

### Stale

The chain hash in the manifest does not match the current
chain hash of the node. The specification has changed since
the artifact was last generated. The artifact must be
regenerated.

### Tampered

The artifact hash in the manifest does not match the hash
of the file on disk. The file was modified outside of the
framework. An artifact can be both stale and tampered.

### Missing

The artifact file does not exist on disk.

### Orphan

The manifest contains an entry whose logical name does not
correspond to any existing node in the spec tree. The node
was deleted or renamed, but the artifact and manifest entry
remain.

---

## Manifest updates

The manifest is updated by the `write_file` tool during
artifact generation. When `write_file` writes an artifact,
it records the chain hash and the artifact hash in the
manifest atomically.

No other operation modifies the manifest. The manifest is
never edited manually.
