# Code Generation with Subagents

How an orchestrator uses subagents to generate code from
specifications. This document assumes familiarity with
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md).

---

## Overview

Code generation should be performed by confined subagents. The
orchestrator dispatches a subagent for each stale source file,
providing only the logical name of the target node. The subagent
receives the spec chain, reviews the specification, and either
generates the code or reports gaps.

---

## Confinement

Ideally, a subagent should only have access to the spec chain for the
target node and the ability to write the declared output files.
It should not explore the filesystem, read unrelated files, or
fetch external information. If the chain is insufficient, the
correct action is to report what is missing.

The `subagent-mcp` tool (see Resources in
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md)) enforces this
confinement. Its tools include:

- `load_chain` — returns the complete spec chain for a logical
  name
- `write_file` — writes a file to disk, validated against the
  node's `implements` list

When `subagent-mcp` is available, the orchestrator should
configure the subagent with access to only these two tools and
no other filesystem access. When it is not available, the
orchestrator is responsible for assembling the chain and
delivering it to the subagent by other means (e.g., in the
prompt), and for restricting the subagent's write access.

---

## Prerequisites

Before dispatching subagents:

1. Resolve all spec and test node staleness. Generating code
   from stale specs is wasteful — the output will be stale
   before it is written.

2. If using `subagent-mcp`, ensure it is installed and configured
   (see [Getting Started](../docs/GETTING_STARTED.md)).

3. Ensure a subagent definition is available. A reference
   implementation is provided at
   [subagents/code_generation_subagent.md](../subagents/code_generation_subagent.md).

---

## Dispatching

For each stale source file:

1. Identify the logical name of the node that generates the file.

2. Dispatch a subagent with that logical name in the prompt.

3. The subagent loads the chain, reviews the specification, and
   either writes the files or reports findings.

Multiple subagents may be dispatched in parallel — each operates
independently.

---

## What to expect

The subagent produces one of two results:

- **Generated files** — written to disk. Each file contains a
  spec comment identifying the source node and version.

- **Findings report** — the specification is ambiguous,
  incomplete, or contradictory. The subagent reports exactly
  what is wrong. This is correct output — fix the spec and
  retry.

Both outcomes are equally valid. The subagent may be dispatched
during specification design specifically to find gaps, or during
a resync to produce files.

---

## After generation

1. Check for remaining stale files. If any remain, dispatch
   subagents for the next batch. Repeat until clean.

2. Build and run tests. If anything fails, trace back to the spec
   and correct it. Do not patch the generated code — fix the spec
   and regenerate.


