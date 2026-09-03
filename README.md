# Code from Spec v7

**Code from Spec** is a methodology where specifications drive the code. To 
change behavior, you change the specifications and regenerate. Generated 
artifacts are never edited directly — a fix the specifications don't carry will 
not survive the next regeneration.

This methodology is designed for AI agents to participate at every stage, from 
spec authoring to artifact generation to debugging.

---

## Versions

> [!WARNING] 
> **This is the development branch (`main`) and may contain unreleased
> changes.** 

For a stable release, use a version branch:

| Version | Branch                                                                                                 |
|---------|--------------------------------------------------------------------------------------------------------|
| v6      | [https://github.com/CodeFromSpec/framework/tree/v6](https://github.com/CodeFromSpec/framework/tree/v6) |
| v5      | [https://github.com/CodeFromSpec/framework/tree/v5](https://github.com/CodeFromSpec/framework/tree/v5) |
| v4      | [https://github.com/CodeFromSpec/framework/tree/v4](https://github.com/CodeFromSpec/framework/tree/v4) |
| v3      | [https://github.com/CodeFromSpec/framework/tree/v3](https://github.com/CodeFromSpec/framework/tree/v3) |
| v2      | [https://github.com/CodeFromSpec/framework/tree/v2](https://github.com/CodeFromSpec/framework/tree/v2) |
| v1      | [https://github.com/CodeFromSpec/framework/tree/v1](https://github.com/CodeFromSpec/framework/tree/v1) |

---

## How it works

Specifications are Markdown files with the `.cfs.md` extension under
`code-from-spec/`. A spec with `type` generates one artifact (or one verdict)
at its declared output path: its content is the instructions for that
generation, and its frontmatter declares what feeds it — `imports` for context,
`input` for material to transform. Specs without `type` carry the shared
content the others import: contracts, conventions, domain knowledge.

```
code-from-spec/
├── conventions/
│   └── golang.cfs.md          ← imported as shared conventions
└── payments/
    ├── fees-contract.cfs.md   ← the module's contract, imported by consumers
    ├── fees.cfs.md            ← generates internal/fees/fees.go
    └── fees-tests.cfs.md      ← generates internal/fees/fees_test.go
```

Staleness is detected automatically by comparing a hash of everything that feeds 
a generation — the spec's `imports`, `input`, and own content — against the hash 
recorded in the manifest at the time of generation. When they differ, the 
artifact is stale and must be regenerated. Generated files carry no framework 
metadata — the manifest holds all the bookkeeping.

Regeneration is not a blind rewrite. The tooling assembles each generation's 
context as a **spec chain**: the current spec content with changes marked, the 
previous content of what changed, and the existing artifact as a reference. The
generator sees exactly what changed since the last generation — so the existing 
code keeps diffs small and stable, without ever overriding what the spec now 
says.

---

## Theory

This methodology's design decisions are not ad hoc — they follow from a theory 
of spec-driven development, published at 
[codefromspec.com/theory](https://codefromspec.com/theory).

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

Code from Spec turns knowledge into software. It owns the path from 
specification to generated code — not the infrastructure the code runs on. 
Provisioning, deployment, and operations (databases, clusters, CI/CD) are out of 
scope. Technical decisions like "use Postgres with serializable isolation" enter 
the specs as constraints that shape generation, but the framework does not make 
or execute those decisions.

The tooling (skills, subagents, orchestration) targets **Claude Code**. The spec 
format and the MCP server are client-agnostic, but the orchestration layer 
assumes Claude Code's Agent tool, `.claude/` directory structure, and `/mcp` 
server management. Porting to other tools is possible but out of scope — 
community contributions welcome.

---

## Repository structure

### Rules (methodology specification)

| File                                                 | Purpose                                                          |
|------------------------------------------------------|------------------------------------------------------------------|
| [`CODE_FROM_SPEC.md`](CODE_FROM_SPEC.md)             | Full methodology: spec structure, staleness, artifact generation |
| [`rules/FILE_FORMAT.md`](rules/FILE_FORMAT.md)       | Detailed file format and parsing rules                           |
| [`rules/CHAIN_ASSEMBLY.md`](rules/CHAIN_ASSEMBLY.md) | Chain format, assembly order, and delivery                       |
| [`rules/CHAIN_HASH.md`](rules/CHAIN_HASH.md)         | Chain hash algorithm for staleness detection                     |
| [`rules/MANIFEST.md`](rules/MANIFEST.md)             | Manifest format and artifact status                              |
| [`rules/CACHE.md`](rules/CACHE.md)                   | Cache structure for disposition computation                      |
| [`rules/TOOLING.md`](rules/TOOLING.md)               | Operations a tool must implement                                 |

### Skills (Claude Code)

| Skill                                                  | Purpose                                        |
|--------------------------------------------------------|------------------------------------------------|
| [`cfs-init-repo`](skills/cfs-init-repo/SKILL.md)       | One-time repository setup                      |
| [`cfs-init-session`](skills/cfs-init-session/SKILL.md) | Load guidelines at session start               |
| [`cfs-status`](skills/cfs-status/SKILL.md)             | Report spec health (errors, cycles, staleness) |
| [`cfs-generate`](skills/cfs-generate/SKILL.md)         | Regenerate stale artifacts                     |

### Subagents

| Agent                                                             | Purpose                                       |
|-------------------------------------------------------------------|-----------------------------------------------|
| [`cfs-artifact-generation`](subagents/cfs-artifact-generation.md) | Confined subagent for generating one artifact |
| [`cfs-verdict-generation`](subagents/cfs-verdict-generation.md)   | Confined subagent for generating one verdict  |

### Guides

| File                                                                     | Purpose                                                                                            |
|--------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| [`docs/BEST_PRACTICES.md`](docs/BEST_PRACTICES.md)                       | Practical guidance for spec authoring                                                              |
| [`docs/LAYERS.md`](docs/LAYERS.md)                                       | Progressive refinement layers                                                                      |
| [`docs/DECOMPOSITION.md`](docs/DECOMPOSITION.md)                         | Splitting software into modules and routing dependencies through interfaces                        |
| [`docs/TESTING.md`](docs/TESTING.md)                                     | Organizing test specs and keeping them independent                                                 |
| [`docs/DOCUMENTATION.md`](docs/DOCUMENTATION.md)                         | Generating project documentation from the same specs that generate the code                        |
| [`RELEASING.md`](RELEASING.md)                                           | Freezing a stable version branch and reopening main for the next version (development branch only) |

### Migration guides

| File                                                                     | Purpose                                            |
|--------------------------------------------------------------------------|----------------------------------------------------|
| [`migration_guides/FROM_V5_TO_V6.md`](migration_guides/FROM_V5_TO_V6.md) | Migrating a v5 spec tree to v6                     |
| [`migration_guides/FROM_V6_TO_V7.md`](migration_guides/FROM_V6_TO_V7.md) | Migrating a v6 spec tree to v7 (under development) |

---

## Reference implementation

| Repository                                                               | Description                                                                                       |
|--------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| [tool-framework-mcp](https://github.com/CodeFromSpec/tool-framework-mcp) | MCP server implementing spec validation, chain loading, artifact writing, and manifest management |

---

## Versioning

`main` is the development branch. Released versions live in dedicated branches 
(`v1`, `v2`, ...) and are frozen — they receive fixes only in exceptional cases. 
Breaking changes always produce a new version branch.

To fetch a specific version of the methodology, use the raw URLs from the 
appropriate branch:

```
https://raw.githubusercontent.com/CodeFromSpec/framework/<version>/CODE_FROM_SPEC.md
```
