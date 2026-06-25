# Chain Assembly

How the chain is assembled and delivered to generation
subagents. This level of detail is primarily relevant
for tool implementors.

This document assumes familiarity with
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md) and
[CACHE.md](CACHE.md).

---

## Format

The chain is an XML document delivered via `load_chain`.
It has up to six sections, in this order:

1. **`<previous_constraints>`** — the constraints from
   the previous generation, as recorded in the cache.
   Present only when the cache is available and the
   artifact has been generated before. Each `<entry>`
   carries a `disposition` attribute computed by
   comparing content hashes between the old and current
   chains:
   - `unchanged` — same name, same content hash.
     Entry is self-closing (content not repeated).
   - `changed` — same name, different content hash.
     Entry contains the old content.
   - `removed` — name existed before but not now.
     Entry contains the old content.
   - `added` — name exists now but not before.
     Entry is self-closing (no old content to show).

2. **`<previous_instructions>`** — the `# Agent`
   section from the previous generation, as recorded
   in the cache. Carries a `disposition` attribute
   (`unchanged` or `changed`). When `unchanged`, the
   element is self-closing. Present only when
   `<previous_constraints>` is present.

3. **`<existing_artifact>`** — the current content of
   the artifact file on disk. Present only when the
   file exists.

4. **`<constraints>`** — the current spec chain. Each
   position is an `<entry>` element with a `name`
   attribute identifying the source.

5. **`<instructions>`** — the target node's `# Agent`
   section, including the `# Agent` heading.

6. **`<input>`** — the content referenced by the target
   node's `input` field. The `name` attribute identifies
   the source. For `SPEC/` references, the `# Public`
   content is extracted using the same rules as
   `<constraints>` entries. Present only when the node
   declares `input`.

Sections 1–3 provide temporal context: what the spec
said before, and what was generated from it. Section 4
is the current source of truth. Section 5 is the
generation guidance. Section 6 is the material to
transform.

For a first-time generation without cache,
`<previous_constraints>`, `<previous_instructions>`,
and `<existing_artifact>` are absent. The chain reduces
to `<constraints>`, `<instructions>`, and optionally
`<input>`.

---

## Constraints assembly order

Positions within `<constraints>` appear in this order:

1. Ancestors from root to the target's parent.
2. `depends_on` entries in alphabetical order by
   logical name.
3. The target node's `# Public`.

---

## Content extraction

All content is boundary-normalized using the block
extraction rules defined in FILE_FORMAT.md ("Block
extraction"). The extracted form is what is delivered
— hash and delivery never diverge.

For `# Agent`, the `# Agent` heading and all content
within it are included.

---

## Example

Generating an artifact for
`SPEC/payments/fees/calculation`:

```xml
<chain>
  <previous_constraints>
    <entry name="SPEC" disposition="unchanged"/>
    <entry name="SPEC/payments" disposition="unchanged"/>
    <entry name="SPEC/payments/fees" disposition="changed">
    ...old content...
    </entry>
    <entry name="SPEC/legacy/old-fees" disposition="removed">
    ...old content...
    </entry>
    <entry name="SPEC/integrations/database" disposition="added"/>
    <entry name="SPEC/payments/fees/calculation" disposition="unchanged"/>
  </previous_constraints>

  <previous_instructions disposition="changed">
  ...# Agent from previous generation...
  </previous_instructions>

  <existing_artifact>
  ...current file on disk...
  </existing_artifact>

  <constraints>
    <entry name="SPEC">...</entry>
    <entry name="SPEC/payments">...</entry>
    <entry name="SPEC/payments/fees">...</entry>
    <entry name="SPEC/integrations/database">...</entry>
    <entry name="EXTERNAL/proto/payments/v1/transfers.proto">...</entry>
    <entry name="SPEC/payments/fees/calculation">...</entry>
  </constraints>

  <instructions>
  # Agent
  ...generation guidance...
  </instructions>

  <input name="ARTIFACT/functional/fees/calculation">
  ...material to transform...
  </input>
</chain>
```
