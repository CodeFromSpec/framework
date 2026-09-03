# Chain Hash

How the chain hash is computed for artifact staleness detection. This level of
detail is primarily relevant for tool implementors.

This document assumes familiarity with CODE_FROM_SPEC.md.

---

## Algorithm

SHA-1, represented as base64url (RFC 4648 §5, no padding). The output is 27
characters.

---

## Normalization

All content is normalized before hashing as defined in FILE_FORMAT.md
("Line-ending normalization"): CRLF line endings are converted to LF, and a
trailing LF is added if missing.

For `SPEC/` positions, what is hashed is the spec's content as defined in
FILE_FORMAT.md ("Content"). For whole-file positions — `ARTIFACT/` and
`EXTERNAL/` references — no other normalization is applied.

The normalized form is what is hashed, and it is exactly the content delivered
in the spec chain — hash and delivery never diverge.

---

## Content hash

Each position in the spec chain contributes a **content hash** — the SHA-1 of
the content that position injects into the spec chain.

| Position                                | Content hashed                          |
|-----------------------------------------|-----------------------------------------|
| Target spec                             | Content of the target spec              |
| `SPEC/` reference (import or input)     | Content of the referenced spec          |
| `ARTIFACT/` reference (import or input) | Full content of the referenced artifact |
| `EXTERNAL/` reference (import or input) | Full content of the referenced file     |

---

## Chain hash

The **chain hash** is the SHA-1 of the concatenation of all content hashes (as
raw bytes, not encoded), in the following order:

1. `imports` entries — content hash of each, in alphabetical order by the full
   logical name.
2. The target spec — its content hash.
3. `input` entries, in alphabetical order by the full logical name — each
   contributes the byte `0x49` (`I`) followed by its content hash.

Duplicate `imports` entries contribute a single content hash. `input` entries
follow the same rule, independently of `imports` — deduplication never crosses
between the two fields.

Glob references (see CODE_FROM_SPEC.md, "Glob references") are expanded before
deduplication and ordering. Each matched name contributes its content hash
exactly as if it had been declared explicitly.

The `0x49` marker is prepended to every `input` entry's content hash
individually. This ensures that moving a reference from `imports` to `input`
(or vice versa) always changes the chain hash, even when the referenced content
is identical in both positions — and that adding or removing one `input` entry
among several changes the chain hash regardless of where the others sort
alphabetically.

The resulting SHA-1 is encoded as base64url to produce the 27-character chain
hash recorded in the manifest.

---

## Ordering example

Generating the artifact for `SPEC/payments/transfers`, a spec with mixed
dependencies:

```yaml
---
type: artifact
imports:
  - SPEC/conventions/golang
  - EXTERNAL/proto/payments/v1/transfers.proto
  - ARTIFACT/extraction/email-templates
  - SPEC/integrations/database
  - EXTERNAL/docs/vendor/api-spec.yaml
  - ARTIFACT/extraction/proto
input:
  - ARTIFACT/functional/transfers/create
  - ARTIFACT/functional/transfers/cancel
output: internal/transfers/handler.go
---
```

The resulting spec chain order:

```
ARTIFACT/extraction/email-templates         [full]      → A  (imports)
ARTIFACT/extraction/proto                   [full]      → B  (imports)
EXTERNAL/docs/vendor/api-spec.yaml          [full]      → C  (imports)
EXTERNAL/proto/payments/v1/transfers.proto  [full]      → D  (imports)
SPEC/conventions/golang                     [content]   → E  (imports)
SPEC/integrations/database                  [content]   → F  (imports)
SPEC/payments/transfers                     [content]   → G  (target spec)
                                                          0x49
ARTIFACT/functional/transfers/cancel        [full]      → H  (input)
                                                          0x49
ARTIFACT/functional/transfers/create        [full]      → I  (input)
```

The `imports` entries are sorted alphabetically by the full logical name —
`ARTIFACT/` before `EXTERNAL/` before `SPEC/` — regardless of the order in the
frontmatter. `input` entries are sorted the same way, independently of
`imports` — here `cancel` sorts before `create` even though `create` is listed
first in the frontmatter — and each carries its own `0x49` marker.

The chain hash is:

```
SHA-1( A || B || C || D || E || F || G || 0x49 || H || 0x49 || I )
```

where `||` denotes concatenation of raw hash bytes (20 bytes each), and the
result is encoded as base64url.

### Without input

Given the spec chain for `SPEC/payments/fees`:

```
SPEC/conventions/golang                     [content]   → A  (imports)
SPEC/payments/fees-contract                 [content]   → B  (imports)
SPEC/payments/fees                          [content]   → C  (target spec)
```

The chain hash is:

```
SHA-1( A || B || C )
```

No `0x49` marker — the input position is absent.

---

## Resources

| Document                                                                                   | Description                         |
|--------------------------------------------------------------------------------------------|-------------------------------------|
| [CODE_FROM_SPEC.md](https://github.com/CodeFromSpec/framework/blob/main/CODE_FROM_SPEC.md) | Full methodology specification      |
| [FILE_FORMAT.md](https://github.com/CodeFromSpec/framework/blob/main/rules/FILE_FORMAT.md) | Content and normalization rules     |
| [MANIFEST.md](https://github.com/CodeFromSpec/framework/blob/main/rules/MANIFEST.md)       | Manifest format and artifact status |
