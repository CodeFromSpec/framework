# Migration Guide: v5 to v6

How to migrate a project with an existing v5 spec tree to
v6 of the methodology.

This is a living document on the development branch: every
change that affects existing projects must add its entry
here when it lands, not at release time. Entries are
grouped by what they demand from the migrating project.

> [!WARNING]
> v6 is under development. This guide is incomplete until
> v6 is frozen.

---

## Breaking changes

Changes that require action before a v5 project works
under v6.

### Manifest header version

The manifest header line changed from `code-from-spec: v5`
to `code-from-spec: v6`. The v6 series of
`tool-framework-mcp` reads and writes the new header.

**Action:** upgrade the tooling to the v6 series (rerun the
tooling download step of `cfs-init-repo`, or replace the
binary in `code-from-spec/.tools/`). The existing manifest
must have its header updated — regenerating the manifest
also works, at the cost of every artifact being treated as
stale.

### Capability tokens for `load_chain` and `write_file`

`load_chain` and `write_file` no longer accept a
`logical_name` parameter. They take a `token` minted by the
new `create_token` operation, which the orchestrator calls
before dispatching each generation subagent. The subagent
receives only the token, cannot mint one itself, and
therefore cannot load the chain of, or write the output
for, any node other than its target (see TOOLING.md,
`create_token`).

**Action:** upgrade the tooling to the v6 series and
re-download the `cfs-generate` skill and the
`cfs-artifact-generation` subagent definition. v5 skills
and subagents pass logical names to `load_chain` and
`write_file`, which the v6 tooling rejects.

---

## Behavioral changes

Changes that do not break the format but alter what the
framework does — for example, a chain hash change that
marks every artifact stale on first validation. None yet.

---

## Migration steps

The consolidated procedure, in order. To be finalized when
v6 freezes.

1. Upgrade `tool-framework-mcp` to the v6 series.
2. Update the manifest header (see above).
3. Re-download the skills and subagent definitions from the
   `v6` branch.
4. Run `validate_specs` and review what it reports.
