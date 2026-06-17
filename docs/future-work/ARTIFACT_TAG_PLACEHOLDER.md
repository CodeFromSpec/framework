# Artifact Tag Placeholder

Status: idea, not designed. Captured from a brainstorm on
2026-06-14.

---

## The problem

Today the subagent receives the chain hash and the logical
name in its prompt, and is responsible for writing the
complete artifact tag (`code-from-spec: <name>@<hash>`)
in the generated file. This exposes two pieces of
information that the subagent does not need for generation
— it only needs to copy them into a comment. But LLMs
anchor on anything that looks significant: the hash (27
alphanumeric characters) and the logical name (which
suggests directory structure, package names, etc.) become
noise that can influence generation in unintended ways.

This is an instance of a broader principle observed in
practice: **everything anchors.** Comments in existing
artifacts anchor the subagent on stale narratives. Hints
injected by the orchestrator anchor on ephemeral
instructions. The hash and logical name in the prompt
anchor on structural inferences. Every piece of
information the subagent receives that is not the spec
is a potential competing signal.

## Proposed solution

The subagent writes a **placeholder** instead of the real
tag:

```
code-from-spec: __TAG__
```

The `write_file` tool replaces `__TAG__` with the actual
`<logical-name>@<chain-hash>` before writing to disk.

Benefits:
- The subagent never sees the hash — zero anchoring on it.
- The logical name is still present in the chain (it is
  part of the spec content), but not repeated in a
  tag-shaped context that invites structural inference.
- The subagent still decides **where** in the file the
  tag goes (which comment format, which line) — that
  decision stays with the subagent because it knows the
  file type. Only the **value** moves to the tool.
- Simpler subagent prompt — no need to explain hash
  extraction from `load_chain`'s first line.

## Changes required

- `write_file` tool: after receiving file content, replace
  the literal string `code-from-spec: __TAG__` with the
  real tag before writing. The tool already receives the
  logical name as a parameter; it needs the chain hash
  too (either passed explicitly or looked up).
- `load_chain` tool: the `chain_hash:` first line could
  be removed from the response, since the subagent no
  longer needs it. The hash would be consumed only by
  `write_file` internally.
- Subagent definition (`cfs-artifact-generation.md`):
  simplified — no hash extraction step, just "place
  `code-from-spec: __TAG__` as early in the file as
  practical."
- `cfs-generate` skill prompt: simplified — no hash
  extraction instructions.

## Relation to other work

Part of a broader effort to minimize anchoring surface
in the subagent's context. Related decisions:
- No comments in generated artifacts (removes narrative
  anchors from the existing artifact)
- Orchestrator must not inject hints beyond the template
  (removes ephemeral anchors from the prompt)
- This proposal removes structural anchors (hash, logical
  name in tag context) from the prompt
