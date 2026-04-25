# Code From Spec

**Code From Spec** is a methodology where code is a generated
artifact, not the source of truth. The source of truth is a
hierarchy of specification files. To change behavior, you change
the spec and regenerate. You never edit generated code directly.

This methodology is designed for AI agent participation at every
stage — writing specs, managing versions, detecting and resolving
staleness, generating code, and assisting non-technical contributors
with spec authoring.

---

## How it works

Specifications are organized as a tree. Each node is a directory
containing a `_node.md` file. Child nodes add precision to their
parents — high-level intent at the root, implementation detail at
the leaves. Only leaf nodes generate code.

```
code-from-spec/
└── payments/
    └── fees/
        ├── calculation/
        │   └── _node.md   ← leaf → generates code
        └── rounding/
            └── _node.md   ← leaf → generates code
```

Every `_node.md` begins with a YAML frontmatter carrying a `version`
integer. When a node changes, its version increments. Dependents
that reference an outdated version are **stale** and must be
reviewed and regenerated.

Leaf nodes may also have test nodes — `*.test.md` files placed
inside the same directory — which generate test files following
the same versioning rules.

---

## Getting started

See [GETTING_STARTED.md](docs/GETTING_STARTED.md) for step-by-step
setup: spec directory, tooling installation, MCP configuration,
and an AGENTS.md template ready to copy.

---

## Methodology files

| File | Purpose |
|---|---|
| [`rules/CODE_FROM_SPEC.md`](rules/CODE_FROM_SPEC.md) | Full methodology: spec structure, versioning, staleness, code generation |
| [`rules/CODE_GENERATION.md`](rules/CODE_GENERATION.md) | Code generation with subagents |

---

## Tools

| Repository | Description |
|---|---|
| [tool-staleness-check](https://github.com/CodeFromSpec/tool-staleness-check) | CLI tool that automates staleness verification |
| [tool-subagent-mcp](https://github.com/CodeFromSpec/tool-subagent-mcp) | MCP server that provides confined tools for code generation subagents |

---

## Versioning

`main` is the development branch. Released versions live in
dedicated branches (`v1`, `v2`, ...) and receive only bugfix
commits. Breaking changes always produce a new version branch.

To fetch a specific version of the methodology, use the raw URLs
from the appropriate branch:

```
https://raw.githubusercontent.com/CodeFromSpec/framework/v1/rules/CODE_FROM_SPEC.md
```
