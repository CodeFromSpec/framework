# Code Generation with Subagents

Instructions for code generation subagents using the
`subagent-mcp` MCP server.

---

## Overview

The subagent's job is to verify that a specification is complete
and unambiguous enough to generate code from. If it is, it proves
it by generating the code. If it is not, it reports exactly what
is missing or contradictory. Both outcomes are equally valid.

The subagent may be called during specification design to find
gaps, or during code generation to produce files. It does not
know which — it behaves the same either way.

---

## Tools

The subagent has access to two MCP tools provided by the
`subagent-mcp` server:

- `load_chain` — loads the full spec chain for a given name
- `write_file` — writes a generated file to disk

The subagent has no other tools or filesystem access.

---

## Workflow

1. Call `load_chain` with the logical name the orchestrator
   provided. This returns a concatenated set of specification
   files. Must be called exactly once.

2. The response contains multiple files separated by delimiters.
   Each file has a `node:` and `path:` header. Find the file
   whose `node:` matches the logical name — this is the
   **target**. Everything else is supporting context.

3. The target file contains a YAML block between `---` delimiters
   at the top. In that block, the `implements` field lists the
   source files to generate, and the `version` field is the
   current version number.

4. For each file listed in `implements`, verify that the target
   and context provide enough information to implement it. Note
   anything ambiguous, missing, or contradictory.

5. If issues were found in step 4, report the findings and stop.
   Otherwise, proceed to step 6.

6. Generate each source file. Use the target file as the primary
   specification and the rest of the context for constraints,
   conventions, and reference material.

7. Call `write_file` once per file listed in `implements` to
   write the result, passing the logical name as `logical_name`.

---

## Rules

### Optimize for human review

A human may need to review the output against the specification.
Everything below serves that goal — the subagent should spend
extra tokens and time if it makes the result easier for a human
to verify.

- **Comment abundantly.** Explain intent, clarify non-obvious
  decisions, and document constraints that influenced the
  implementation.
- **Write straightforward code.** Simple and readable over clever
  and compact.
- **Minimize changes.** When updating an existing file, only
  modify what is needed to meet the specification — no unnecessary
  reformatting or restructuring. Smaller diffs are easier to
  review.
- **Skip unnecessary work.** If the existing code already
  satisfies the specification, do not regenerate it.

### Spec comment

Every generated file must contain the string:

```
spec: <logical-name>@v<version>
```

- `logical-name` is the logical name of the target node.
- `version` is the `version` field from the target's YAML block.

Place it inside a comment as early in the file as the language
allows. The comment syntax does not matter — what matters is
that the string appears in the file.

Example (Go):

```go
// spec: ROOT/architecture/backend/config@v5
package configuration
```

Example (Python):

```python
# spec: ROOT/architecture/backend/config@v5
```

Example (SQL):

```sql
-- spec: ROOT/architecture/backend/config@v5
```

### Strict compliance

Every rule and convention in the context is mandatory; nothing
is optional.
