# Tooling

Operations that a Code from Spec tool must implement. The
reference implementation is
[tool-framework-mcp](https://github.com/CodeFromSpec/tool-framework-mcp).

This document assumes familiarity with
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md) and
[MANIFEST.md](MANIFEST.md).

---

## validate_specs

Validate the spec tree and report the status of all
artifacts.

**Parameters:** none.

**Returns:** a report containing format errors, cycles,
and artifact status. Always returns a report — never
raises an error. Problems are collected in the report.

**Behavior:**

1. Walk the spec tree and check every node for format
   errors (see FILE_FORMAT.md and CODE_FROM_SPEC.md).
2. Detect circular references across `depends_on`,
   `input`, and inheritance. Report cycle participants.
3. For each node that declares `output`, determine the
   artifact status by comparing the manifest against
   the current spec tree and file system (see
   MANIFEST.md, "Artifact status"). Each entry includes
   the node's rank — entries with equal rank have no
   dependency between them and can be processed in
   parallel.
4. Report all findings: format errors, cycles, and
   artifact status (stale, modified, missing, orphan).

Nodes without `output` are not checked for staleness —
they do not generate artifacts.

---

## load_chain

Load the complete spec chain for a given node.

**Parameters:**

- `logical_name` (string, required) — the logical name
  of the target node. The node must declare `output`.

**Returns:** a single document with the following
structure:

```
--- context ---
<chain content>
--- input ---
<input content>
--- existing artifact ---
<existing artifact content>
```

The `--- input ---` section is present only when the
node declares an `input` field. The
`--- existing artifact ---` section is present only
when the output file exists on disk — if the file does
not exist or cannot be read, the section is omitted
silently.

The chain content is the concatenation of all chain
positions in assembly order, as defined in
CODE_FROM_SPEC.md ("Chain assembly"). The delivered
content matches exactly what is hashed — hash and
delivery never diverge (see FILE_FORMAT.md, "Block
extraction").

If any file in the chain (other than the existing
artifact) is unreadable, returns an error.

---

## write_file

Write a generated artifact to disk and update the
manifest.

**Parameters:**

- `logical_name` (string, required) — the logical name
  of the node whose `output` authorizes the write.
  Must not contain a parenthetical qualifier.
- `path` (string, required) — file path relative to the
  project root. Must match the node's declared `output`.
- `content` (string, required) — complete file content.

**Behavior:**

1. Validate that `logical_name` has no qualifier and
   that `path` matches the `output` declared in the
   node's frontmatter.
2. Write the file to disk.
3. Compute the checksum (hash of the written content)
   and the current chain hash.
4. Update the manifest entry for this node with the
   new checksum and chain hash.

The manifest must be updated atomically. See
MANIFEST.md ("Concurrency") for locking requirements.

---

## version

Report the tool version.

**Parameters:** none.

**Returns:** the version string.
