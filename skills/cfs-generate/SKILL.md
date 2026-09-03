---
name: cfs-generate
description: Generates or regenerates artifacts and verdicts from Code from Spec specs. Use when stale artifacts or verdicts exist, or when the user asks to generate or regenerate them.
---

# Artifact and Verdict Generation

Generate artifacts and verdicts for stale entries reported
by `validate_specs`.

## When invoked

Run this skill when the user asks to generate or regenerate
artifacts or verdicts, or when stale entries exist.

## Prerequisites

1. Verify the framework-mcp MCP server is connected (the
   `validate_specs`, `create_token`, `load_chain`,
   `write_artifact`, and `write_verdict` tools must be
   available).

2. Run `validate_specs`. If `format_errors` are reported, stop
   and tell the user to fix them first — generation requires
   clean specs.

## Algorithm

1. Run `validate_specs` and collect all stale/missing entries —
   artifacts and verdicts.
2. If no stale entries, report that everything is up to date
   and stop.
3. Group stale entries by rank. The rank (returned by
   `validate_specs`) reflects dependency depth — entries
   with lower rank must be generated before entries with
   higher rank, because higher-rank entries may depend on
   them. Process ranks in ascending order. Within the same
   rank, entries are independent and should be dispatched
   in parallel. Skip entries flagged as blocked and report
   them; a blocked entry is never dispatched. For each
   remaining entry, call `create_token` with the spec's
   logical name to mint a token, then dispatch a subagent by
   the entry's prefix: `cfs-artifact-generation` for
   `ARTIFACT/` entries, `cfs-verdict-generation` for
   `VERDICT/` entries.

   The prompt is the same for both subagent types — the
   token and nothing else:

   > Token: `<token>`

4. **After each rank completes, run `validate_specs` again
   before starting the next rank.** This is mandatory, not
   an optimization to skip. Regenerating rank N changes
   artifact content, which may cause rank N+1 entries
   that depend on them to become stale. Without
   re-validating, newly stale entries are missed. The
   `validate_specs` call between ranks is what keeps the
   generation session consistent.
5. After all ranks are processed, run `validate_specs` a
   final time. Report the remaining stale and blocked items
   (if any) to the user.

## Rules

- Dispatch one subagent per entry, each with its own token.
- **Never put a logical name in the subagent prompt — only the
  token.** The token is what confines the subagent to its target
  spec: it cannot mint tokens itself, so it cannot load the chain
  of, or write the output for, any other spec. `create_token` is
  for the orchestrator only.
- **Do not add guidance, hints, or corrections to the subagent
  prompt beyond the template above.** The subagent must
  generate from the chain alone. If a previous generation
  produced a wrong result, the fix belongs in the spec — not
  in an ad-hoc instruction injected into the prompt. Prompt
  additions bypass the chain, are not versioned, do not
  participate in the hash, and will not reproduce on the next
  regeneration.
- Entries with the same rank are independent — dispatch them
  in parallel (single message with multiple Agent tool calls).
  Wait for all entries in a rank to complete before starting
  the next rank.
- Never edit generated files manually — always regenerate via
  a subagent.
- After each subagent completes, report **any feedback** the
  subagent provided beyond confirming the file was written —
  assumptions, decisions, ambiguities, dependencies on code
  not in the chain, or anything else the subagent chose to
  mention. Present the subagent's exact text to the user
  **before** continuing to the next entry. Do not filter
  or classify the feedback — the user decides what matters.
- Report every verdict's result (`pass` or `fail`) to the
  user, with the reasoning from the verdict document.
- If a subagent reports a spec gap that prevented generation,
  surface it to the user. Do not attempt to fill the gap by
  reading the codebase yourself.
- After generation, do not automatically run build or tests
  unless the user asks — report what was generated and let the
  user decide.
- Track and report token usage. After each rank completes,
  report the cumulative subagent tokens spent in this
  generation session. At the end, report the total.
