# Future Work

Ideas and plans that are not yet part of the framework.

---

## Agent failure modes

The methodology depends on AI agents. Agents are not
infallible. This section catalogs known failure modes
and the mitigations in place or planned.

### Literal interpretation of ambiguous specs

Spec says "only accepts ROOT/ references." Agent reads
this literally and rejects `ROOT` (without trailing
slash). Spec says "include the public section content."
Agent omits the `# Public` heading because the spec
did not say to include headings.

Every ambiguity that seems clear to a human has two
reasonable interpretations for an agent.

*Mitigation:* Iterative precision. Each failure traces
to a spec gap. Fix the spec, regenerate. The gap is
closed permanently.

### Test scenarios silently omitted

The spec listed a test case. The agent did not generate
it and did not report the omission. The gap was found
during manual review.

*Mitigation:* The artifact generation skill requires
the subagent to report assumptions and gaps. But this
depends on the agent noticing the omission — which is
the same agent that omitted it. Defense in depth: test
specs should be prescriptive enough that omissions
cause compilation errors (e.g., a test that references
a specific setup value will fail to compile if the
setup is missing).

### Cosmetic variation between regenerations

Variable names, struct organization, helper function
grouping differ between regenerations of the same spec.
Pollutes git diffs and makes review harder.

*Mitigation adopted:* No comments in generated code
(comments are the most variable part). Prescribe names
in the spec — function names, error names, record
names, field names. The more the spec prescribes, the
less the agent invents.

*Mitigation planned:* Code formatter pass after
generation to normalize whitespace and import order.

### Over-engineering

Agent adds abstractions, error handling, features, or
patterns not requested by the spec.

*Mitigation:* Subagent rules say "write straightforward
code." But this is a soft constraint. The real defense
is tests — over-engineered code that passes the same
tests as simple code is harmless, just noisy.

### Hallucinated imports or packages

Agent invents an import path that does not exist.

*Mitigation:* Build verification catches this
immediately. The regeneration cycle always ends with
build + test.

### Under-specification

Spec is too vague. Agent fills gaps with reasonable but
unpredictable choices. Different runs produce different
implementations.

*Mitigation:* Variability analysis — review generated
code, classify each variable aspect as "accept" or
"prescribe," tighten the spec. This is ongoing work,
not a one-time fix. Specs get more precise over time.

### Context window exceeded

With hundreds of nodes, the chain from root to a deep
leaf may exceed the agent's context window. The agent
would miss constraints from ancestor nodes.

*Mitigation:* The chain mechanism naturally limits
context to what the node declared. Keep specs concise.
Remove redundancy aggressively. In practice, even deep
chains stay well within current context limits (1M
tokens) for individual node generation.

---

## Standard language layers

Today, adopting Code from Spec for a Go project
requires writing the entire `golang/` layer —
translation rules, interface conventions, test
patterns. This is project-specific work that is
largely the same across projects.

A standard "golang layer pack" would provide the
intermediate nodes (translation rules, error
conventions, test patterns) as a reusable package.
A non-programmer could write functional specs, drop
in the layer pack, and generate Go code.

**What is needed:**

- Extract generic translation rules from
  project-specific configuration (module path,
  dependencies).
- A configuration mechanism at the layer root for
  project-specific values.
- A scaffolding tool that generates leaf nodes from
  the functional tree structure (each functional node
  gets a corresponding interface, implementation, and
  test node).
- Validation with a second project to confirm the
  rules are truly generic.

This extends to other languages: a `python/` layer,
a `typescript/` layer, each with their own translation
rules and conventions.

---

## Automated variability analysis

Currently, variability analysis is manual: a human
reads the generated code and classifies each variable
aspect.

**Planned:** An agent-assisted workflow that:

1. Generates code from a spec twice (different
   sessions).
2. Diffs the two outputs.
3. Classifies each difference as cosmetic or
   behavioral.
4. Reports behavioral differences as spec gaps.

This could run periodically to monitor spec precision
and identify nodes that need tighter prescription.

---

## Spec import from external repositories

Guard nodes, conventions, and platform standards could
live in a shared repository and be imported into
project-specific spec trees. This would allow
organization-wide standards to propagate automatically.

Example: a `platform-standards` repository with guard
nodes for security, compliance, and coding conventions.
Each project imports them and depends on them. When the
standard changes, all projects detect staleness.

This connects to standard language layers — a Go layer
pack could be distributed as an importable repository.

---

## Spec tooling for non-technical contributors

The spec tree uses YAML frontmatter and markdown
conventions that non-technical contributors may find
unfamiliar.

**Planned:**

- A CLI or web tool that guides spec creation: asks
  questions, generates frontmatter, places the file in
  the correct directory.
- Templates for common node types.
- Validation that frontmatter is well-formed before
  commit.
- AI-assisted spec authoring: the contributor describes
  behavior in natural language, the agent structures it
  into a valid spec node.

---

## Spec visualizer

A dedicated application for browsing and editing specs
visually. Instead of navigating the filesystem, users
would see a node navigator with hierarchy, search, and
inline editing.

### Node ordering

The filesystem sorts directories alphabetically, which
does not always reflect the intended reading order —
especially for layers (e.g., `database/` appears before
`domain/`).

The visualizer would support an `order` field in the
frontmatter to control display order among sibling
nodes:

```yaml
---
order: 10
depends_on:
  - ROOT/external/payments-api
outputs:
  - id: transfers
    path: internal/transfers/transfers.go
---
```

Lower values appear first. Nodes without `order` are
sorted alphabetically after ordered nodes. The framework
ignores unknown frontmatter fields, so `order` can be
added to specs today without breaking anything.

---

## Fragment removal

The `external` field currently supports `fragments` —
line ranges with content hashes for importing specific
portions of large files. In practice, this feature is
difficult to use: it requires knowing exact line
numbers, computing SHA-1 hashes manually, and
maintaining both when the file changes.

A better approach is to use a dedicated extraction node:
import the entire file via `external`, instruct the
agent to extract the relevant portion, and output a
curated artifact. Other nodes consume the artifact via
`depends_on: ARTIFACT/...`.

This uses existing primitives (layers, `external`,
`outputs`) and requires no special tooling. The
extraction re-runs automatically when the source file
changes (the chain hash detects staleness).

**Planned:** Remove `fragments` from `external` in a
future version. This simplifies `CODE_FROM_SPEC.md`,
frontmatter parsing, spec tree validation, chain hash
computation, and eliminates the `hash_fragment` MCP
tool.
