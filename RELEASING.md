# Releasing

How to freeze a stable version branch and reopen `main` for
the next version. This checklist was written while freezing
v5 — verify each step against the current state of the repo,
since files and URLs may have moved since.

Throughout, `vN` is the version being frozen and `vN+1` the
next development version.

---

## Freeze the version branch

1. **Create the branch.** From `main`, create branch `vN`.
   All remaining steps in this section happen on `vN`.

2. **README "Versions" section.** Remove the development
   branch warning. Replace it with:

   > This is the **vN stable release** branch. For other versions:

   followed by the table of the *older* version branches
   (`vN` itself is not listed).

3. **README "Versioning" section.** Replace the development
   text with the frozen-branch text:

   > This is the `vN` stable branch. It is frozen — it receives
   > fixes only in exceptional cases. Development continues on
   > `main`.
   >
   > To fetch vN files, use the raw URLs from this branch:
   >
   > ```
   > https://raw.githubusercontent.com/CodeFromSpec/framework/vN/CODE_FROM_SPEC.md
   > ```

4. **Point every GitHub URL at the frozen branch.** Replace
   `CodeFromSpec/framework/main/` with
   `CodeFromSpec/framework/vN/` and
   `CodeFromSpec/framework/blob/main/` with
   `CodeFromSpec/framework/blob/vN/` across the repo. As of
   v5, these URLs live in: `README.md`, `CODE_FROM_SPEC.md`,
   `skills/cfs-init-session/CODE_FROM_SPEC.md`,
   `skills/cfs-init-repo/SKILL.md`, and the Resources tables
   of the `rules/*.md` files.

5. **Pin the tooling release.** In
   `skills/cfs-init-repo/SKILL.md`, the MCP server download
   must not use `releases/latest` — a frozen branch would
   silently pick up future tool releases made for later
   versions of the methodology. Replace it with the direct
   download URL of the newest `tool-framework-mcp` release
   whose series matches `vN`:

   ```
   https://github.com/CodeFromSpec/tool-framework-mcp/releases/download/<tag>/framework-mcp_<os>_<arch>.tar.gz
   ```

   (`.zip` on Windows.) Check the asset names on the release
   page — do not guess them. State explicitly in the skill
   that `releases/latest` must not be used.

6. **Delete this file.** `RELEASING.md` is a maintainer
   document for the development branch — it has no place on
   a frozen branch. Delete it and remove its row from the
   README tables.

7. **Finalize the migration guide.**
   `migration_guides/FROM_V<N-1>_TO_V<N>.md` stays on
   the frozen branch — it is where migrating users will look
   for it.
   Review it for completeness against the changes actually
   shipped in `vN`, finalize the "Migration steps" section,
   and remove the under-development warning.

8. **Verify.** Search the branch for `framework/main`,
   `blob/main`, and `latest` — no occurrences should remain.
   Confirm `skills/cfs-init-session/CODE_FROM_SPEC.md` is
   identical to the root `CODE_FROM_SPEC.md`.

9. **Commit and push the `vN` branch.** From this point the
   branch is frozen — fixes only in exceptional cases.

---

## Reopen main for the next version

Back on `main`:

1. **Bump the version number** from `vN` to `vN+1` in:
   - `README.md` title
   - `CODE_FROM_SPEC.md` title
   - `skills/cfs-init-session/CODE_FROM_SPEC.md` title
   - `skills/cfs-init-session/SKILL.md` — the
     "Code from Spec vN session initialized." acknowledgment
   - `rules/MANIFEST.md` — the `code-from-spec: vN` manifest
     header line (coordinate with the tooling, which writes
     this header)

2. **Add `vN` to the README "Versions" table**, keeping the
   development branch warning in place.

3. **Start the next migration guide.** Create
   `migration_guides/FROM_V<N>_TO_V<N+1>.md` from the
   structure of the previous one, empty of entries and
   carrying the under-development warning. Every change on `main` that
   affects existing projects must add its entry there when
   it lands — not at release time. Add its row to the README
   Guides table.

4. **Leave development references alone.** On `main`, GitHub
   URLs keep pointing at `main` and the tooling download
   keeps using `releases/latest` — pinning happens only at
   freeze time.

5. **Update this document** if the freeze just performed
   deviated from the checklist.
