# Code From Spec

**Code From Spec** is a methodology where code is a generated
artifact, not the source of truth. The source of truth is a
hierarchy of specification files. To change behavior, you change
the spec and regenerate. You never edit generated code directly.

This methodology is designed for AI agent participation at every
stage — writing specs, managing versions, detecting staleness,
running resyncs, generating code, and assisting non-technical
contributors with spec authoring.

---

## How it works

Specifications are organized as a tree. Each node is a directory
containing a `_node.md` file. Child nodes add precision to their
parents — business intent at the root, implementation contracts at
the leaves. Only leaf nodes generate code.

```
spec/
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

External dependencies (third-party APIs, shared libraries) live
under `external/`, each as a folder with an `_external.md` entry
point.

---

## Methodology files

| File | Purpose |
|---|---|
| [`framework/CODE_FROM_SPEC.md`](framework/CODE_FROM_SPEC.md) | Full methodology: spec structure, versioning, staleness, resync procedure |
| [`framework/AGENT_CODE_GENERATION.md`](framework/AGENT_CODE_GENERATION.md) | Agent instructions: code generation |

---

## Tools

| Repository | Description |
|---|---|
| [tool-staleness-check](https://github.com/CodeFromSpec/tool-staleness-check) | CLI tool that automates staleness verification |

---

## Versioning

`main` is the development branch. Released versions live in
dedicated branches (`v1`, `v2`, ...) and receive only bugfix
commits. Breaking changes always produce a new version branch.

To fetch a specific version of the methodology, use the raw URLs
from the appropriate branch:

```
https://raw.githubusercontent.com/CodeFromSpec/framework/v1/framework/CODE_FROM_SPEC.md
```
