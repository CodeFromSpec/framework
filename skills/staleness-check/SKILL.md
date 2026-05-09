---
name: staleness-check
description: Runs the staleness-check tool and reports which artifacts are stale or missing. Use when the user asks to check staleness, find stale files, or see what needs regeneration.
---

# Staleness Check

Run the staleness-check tool and present the results.

## When invoked

Run this skill when the user asks to check staleness, find stale
artifacts, or see what needs regeneration.

## Prerequisites

Verify the staleness-check binary exists
(`tools/staleness-check.exe` on Windows,
`tools/staleness-check` elsewhere). If not, tell the user it is
missing and stop.

## Algorithm

1. Run the staleness-check tool.
2. Parse the output. Present results to the user grouped by
   status:
   - **Stale** — artifacts whose chain hash differs from the hash
     in their spec comment.
   - **Missing** — artifacts whose files do not exist on disk.
3. If everything is clean, report that all artifacts are up to
   date.

## Rules

- This skill is read-only. Do not modify any files.
- Do not automatically trigger artifact generation. Present the
  results and let the user decide what to do next.
