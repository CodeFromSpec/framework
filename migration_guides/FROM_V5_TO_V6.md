# Migration Guide: v5 to v6

How to migrate a project with an existing v5 spec tree to
v6 of the methodology.

---

## Prerequisites

- A project already initialized with Code from Spec v5.

## Steps

1. **Update the manifest header.** In
   `code-from-spec/.manifest`, change the first line from
   `code-from-spec: v5` to `code-from-spec: v6`.

2. **Update spec node frontmatter.** For every `_node.md`
   in the spec tree:

   a. Rename `depends_on` to `imports`, if present.

   b. On leaf nodes that declare `output:`, add
      `type: artifact` to the frontmatter.

   c. Move any project-specific top-level frontmatter
      fields under a `custom:` mapping, if present. The
      recognized top-level fields are: `type`, `imports`,
      `input`, `output`, `wait_on`, and `custom`. Any
      other field must move under `custom:`.

3. **Remove v5 tooling.** Delete:

   - `code-from-spec/.tools/`
   - `.claude/skills/cfs-init-repo/`
   - `.claude/skills/cfs-generate/`
   - `.claude/skills/cfs-status/`
   - `.claude/skills/cfs-init-session/`
   - `.claude/agents/cfs-artifact-generation.md`

4. **Download `cfs-init-repo`.** Download the v6
   `cfs-init-repo` skill from
   `https://raw.githubusercontent.com/CodeFromSpec/framework/v6/skills/cfs-init-repo/SKILL.md`
   and save to `.claude/skills/cfs-init-repo/SKILL.md`.

5. **Run `cfs-init-repo`.** It will download the v6 MCP
   server, skills, subagent definitions, and configure
   the project.

6. **Verify.** Run `validate_specs` and review the report.
   Fix any errors it reports.
