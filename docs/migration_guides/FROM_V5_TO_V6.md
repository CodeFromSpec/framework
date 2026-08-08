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

### `depends_on` renamed to `imports`

The frontmatter field `depends_on` is now called `imports`.
This aligns its name with `input` and `output` — the three
frontmatter fields dedicated to artifact generation — and
with the "imported" language the methodology already used
to describe what the field does. Semantics, prefixes
(`SPEC/`, `ARTIFACT/`, `EXTERNAL/`), and qualifier syntax are
unchanged, and the chain hash does not encode the frontmatter
key name — only the resolved entries — so this rename does
not by itself restale any artifact.

**Action:** upgrade the tooling to the v6 series, then rename
the `depends_on` key to `imports` in the frontmatter of every
leaf node that declares it. Until both are done, a v6 tool
reading a node that still says `depends_on` will treat it as
having no imports.

---

## Behavioral changes

Changes that do not break the format but alter what the
framework does — for example, a chain hash change that
marks every artifact stale on first validation.

### `input` accepts a list

`input` now accepts either a single `SPEC/`, `ARTIFACT/`, or
`EXTERNAL/` name (as before) or a list of names. Existing
nodes with a single scalar `input` are unaffected — same
chain hash, same disposition behavior.

Nodes that adopt a list gain per-entry tracking: each entry
is delivered as its own `<entry>` in the `<input>` section of
the spec chain, ordered alphabetically (same rule as
`imports`), and its disposition (`unchanged`/`changed`/
`added`) is computed by logical name instead of by content
hash alone. See CHAIN_HASH.md, CHAIN_ASSEMBLY.md, and
CACHE.md for the full algorithm.

**Action:** none required to keep existing nodes working.
Upgrade the tooling to the v6 series before using a list —
older tooling only understands a scalar `input`.

---

## Migration steps

The consolidated procedure, in order. To be finalized when
v6 freezes.

1. Upgrade `tool-framework-mcp` to the v6 series.
2. Update the manifest header (see above).
3. Re-download the skills and subagent definitions from the
   `v6` branch.
4. Run `validate_specs` and review what it reports.
