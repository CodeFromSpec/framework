# Public Subsection Requirement

All content in `# Public` must be under a `##`
subsection heading. Content directly under `# Public`
without a subsection heading ("loose content") should
be prohibited.

---

## The problem

When the chain assembler concatenates multiple nodes'
`# Public` sections, loose content — text between the
`# Public` heading and the first `##` — has no
delimiter. The subagent receives a stream of text
where the end of one node's loose content and the
beginning of the next node's content (or first `##`)
are visually indistinguishable.

Example of ambiguous chain output:

```
# Public
This node handles account operations.

## Interface
...
```

The subagent cannot tell whether "This node handles
account operations" belongs to the current node or
is trailing content from a previous node in the chain.

---

## Proposed rule

All content in `# Public` must be under a `##`
subsection. Loose content directly under `# Public`
is a format error.

**Before (current, allowed):**

```markdown
# Public

This is the account service.

## Interface
...
```

**After (required):**

```markdown
# Public

## Context

This is the account service.

## Interface
...
```

---

## Changes required

**CODE_FROM_SPEC.md** — Add to the Body > Public
section: "All content in `# Public` must be under a
`##` subsection. Content directly under `# Public`
without a subsection heading is a format error."

**framework-mcp (validate_specs)** — Add a validation
rule that checks for content between `# Public` and
the first `##` heading. Report as a format error with
a clear message suggesting the author wrap the content
in a `## Context` or similar subsection.

**Chain assembler** — No change needed if the rule is
enforced. The assembler already handles `##` subsections
correctly. Enforcing the rule upstream ensures the
assembler never encounters the ambiguous case.

---

## Migration

Existing projects may have loose content in `# Public`.
The validation could initially be a warning, upgraded
to an error after a transition period. A codemod that
wraps loose content in `## Context` would handle most
cases automatically.
