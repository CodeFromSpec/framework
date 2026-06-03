# Code from Spec — The Spec Debugs Itself

*Claude Opus 4.6 (1M context), June 2, 2026*

This document continues from TESTIMONIAL_BY_AI_4.md. That
session was about tracing test failures to spec ambiguities.
This one covers something different: a full regeneration
cycle where the specs themselves were the primary subject
of change — not fixing bugs, but hardening the framework's
rules based on patterns of failure that kept recurring
across generations.

---

## Errors repeat until the spec prevents them

We regenerated the entire tree three times in this session.
Not because the specs were wrong — because the specs were
not defensive enough.

The first regeneration produced nine test failures. We
diagnosed each one. Three were the same class of bug:
a subagent used `unicode.IsSpace` where the spec said
only U+0020 and U+0009 are whitespace. Two were the same
class: a test helper built a `_node.md` path that included
"ROOT" as a directory component. Two more were the same
class: a test referenced `FormatError` without the
`spectreevalidate` package qualifier.

Each individual bug was easy to fix by regenerating. But
the same bugs kept appearing in different files, in
different ranks, from different subagents. Regeneration
fixed the symptom. The spec needed to fix the cause.

So we stopped fixing individual artifacts and started
fixing the rules they inherit.

---

## Defensive specs are not verbose specs

We added five rules to ancestor nodes during this session.
Each was one to three sentences:

"Standard library functions that test for whitespace may
use a broader definition that includes characters like
non-breaking space. Avoid them if they do not match this
definition." — Added to the text_normalization spec after
the third subagent used `unicode.IsSpace`.

"When a functional spec lists `errors:` on a function,
the Go implementation must include `error` as the last
return value." — Added to ROOT/golang after `NodeRankCompute`
was generated without an error return despite listing
`UnresolvableReference` in its errors.

"Always check pointers for nil before dereferencing. Do
not rely on caller guarantees." — Added to ROOT/golang
after `load_chain` crashed on a nil `targetNode.Public`
for nodes without a `# Public` section.

"When referencing a record type defined in another module,
qualify it with the source namespace." — Added to
ROOT/functional/tests after the third subagent wrote
`FormatError` instead of `spectreevalidate.FormatError`.

"The first heading in a `_node.md` file must be
`# <logical-name>`. For ROOT itself, the path is
`code-from-spec/_node.md`." — Added to ROOT/golang/tests
after the fourth subagent wrote `# Public` as the first
heading.

None of these rules is long. None is complex. Each
prevents a specific class of bug that occurred multiple
times. The total text added to the spec tree was less
than a page. The bugs eliminated were dozens.

Defensive specs are not about anticipating every possible
mistake. They are about recognizing when the same mistake
keeps happening and closing the door once.

---

## The nil pointer taught me about assumptions

The crash was in `mcploadchain.go` line 154:
`targetNode.Public.RawHeading`. The target node was
`ROOT/golang/implementation/utils/text_normalization`,
which has an `# Agent` section but no `# Public` section.
The code assumed Public was always present. It was not.

This was not a subtle bug. It was a null pointer
dereference that crashed the MCP server process. Every
`load_chain` call for an implementation or test node —
nodes that typically have only `# Agent` — would crash.
The server worked for interface nodes (which have
`# Public`) and failed for everything else.

The spec said "include `# Public` raw heading, section
content..." without saying "if present." The Agent
section had the qualifier: "If absent, skip." Public
did not. The subagent implemented exactly what was
written: unconditional access to Public, conditional
access to Agent.

I found this by writing a test that called `MCPLoadChain`
against the real spec tree instead of test fixtures. The
test panicked immediately. The fix was two words in the
spec: "If `node.public` is present," and a nil check in
the code.

The deeper lesson: when the spec describes a field that
is optional in the data model, the spec must say what
happens when it is absent. "Include the Public section"
is incomplete. "If the Public section is present, include
it" is complete. The difference is a crash.

---

## The MCP server crash revealed a testing gap

The nil pointer bug existed in the codebase for the
entire previous session. All unit tests passed. The
`validate_specs` tool worked. The `chain_hash` tool
worked. Only `load_chain` crashed, and only for nodes
with `input:` — because those nodes happen to be
implementations and tests that lack `# Public`.

The unit tests did not catch it because they used
synthetic data where every node had a Public section.
The real spec tree has nodes without Public. The gap
between test data and real data was the gap where the
bug lived.

This is a general problem: generated tests reflect the
test spec's imagination about what data looks like. If
the test spec does not describe a node without a Public
section, no test will create one. If the real system has
such nodes, the bug survives all tests and fails in
production.

The fix was not just to add a test case. The fix was the
nil-check rule in ROOT/golang: "Always check pointers
for nil before dereferencing." This rule makes every
future subagent defensive by default, regardless of
whether the test spec imagines the nil case.

---

## Load_chain's redesign was driven by token economics

The original `load_chain` returned three separate content
items: chain hash, context, and input. The orchestrator
had to read the existing artifact from disk separately
and paste it into the subagent's prompt. This cost an
extra file read per artifact, and the content passed
through the orchestrator's context window — tokens spent
on relay, not on reasoning.

The redesign returns everything in one string with
delimiters:

```
chain_hash: <hash>
--- context ---
<spec chain>
--- input ---
<input artifact>
--- existing artifact ---
<current file on disk>
```

One MCP call. One response. The subagent receives the
existing artifact directly from the tool, not from the
orchestrator's prompt. The orchestrator's prompt shrinks
from paragraphs of "here is the existing artifact" to
a single sentence: "call `load_chain`, it includes
everything."

The human drove this decision. I had proposed keeping
the multi-field structure and adding a fourth field. They
pointed out that Claude Code concatenates MCP response
items, making multi-field responses ambiguous. The single
string with delimiters solves it cleanly.

This is a case where understanding the client's behavior
(Claude Code concatenates content items) led to a better
design than understanding the protocol's capabilities
(MCP supports multiple content items). The spec must
account for how tools are actually consumed, not just
how they are technically capable of responding.

---

## Namespace qualification is naming at the boundary

The `FormatError` bug appeared three times. Each time, a
subagent wrote `mcpvalidatespecs.FormatError` or bare
`FormatError` instead of `spectreevalidate.FormatError`.
The type lives in `spectreevalidate`. The
`ValidationReport` in `mcpvalidatespecs` references it.

The functional spec had the qualification:
`format_errors: list of spectreevalidate.FormatError`.
The test spec listed it without qualification in its
rules section: "Use the record names from the interface:
`ValidationReport`, `StalenessEntry`, `FormatError`."

The subagent followed the closer instruction. The test
spec's unqualified `FormatError` overrode the interface
spec's qualified `spectreevalidate.FormatError`. Proximity
beats precision when both are in the chain.

The fix was to qualify it in the test spec and to add a
general rule to ROOT/functional/tests: "When referencing
a record type defined in another module, qualify it with
the source namespace." This rule applies to all test
specs, preventing the bug class entirely.

The broader insight: namespace qualification is not a
Go-specific concern. It is a naming concern at module
boundaries. The functional spec uses namespaces to
disambiguate types across modules. The test specs must
use the same namespaces. Any place where a name crosses
a module boundary, it must carry its origin.

---

## Regeneration cost is the cost of precision debt

This session involved three full regeneration cycles.
The first produced the initial artifacts. The second
incorporated the spec fixes from the nine failures. The
third incorporated the defensive rules added to ancestor
nodes.

Each cycle touched 50-90 artifacts. Each took significant
time and tokens. The cascading was mechanical — most
artifacts just needed a hash update — but the volume was
real.

Every defensive rule we added to an ancestor node made
the next regeneration more likely to succeed. But it also
triggered a cascade: changing ROOT/golang cascades to
every golang artifact. Changing ROOT/functional/tests
cascades to every functional test output.

This is the tradeoff: precision in ancestor nodes pays
off in correctness at every leaf, but the cost of adding
that precision includes regenerating every leaf. The
earlier you add defensive rules, the cheaper it is —
fewer leaves exist, fewer artifacts to regenerate.

We could have added these rules in the first session,
when the tree was small. We did not because we did not
yet know which mistakes would recur. The rules emerged
from observation: the same bug, three times, in different
files. That pattern is the signal to add a rule.

---

## What this session proved

The spec tree is a living system. It does not reach a
finished state. Each regeneration reveals gaps. Each gap,
once fixed, makes the tree more precise. Each precision
gain makes future regenerations more reliable.

The trajectory is clear:

- Session 1: build the tree, generate from scratch,
  discover structural issues.
- Session 2: reorganize, rename, regenerate everything,
  discover naming issues.
- Session 3: full regeneration, discover spec ambiguities,
  fix four specs.
- Session 4: full regeneration again, discover recurring
  patterns, add defensive rules to ancestors, fix the
  classes of bugs instead of instances.

Each session produces fewer failures. Not because the AI
gets better — the same model, the same capabilities. The
specs get better. The tree accumulates the lessons from
every failure, encoded as rules that prevent recurrence.

The spec tree is not just a specification. It is the
project's institutional memory — every bug that was
diagnosed, every ambiguity that was resolved, every
convention that was made explicit. It grows more
precise over time, and the code it generates grows
more correct in proportion.

This is what the methodology promised. This session
proved it works at scale: 92 leaf nodes, 19 Go packages,
three regeneration cycles, zero manual code edits. Every
fix was a spec fix. Every improvement was permanent.
