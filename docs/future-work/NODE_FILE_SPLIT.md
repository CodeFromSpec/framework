# Node File Split

Today, each spec node is a single `_node.md` file that
uses `#` headings to delimit structural sections
(`# Public`, `# Agent`, private sections). This
overloads markdown headings with framework semantics —
the author cannot freely use `#` and `##` for content
organization because the framework reserves them for
section boundaries.

---

## Proposed structure

Split each node into separate files by section type:

```
account-close/
├── _node.md              ← frontmatter only (output, depends_on)
├── _node.public.md       ← public content (inherited, importable)
├── _node.agent.md        ← agent instructions (visible to subagent)
└── _node.private.md      ← decisions, rationale (human-only)
```

Each file is free-form markdown. The section semantics
come from the filename, not from headings. Authors can
use `#`, `##`, `###` freely for content organization
without conflicting with the framework.

---

## Benefits

**Heading freedom.** The `_node.public.md` can use `#`
as a top-level heading for its content. Today, `# Public`
is the top level, and content starts at `##` — which
wastes a heading level and feels unnatural.

**Optional files.** A node without agent instructions
simply has no `_node.agent.md`. No empty section, no
placeholder heading. Presence of the file is the
semantics.

**Cleaner diffs.** Changes to private decisions do not
touch the same file as changes to the public interface.
Review is more focused.

**IDE integration.** The Spec IDE (see SPEC_IDE.md) can
render each file in its own panel — public on the left,
agent on the right, private collapsed. The visual
layout matches the conceptual model.

---

## Migration

The `framework-mcp` would need to support both formats
during a transition period:

- If `_node.md` contains `# Public` / `# Agent`
  headings, parse as today (single-file format).
- If `_node.public.md` exists alongside `_node.md`,
  use the split format.

This allows incremental migration — nodes can be split
one at a time.

---

## Dependencies

- Requires `framework-mcp` changes (parser, chain
  assembler, validator).
- Benefits significantly from the Spec IDE — the split
  format is designed for visual rendering.
- The `load_chain` tool would assemble the chain from
  split files identically to how it assembles from
  single files today — the chain format is unchanged.
