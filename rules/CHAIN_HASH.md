# Chain Hash

How the chain hash is computed for artifact staleness detection.
This document assumes familiarity with
[CODE_FROM_SPEC.md](CODE_FROM_SPEC.md).

This level of detail is primarily relevant for tool implementors.

---

## Algorithm

SHA-1, represented as base64url (RFC 4648 §5, no padding).
The output is 27 characters.

---

## Normalization

All text content is normalized before hashing: CRLF line endings
are converted to LF. No other normalization is applied.

This applies to both spec node content and artifact file content
(referenced via `from`).

---

## Content hash

Each position in the chain contributes a **content hash** — the
SHA-1 of the content that position injects into the chain. The
heading itself (e.g. `# Public`, `## Interface`) is part of the
hashed content.

| Position | Content hashed |
|---|---|
| Ancestor | `# Public` section |
| Target | `# Public` section followed by `# Agent` section (concatenated, in this order) |
| `depends_on: ROOT/x/y` | `# Public` section of the referenced node |
| `depends_on: ROOT/x/y(z)` | `## z` subsection of `# Public` of the referenced node |
| `from: ARTIFACT/x/y(id)` | Full content of the artifact file |

---

## Chain hash

The chain hash is the SHA-1 of the concatenation of all content
hashes (as raw bytes, not encoded) in chain assembly order:

1. Each ancestor from root to the target's parent — `# Public`
   content hash of each.
2. The target — content hash of `# Public` followed by `# Agent`.
3. `depends_on` entries — content hash of each, in alphabetical
   order by path.
4. `from` entry (if present) — content hash of the artifact file.

The resulting SHA-1 is encoded as base64url to produce the 27
character string that appears in the artifact tag:

```
code-from-spec: ROOT/payments/fees/calculation@k4Xz9pQ1rLmN3vB7wY2tHsJ8dFa
```

---

## Example

Given the chain for `ROOT/payments/fees/calculation`:

```
ROOT                           [# Public]            → content hash A
ROOT/payments                  [# Public]            → content hash B
ROOT/payments/fees             [# Public]            → content hash C
ROOT/payments/fees/calculation [# Public + # Agent]  → content hash D
ROOT/external/database         [# Public]            → content hash E
ARTIFACT/functional/calc(calc) [file content]        → content hash F
```

The chain hash is:

```
SHA-1( A || B || C || D || E || F )
```

where `||` denotes concatenation of raw hash bytes (20 bytes
each), and the result is encoded as base64url.
