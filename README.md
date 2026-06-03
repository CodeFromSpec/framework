# Code From Spec v3

**Code From Spec** is a methodology where code is a generated
artifact, not the source of truth. The source of truth is a
hierarchy of specification files. To change behavior, you change
the spec and regenerate. You never edit generated artifacts
directly.

This methodology is designed for AI agent participation at every
stage — writing specs, generating artifacts, detecting staleness,
and assisting non-technical contributors with spec authoring.

---

## Versions

> [!WARNING] 
> **This is the development branch (`main`) and may contain unreleased
> changes.** 

For a stable release, use a version branch:

| Version | Branch                                                                                                 |
|---------|--------------------------------------------------------------------------------------------------------|
| v2      | [https://github.com/CodeFromSpec/framework/tree/v2](https://github.com/CodeFromSpec/framework/tree/v2) |
| v1      | [https://github.com/CodeFromSpec/framework/tree/v1](https://github.com/CodeFromSpec/framework/tree/v1) |

---

## How it works

Specifications are organized as a tree. Each node is a directory
containing a `_node.md` file. Child nodes add precision to their
parents — high-level intent at the root, implementation detail at
the leaves. Only leaf nodes generate artifacts.

```
code-from-spec/
└── payments/
    └── fees/
        ├── calculation/
        │   └── _node.md   ← leaf → generates artifacts
        └── rounding/
            └── _node.md   ← leaf → generates artifacts
```

Staleness is detected automatically by comparing a hash of each
node's chain against the hash recorded in its generated artifacts.
When they differ, the artifact is stale and must be regenerated.

---

## Getting started

### 1. Install the init skill

Copy and paste the following prompt into Claude Code:

````
Download the Code from Spec init skill from
https://raw.githubusercontent.com/CodeFromSpec/framework/main/skills/cfs-init/SKILL.md
and save it to `.claude/skills/cfs-init/skill.md`.
Create the directory if needed.
````

### 2. Initialize the project

```
/cfs-init
```

This will create the spec root, download tooling, configure
the MCP server, and install skills and subagent definitions.

### 3. Working with the agent

At the start of each session, reference the methodology file:

```
@CODE_FROM_SPEC.md
```

If context gets cluttered during a long session, `/clear`
and `@CODE_FROM_SPEC.md` again to reset.

---

## Methodology

### Core

| File | Purpose |
|---|---|
| [`rules/CODE_FROM_SPEC.md`](rules/CODE_FROM_SPEC.md) | Full methodology: spec structure, staleness, artifact generation |
| [`rules/FILE_FORMAT.md`](rules/FILE_FORMAT.md) | Detailed file format and parsing rules |
| [`rules/CHAIN_HASH.md`](rules/CHAIN_HASH.md) | Chain hash algorithm for staleness detection |
| [`rules/ARTIFACT_GENERATION.md`](rules/ARTIFACT_GENERATION.md) | Artifact generation with subagents |

### Guides

| File | Purpose |
|---|---|
| [`docs/BEST_PRACTICES.md`](docs/BEST_PRACTICES.md) | Practical guidance for spec authoring and external imports |
| [`docs/LAYERS.md`](docs/LAYERS.md) | Progressive refinement layers and extraction layers |

---

## Tools

| Repository | Description |
|---|---|
| [tool-framework-mcp](https://github.com/CodeFromSpec/tool-framework-mcp) | MCP server for spec validation, chain loading, chain hashing, and artifact writing |

---

## Client compatibility

The methodology is client-agnostic in principle — the spec
tree format and the MCP server work with any MCP-compatible
client. In practice, the skills and subagent orchestration
currently assume **Claude Code**: the Agent tool for
dispatching subagents, the `.claude/` directory for skills
and agent definitions, and `/mcp` for server management.

Using Code from Spec with a different client requires
reimplementing the orchestration layer (how subagents are
dispatched, how ranks are parallelized, how existing
artifacts are passed). The MCP server itself is portable.

---

## Versioning

`main` is the development branch. Released versions live in
dedicated branches (`v1`, `v2`, ...) and receive only bugfix
commits. Breaking changes always produce a new version branch.

To fetch a specific version of the methodology, use the raw URLs
from the appropriate branch:

```
https://raw.githubusercontent.com/CodeFromSpec/framework/v2/rules/CODE_FROM_SPEC.md
```
