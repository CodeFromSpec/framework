# Documentation

How to generate project documentation — API references,
deployment guides, READMEs — from the same spec tree that
generates the code.

This document assumes familiarity with
[CODE_FROM_SPEC.md](../CODE_FROM_SPEC.md) and
[LAYERS.md](LAYERS.md). Documentation generation is a layer:
it consumes specs from other subtrees via `input` and produces
Markdown artifacts.

---

## Why documentation drifts

Hand-written documentation describes the same system the code
implements, but nothing forces the two to change together. A
field is added to a struct; the doc page is not updated. A
status transitions from terminal to transient; the README
keeps the old table. The drift is invisible until someone
reads the docs and acts on stale information.

The usual remedy is discipline — review checklists, "update
the docs" in the PR template. Discipline works until it does
not, and the failure is silent: no build breaks, no test
fails, no staleness is detected. The docs simply stop being
true.

Code from Spec already solves this for code: the spec is the
source of truth, the code is derived, and staleness is
detected by hash. Documentation is the same problem on a
different output format. The same machinery applies.

---

## Documentation as a layer

A **documentation layer** is a subtree — typically
`code-from-spec/doc/` — whose leaf nodes consume specs from
other branches of the tree and produce Markdown files. Each
leaf node's `input` points to the spec that defines the
contract being documented. When that spec changes, the doc
page's chain hash changes, staleness is detected, and the
page is regenerated.

The documentation layer does not describe the system from
scratch. It transforms existing contracts — operation
interfaces, configuration structs, domain descriptions —
into a format intended for a different audience. The
operation spec describes a Go function signature, request
parameters, and error variables; the doc page describes a
JSON request, a response example, and an error table. The
spec is the same; the rendering is different.

This is the key property: **documentation is a rendering of
the contract, not a copy that must be kept in sync.** A copy
can drift. A rendering is regenerated from the source.

---

## The convention tree

Documentation pages share style rules — field table formats,
section structure, content restrictions, terminology. In a
hand-written project these rules live in a style guide that
authors consult. In Code from Spec they live in
**convention nodes**: intermediate nodes with no `output`
that exist solely to inject inherited rules into every
descendant's chain.

```
code-from-spec/doc/
├── _node.md              ← root conventions (audience, house style)
├── api/
│   ├── _node.md          ← API page template, common patterns
│   ├── index/
│   │   └── _node.md      → doc/API.md
│   ├── create-user/
│   │   └── _node.md      → doc/API/CreateUser.md
│   └── list-users/
│       └── _node.md      → doc/API/ListUsers.md
└── deploy/
    ├── _node.md           ← deployment page template
    └── index/
        └── _node.md       → doc/Deployment.md
```

The root convention node (`doc/_node.md`) defines what
applies everywhere: the audience, what pages may and may
not contain, the house style. Category convention nodes
(`doc/api/_node.md`, `doc/deploy/_node.md`) define the page
template for their category. Leaf nodes generate individual
pages.

When a generation subagent processes `doc/api/create-user`,
it sees the full chain: `doc/_node.md` → `doc/api/_node.md`
→ `doc/api/create-user/_node.md`. Every style rule, every
content restriction, every template structure flows down
automatically. A new doc page added six months later inherits
the same rules without the author needing to repeat them.

---

## What convention nodes carry

Convention nodes answer two questions: what does every page
in this category look like, and what must no page contain.

### Audience

State who reads these pages and what they already know. This
scopes the subagent's choices — technical depth, assumed
terminology, what needs explanation and what does not.

### Page template

Define the section structure that every page in this
category follows. For API operations, a template might
prescribe: title, description, request section (JSON example
+ field table), response section (JSON example + field
table), errors section (table), optional notes section. For
deployment, a different template: infrastructure, environment
variables, permissions, schedules.

The template is not a layout hint — it is a structural
contract. Every page generated under this convention node
will follow it, because the subagent sees it in the chain
and the Agent section of the leaf node says "follow the
structure defined in the parent node."

### Content restrictions

State what must never appear. For API documentation aimed at
integrators, this typically includes: internal package names,
function signatures, database column names, implementation
algorithms. These restrictions prevent the subagent from
leaking implementation details into user-facing pages, even
when the `input` spec contains them (which it will — the
input is an implementation spec).

### House style

Formatting rules that apply to every page: heading levels,
table column names, how monetary amounts are described, how
dates are formatted, whether to use emoji. The more specific
the rules, the more consistent the output across independent
subagent runs.

### Common patterns

Recurring structures that appear across multiple pages. For
example, if many API operations accept an optional
`CompanyID` field with the same type, requiredness, and
description, define the pattern once in the convention node.
If pagination follows a standard shape (cursor, page size,
end-of-list flag), define it once. The subagent applies the
pattern rather than reinventing it for each page.

---

## The leaf node

A documentation leaf node has three parts:

**Frontmatter** — declares `input` (the spec being
documented), optional `imports` (additional context), and
`output` (the generated file path).

**Public section** — contains `## Path` with the output
path. This is importable by other nodes via
`SPEC/doc/api/create-user(Path)`, which lets index pages
build links without depending on the content of individual
pages.

**Agent section** — tells the subagent what to produce.
For most pages, this is one line: "Render the input as an
operation page, following the structure defined in the
parent node." For pages with special requirements — a "How
it works" section, financial code tables, a status values
reference — the Agent section adds specific guidance.

A typical leaf node:

```yaml
---
input: SPEC/implementation/api/operations/create-user
output: doc/API/CreateUser.md
---
```

```markdown
# SPEC/doc/api/create-user

Generates the CreateUser documentation page.

# Public

## Path

`doc/API/CreateUser.md`

# Agent

Render the input as an operation page, following
the structure defined in the parent node.
```

The input spec contains Go structs, error variables,
request parameter tables, and behavioral context. The
convention node (inherited in the chain) contains the
rules for transforming these into JSON-facing API
documentation. The subagent does the transformation.

---

## Cross-linking with qualified imports

Index pages need to link to individual pages. A naive
approach — hardcoding relative paths — breaks when pages
move. Declaring `imports` on the full content of each page
pulls too much into the chain and makes the index stale whenever
any page's content changes.

The solution uses qualified `imports` with the `(Path)`
selector. Each doc leaf node exposes a `## Path` subsection
in its `# Public` containing only the output file path. The
index node depends on `SPEC/doc/api/create-user(Path)`,
which delivers one line — `doc/API/CreateUser.md` — without
pulling in any of the page's other content.

```yaml
imports:
  - SPEC/doc/api/create-user(Path)
  - SPEC/doc/api/list-users(Path)
  - SPEC/doc/api/delete-user(Path)
```

The index subagent receives the paths and builds relative
markdown links. When a page is renamed (its `output` and
`## Path` change), the index becomes stale and regenerates
with the updated link. When a page's content changes but its
path does not, the index is unaffected.

---

## Choosing what to document

Not every spec node needs a documentation page. The
documentation layer generates pages for the **external
contract** — what a consumer, operator, or integrator needs
to use the system. Internal implementation details are
documented by the spec tree itself and do not need a
second rendering.

A useful heuristic: if someone outside the team needs to
read it, it belongs in the documentation layer. If only the
spec tree's own generation subagents need it, it does not.

Common documentation artifacts:

- **API reference** — one page per operation, generated from
  the operation's interface spec. An index page with tables
  grouping operations by domain.
- **Deployment guide** — generated from the configuration
  spec (environment variables, timeouts, feature flags) and
  infrastructure specs (IAM policies, database tables,
  scheduled jobs).
- **README** — generated from the domain spec (capabilities,
  ecosystem, constraints), with links to the API reference
  and deployment guide.
- **Event catalog** — one page per event type, generated from
  the event spec. An index page with queue assignments.
- **Client library reference** — generated from the client
  library's interface spec.

Each of these is a category in the convention tree, with its
own intermediate convention node defining the page template.

---

## The generated-file banner

Generated documentation files will be edited by someone who
does not know they are generated — a new team member, a
contributor from another project, a future version of the
author who has forgotten. An HTML comment on the first line
prevents the wasted effort:

```html
<!-- Generated from code-from-spec/. Do not edit directly. -->
```

Place this rule in the root convention node so every page
inherits it. The banner is invisible when rendered (GitHub,
documentation sites) but visible when the file is opened in
an editor — exactly where the warning is needed.

---

## Adding a new page

When a new operation is added to the system, adding its
documentation page is four steps:

1. Create the directory and `_node.md` under the appropriate
   category (`doc/api/new-operation/`).
2. Set `input` to the operation's interface spec, `output` to
   the doc file path.
3. Add `## Path` in `# Public` and a one-line `# Agent`
   directing the subagent to follow the parent template.
4. Add a `SPEC/doc/api/new-operation(Path)` entry to the
   index node's `imports` so the index links to the new
   page.

Run `validate_specs` — the new page appears as `missing`.
Generate it. The index regenerates in a later rank (its chain
changed because of the new `imports` entry). The new page
inherits every convention, every restriction, every template
rule from the ancestors it has never seen.

---

## The input is not the audience

The most common mistake in documentation specs is writing
the `# Agent` section as if the subagent's audience were the
spec author. The subagent's audience is the reader of the
generated page — an integrator, an operator, a new team
member. The convention node states who this audience is;
the Agent section should not contradict it.

A subtler form of the same mistake: letting implementation
terminology leak into the Agent instructions. "Include the
Go struct fields" produces a page with Go types. "Include
the request fields with their JSON types" produces a page
for an API consumer. The convention node's content
restrictions exist to catch the cases the Agent section
misses, but the Agent section should not rely on them as
a safety net.

---

## Limits

Documentation generation works best for structured,
reference-style content: field tables, status codes, error
catalogs, configuration parameters. This is content that
maps directly to spec content and benefits most from the
drift guarantee.

It works less well for narrative content — architecture
overviews, design rationale, onboarding guides — where the
value comes from editorial judgment rather than spec
accuracy. A subagent can produce competent prose, but the
prose is not the point; the point is that when the spec
changes, the page regenerates. If the page's value is in
how it tells a story rather than what facts it states, the
regeneration may produce a different story that is equally
correct but less useful.

The practical boundary: generate reference material,
hand-write narrative material, and do not mix them in the
same file. A README that combines a status table (reference)
with a design philosophy section (narrative) will regenerate
the table correctly and rewrite the philosophy
unpredictably. Split them, or accept that the narrative
section will vary across generations.
