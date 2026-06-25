# Artifact Generation with Subagents

How to generate artifacts for a given logical name using a
confined subagent.

This document assumes familiarity with
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md).

---

## Overview

Artifact generation should be performed by confined subagents.
Given a logical name, the orchestrator dispatches a subagent that
receives the spec chain, reviews the specification, and either
generates the artifacts or reports gaps.

---

## Confinement

Ideally, a subagent should only have access to the spec chain
for the target node and the ability to write the declared output
file. It should not explore the filesystem, read unrelated
files, or fetch external information. If the chain is
insufficient, the correct action is to report what is missing.

The `framework-mcp` tool (see Resources in
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md)) enforces this
confinement. Its tools include:

- `load_chain` — returns the complete spec chain for a logical
  name
- `write_file` — writes a file to disk, validated against the
  node's `output` path, and updates the manifest

The orchestrator configures the subagent with access to only
these tools and no other filesystem access. A reference
subagent definition is provided at
[subagents/cfs-artifact-generation.md](../subagents/cfs-artifact-generation.md).

---

## Regeneration context

The `load_chain` tool assembles the full chain as an XML
document (see CODE_FROM_SPEC.md, "Chain assembly"). When
regenerating a stale artifact, the chain may include
temporal context beyond the current spec:

- **Previous constraints and instructions** — when the
  cache is available, `load_chain` includes the
  constraints and `# Agent` section from the previous
  generation. This lets the subagent see what changed
  in the spec without having to infer it.

- **Existing artifact** — when the output file exists
  on disk, `load_chain` includes its content. The
  subagent uses it as a starting point, producing
  minimal changes and preserving what already satisfies
  the spec. This reduces diff noise and makes code
  review practical.

Neither the existing artifact nor the previous chain
content affects staleness detection. They are delivered
alongside the current chain as reference only.

If the subagent anchors on the existing artifact
(reproducing a bug instead of following the spec),
delete the artifact and regenerate from scratch. The
decision to include or exclude the existing artifact
is the human's, case by case.

---

## How to generate

Given a logical name:

1. Dispatch a subagent with that logical name in the prompt.

2. The subagent obtains the spec chain, reviews the
   specification, and produces one of two results:

   - **Generated artifacts** — written to disk via `write_file`.
     The manifest is updated automatically.

   - **Findings report** — the specification is ambiguous,
     incomplete, or contradictory. The subagent reports exactly
     what is wrong. This is correct output — fix the spec and
     retry.

   Both outcomes are equally valid. The subagent may be
   dispatched during specification design specifically to find
   gaps, or during artifact generation to produce files.
