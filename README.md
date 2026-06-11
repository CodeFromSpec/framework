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
node's context against the hash recorded in its generated artifacts.
When they differ, the artifact is stale and must be regenerated.

---

## Getting started

### 1. Install the init skill

Copy and paste the following prompt into Claude Code:

````
Download the Code from Spec init skill from
https://raw.githubusercontent.com/CodeFromSpec/framework/main/skills/cfs-init-repo/SKILL.md
and save it to `.claude/skills/cfs-init-repo/SKILL.md`.
Create the directory if needed.
````

### 2. Initialize the project

```
/cfs-init-repo
```

This will create the spec root, download tooling, configure
the MCP server, and install skills and subagent definitions.

### 3. Start each session

```
/cfs-init-session
```

This loads the working guidelines into the orchestrator's
context for the current session.

---

## Scope

The tooling (skills, subagents, orchestration) targets
**Claude Code**. The spec tree format and the MCP server
are client-agnostic, but the orchestration layer assumes
Claude Code's Agent tool, `.claude/` directory structure,
and `/mcp` server management. Porting to other tools is
possible but out of scope — community contributions welcome.

---

## Repository structure

### Rules (methodology specification)

| File | Purpose |
|---|---|
| [`rules/CODE_FROM_SPEC.md`](rules/CODE_FROM_SPEC.md) | Full methodology: spec structure, staleness, artifact generation |
| [`rules/FILE_FORMAT.md`](rules/FILE_FORMAT.md) | Detailed file format and parsing rules |
| [`rules/CHAIN_HASH.md`](rules/CHAIN_HASH.md) | Chain hash algorithm for staleness detection |
| [`rules/ARTIFACT_GENERATION.md`](rules/ARTIFACT_GENERATION.md) | Artifact generation with subagents |

### Skills (Claude Code)

| Skill | Purpose |
|---|---|
| [`cfs-init-repo`](skills/cfs-init-repo/SKILL.md) | One-time repository setup |
| [`cfs-init-session`](skills/cfs-init-session/SKILL.md) | Load guidelines at session start |
| [`cfs-status`](skills/cfs-status/SKILL.md) | Report spec tree health (errors, cycles, staleness) |
| [`cfs-generate`](skills/cfs-generate/SKILL.md) | Regenerate stale artifacts |
| [`cfs-spec-review`](skills/cfs-spec-review/SKILL.md) | Review a spec for ambiguities before generating |
| [`cfs-check-meta-language`](skills/cfs-check-meta-language/SKILL.md) | Detect tree-structure references in spec content |

### Subagents

| Agent | Purpose |
|---|---|
| [`cfs-artifact-generation`](subagents/cfs-artifact-generation.md) | Confined subagent for generating one artifact |
| [`cfs-spec-review`](subagents/cfs-spec-review.md) | Confined subagent for reviewing one spec |

### Guides

| File | Purpose |
|---|---|
| [`docs/BEST_PRACTICES.md`](docs/BEST_PRACTICES.md) | Practical guidance for spec authoring |
| [`docs/LAYERS.md`](docs/LAYERS.md) | Progressive refinement layers |

### Documentation

| Directory | Contents |
|---|---|
| [`docs/future-work/`](docs/future-work/) | Planned features and ideas |

The rationale behind the methodology — why it exists and why it
works — is published at
[codefromspec.com](https://codefromspec.com/rationale), along with
[articles](https://codefromspec.com/articles) on specific design
decisions.

---

## Tools

| Repository | Description |
|---|---|
| [tool-framework-mcp](https://github.com/CodeFromSpec/tool-framework-mcp) | MCP server for spec validation, chain loading, chain hashing, and artifact writing |

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
