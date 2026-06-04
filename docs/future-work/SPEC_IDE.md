# Spec IDE

A dedicated application for browsing and understanding
the spec tree visually. The IDE is for **reading and
navigation**, not for editing — spec editing is done
through AI-assisted tools like Claude Code, which
understand the semantic implications of changes
(updating a config field implies updating tests,
implementation, service dispatch, etc.).

---

## Core concept

The spec tree lives on the filesystem as directories
and `_node.md` files. This is the right persistence
format — it works with git, grep, and standard tools.
But it is not the best navigation format. A node's
`_node.md` shows only its own content. The full picture
— inherited context, imported dependencies, artifact
status — requires opening multiple files and mentally
assembling the chain.

The IDE would render the full picture for any node:

```
ROOT/.../account-close/implementation

├─ Inherited context (read-only, collapsed by default)
│  ├─ ROOT                          # Public (3 lines)
│  ├─ architecture                  # Public (12 lines)
│  ├─ architecture/backend          # Public (180 lines) ← guard node
│  ├─ internal                      # Public (8 lines)
│  ├─ api                           # Public (5 lines)
│  ├─ operations                    # Public (2 lines)
│  └─ account-close                 # Public (45 lines)
│
├─ Dependencies
│  ├─ errors(Errors)                # Public > ## Errors
│  ├─ celcoin-gateway(Interface)    # Public > ## Interface
│  └─ database                      # Public
│
├─ This node
│  ├─ # Agent (implementation steps)
│  └─ output: internal/accountclose/accountClose.go
│
└─ Artifact status: ✓ up to date
```

This is essentially what `load_chain` assembles for
generation — but rendered for human consumption.

---

## Node ordering

The filesystem sorts directories alphabetically, which
does not always reflect the intended reading order —
especially for layers (e.g., `database/` appears before
`domain/`).

The IDE would support an `order` field in the
frontmatter to control display order among sibling
nodes:

```yaml
---
order: 10
depends_on:
  - ROOT/external/payments-api
output: internal/transfers/transfers.go
---
```

Lower values appear first. Nodes without `order` are
sorted alphabetically after ordered nodes. The framework
ignores unknown frontmatter fields, so `order` can be
added to specs today without breaking anything.

---

## Non-technical contributor support

The spec tree uses YAML frontmatter and markdown
conventions that non-technical contributors may find
unfamiliar. The IDE would provide:

- Guided spec creation: asks questions, generates
  frontmatter, places the file in the correct
  directory.
- Templates for common node types.
- Validation that frontmatter is well-formed before
  commit.
- AI-assisted spec authoring: the contributor
  describes behavior in natural language, the agent
  structures it into a valid spec node.

---

## What the IDE is not

The IDE is not a code editor. It does not generate
code, run tests, or manage git. It is a **spec
browser** — the interface through which humans
understand and navigate the spec tree. All mutations
to the tree go through Claude Code or equivalent
AI-assisted tools, which understand the ripple effects
of changes.

This separation is deliberate: a markdown editor
cannot know that adding a field to config implies
updating 6 other specs. An AI assistant can.
