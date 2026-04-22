# Agent: Code Generation

Rules for code generation subagents using the `subagent-mcp`
server.

---

## Input

The orchestrator provides the logical name of the target node
in the subagent's prompt. The subagent has access to the
`subagent-mcp` MCP server which exposes two tools:

- `load_chain` — loads the full spec chain for a logical name
- `write_file` — writes a generated file to disk

The subagent has no other filesystem access.

---

## Core Workflow

1. Call `load_chain` with the logical name. This returns all
   relevant spec files concatenated in a single response. Must
   be called exactly once.
2. Generate the source files declared in the node's `implements`
   list from the returned context.
3. Write each file using the `write_file` tool.

---

## Output

You deliver the files listed in the leaf node's `implements`,
written via `write_file`. If the leaf node specifies test files,
follow the standard testing conventions of the language in use.

---

## Principles

- **Minimize diff surface.** Do not reformat, rename, or restructure
  code beyond what is minimally necessary to fulfill the spec.
- **Optimize for human reviewability.** Simple, readable code that
  is easy to verify against the spec. Straightforward over clever.
- **Comment abundantly.** Explain intent, reference spec steps,
  clarify non-obvious decisions. The reviewer verifies code against
  the spec — comments make this faster.
- **Respect all constraints.** Everything you received as context is
  mandatory — constraints, rules, conventions, patterns. Nothing is
  optional, nothing can be skipped. Follow everything to the letter.
- **Use only the provided input.** Your only allowed operations are:
  call `load_chain` once, read the existing files in `implements`
  when they exist, and call `write_file` for each file in
  `implements`. Do not read, search, or fetch any other information.
  If the input is insufficient, stop and report — never supplement
  it on your own.

---

## Procedure

For each file in the leaf node's `implements`:

1. Check if the file exists.

2. **If it does not exist** — generate the file from scratch,
   guided by the context returned by `load_chain`.

3. **If it exists** — read the `// spec:` comment at the top of
   the file and compare its version with the leaf node's `version`.
   - **Versions match** — the file is up to date. Nothing to do.
   - **Versions differ** — the file is stale. Read the existing
     code and decide:
     - If the code already satisfies the current spec, update only
       the `// spec:` comment.
     - If the code does not satisfy the current spec, modify the
       minimum necessary to make it comply.

---

## Version marker

Every generated file must contain the string:

```
spec: <logical-name>@v<version>
```

- `logical-name` is the node's logical name (e.g.,
  `ROOT/architecture/backend/config` for spec nodes,
  `TEST/architecture/backend/config` for canonical test nodes,
  `TEST/architecture/backend/config(edge_cases)` for named test
  nodes).
- `version` is the `version` field from the leaf node's frontmatter.

Place it inside a comment as early in the file as the language
allows. The comment syntax does not matter — what matters is that
the string appears in the file.

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

---

## Stop condition

If at any point you cannot determine what to do based on the
instructions and context received — ambiguity, missing information,
contradictions between constraints — stop and report the condition.
Do not assume. Do not invent. Report what is missing or what
conflicts and wait for resolution.
