# Artifact Generation with Subagents

How to generate artifacts for a given logical name using a
confined subagent.

This document assumes familiarity with
CODE_FROM_SPEC.md.

---

## Overview

Artifact generation must be performed by confined
subagents. The orchestrator dispatches a subagent and
provides it with the target node's logical name. The subagent
calls `load_chain` to receive the spec chain, generates
the artifact (or reports gaps), and calls `write_file`
to save the result. The tooling handles chain assembly,
manifest updates, and confinement enforcement.

---

## Confinement

A subagent must only have access to the spec chain for
the target node and the ability to write the declared
output file. It must not explore the filesystem, read
unrelated files, or fetch external information. If the
chain is insufficient, the correct action is to report
what is missing.

The `framework-mcp` tool enforces this
confinement. Its tools include:

- `load_chain` — returns the complete spec chain for a logical
  name
- `write_file` — writes a file to disk, validated against the
  node's `output` path, and updates the manifest

The orchestrator configures the subagent with access to only
these tools and no other filesystem access. A reference subagent definition is provided at
cfs-artifact-generation.md (see Resources).

---

## How to generate

Given a logical name:

1. Dispatch a subagent with that logical name in the prompt.

2. The subagent obtains the spec chain, reviews the
   specification, and produces one of two results:

   - **Generated artifact** — written to disk via `write_file`.
     The manifest is updated automatically.

   - **Findings report** — the specification is ambiguous,
     incomplete, or contradictory. The subagent reports exactly
     what is wrong. This is correct output — fix the spec and
     retry.

   Both outcomes are equally valid.

---

## Resources

| Document | Description |
|---|---|
| [CODE_FROM_SPEC.md](https://github.com/CodeFromSpec/framework/blob/main/CODE_FROM_SPEC.md) | Full methodology specification |
| [CHAIN_ASSEMBLY.md](https://github.com/CodeFromSpec/framework/blob/main/rules/CHAIN_ASSEMBLY.md) | Chain format, assembly order, and delivery |
| [MANIFEST.md](https://github.com/CodeFromSpec/framework/blob/main/rules/MANIFEST.md) | Manifest format and artifact status |
| [cfs-artifact-generation.md](https://github.com/CodeFromSpec/framework/blob/main/subagents/cfs-artifact-generation.md) | Reference subagent definition |
