# Orphan Detection

Status: idea, not designed. Captured from a brainstorm on
2026-06-12.

---

## The problem

An **orphan** is a generated artifact whose artifact tag points
at a node that no longer exists. Orphans appear after the
operations every maturing tree goes through — nodes renamed,
moved, merged, or deleted — and they are silent: the artifact
sits on disk looking legitimate, but no spec governs it, no
staleness check covers it, and no regeneration will ever
update it.

`validate_specs` reports the symmetric case (`missing`: a node
whose artifact does not exist) but cannot see this one. An
orphan is, by definition, a file that no node points at —
walking the tree can never find it.

---

## Why it requires a full scan

The only possible direction is the inverse one: scan files
from the project root looking for the `code-from-spec:` tag
string, and check that the logical name in each tag still
resolves to an existing node.

This is cheaper than it sounds:

- The artifact tag was designed to be greppable — a fixed
  string in any position of any text file. A
  gitignore-aware, binary-skipping, single-pass scan
  (ripgrep-style) handles large repositories in milliseconds.
  `_tools/`, `node_modules/`, `.git/` are excluded for free.
- The scan can be restricted to git-tracked files
  (`git ls-files`) — an orphan worth auditing is a versioned
  one; untracked build debris is not the target.

---

## Design direction

**A separate, explicit command — not part of the default
`validate_specs` run.** Staleness checking runs constantly and
must stay cheap and tree-driven. Orphans are created by rare
events (reorganizations, renames, node removals), so the scan
runs when those events happen, or periodically as a health
check — the same category as full regeneration from scratch.

Possible shapes: a dedicated MCP tool (`find_orphans`) or a
flag (`validate_specs --orphans`).

For each hit, report: file path, the logical name in the tag,
and the resolution failure (node directory gone). The human
decides per case: delete the artifact, re-point it by
recreating the node, or strip the tag if the file was adopted
as a manual file.

**What not to do:** maintain a registry/manifest of generated
outputs to avoid the scan. That would break the framework's
statelessness — a second source of truth that can lie. A stale
manifest reporting false orphans is worse than an honest scan.

---

## Relation to other work

[LAYER_MAPPING.md](LAYER_MAPPING.md) lists orphan detection as
one of its acid-test questions (a removed upstream leaf
orphans the mirrored artifact). This feature is useful
independently of layer mapping and should not wait for it.
