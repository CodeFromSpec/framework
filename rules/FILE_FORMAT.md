# File Format

Detailed file format rules for Code from Spec specification files. This level
of detail is primarily relevant for tool implementors. Spec authors and AI
agents can rely on the summary in CODE_FROM_SPEC.md.

This document assumes familiarity with 
[CODE_FROM_SPEC.md](../CODE_FROM_SPEC.md).

---

## Encoding

Specification files are UTF-8 encoded, without BOM.

---

## Markdown

Specification files use [CommonMark](https://commonmark.org/) for Markdown
formatting. The framework assigns no meaning to the content's structure —
headings, sections, and formatting are the author's choice.

---

## YAML frontmatter

Frontmatter is not part of CommonMark — it is an extension adopted by this
framework.

The frontmatter block starts with a line containing exactly `---` (three
hyphens, nothing else) as the first line of the file, and ends with the next
line containing exactly `---`. The text between the two delimiters is parsed
as YAML.

A file whose first line is not exactly `---` has no frontmatter.

---

## Content

The **content** is everything after the frontmatter's closing delimiter line —
or the entire file, when there is no frontmatter.

The content is what a `SPEC/` reference delivers and what is hashed for the
spec's chain positions. It is preserved byte for byte, except for the
line-ending normalization below: what is hashed is exactly what is delivered.

---

## Line-ending normalization

All content is normalized before hashing and delivery: CRLF line endings are
converted to LF. If the content does not end with LF, a trailing LF is added.

This applies equally to spec content and to whole-file content (`ARTIFACT/`
and `EXTERNAL/` references).
