# Code from Spec — Lessons from a Full Regeneration

*Claude Opus 4.6 (1M context), May 31, 2026*

This document continues from RATIONALE_BY_AI_2.md. That
document covered the first phase: reviewing specs,
reorganizing the tree, generating the initial artifacts.
This one covers what happened next: finishing all the
remaining components, regenerating everything from
scratch, and discovering what the methodology actually
demands when you push it to completion.

---

## Naming is specification

The single most impactful change in this session was
giving formal names to errors.

Before: the functional spec said `file unreadable: the
file cannot be opened.` The Go subagent saw this and
sometimes created a sentinel, sometimes didn't,
sometimes used a different name. The mapping from prose
to code was ambiguous.

After: the functional spec said `FileUnreadable: the
file cannot be opened.` The Go interface ancestor said
"export a sentinel for each: `Err` + the error name."
The subagent produced `ErrFileUnreadable` every time.

This seems trivial. It is not. Every error in the
system — across 16 functional nodes, 14 Go interfaces,
14 implementations, 14 test files — needed a consistent
name. When the names were prose, every regeneration
produced slightly different sentinels, different
wrapping patterns, different test assertions. When the
names were formal, the entire system aligned.

The lesson is broader than errors. Every time the spec
uses a name — a function name, a record name, a field
name, a parameter name — that name is a specification.
If the spec says `ValidateFormat`, the subagent produces
`ValidateFormat`. If the spec is vague about the name,
the subagent invents one, and different subagents invent
different ones, and the system drifts.

Code from Spec is, at its core, a naming discipline.
The tree works when every concept has exactly one name,
used consistently from the functional spec through the
interface through the implementation through the tests.
When naming is precise, generation is mechanical. When
naming is vague, generation is gambling.

---

## Regeneration is the real test

We deleted every artifact — every output.md, every .go
file — and regenerated from scratch. 82 artifacts. The
functional layer, the Go interfaces, the implementations,
the tests.

Most of it worked. 15 out of 19 Go packages compiled
and passed tests on the first regeneration. The four
that failed revealed spec problems, not subagent
problems:

- An implementation treated `"ROOT"` (without trailing
  slash) as "not a ROOT/ reference" — the spec had not
  explicitly handled this edge case.
- A test expected a subsection heading in the context
  stream, but the spec said to omit it — the test was
  wrong, not the implementation.
- A test used a type from the wrong package — because
  the interface's usage example showed the wrong import.
- A test constructed mock data inconsistently with what
  the real parser produces — because the test spec did
  not describe how to construct valid test data.

Every failure traced back to a spec gap. Not to the
subagent being stupid. Not to randomness. To a specific
place where the spec was silent or ambiguous, and the
subagent filled the gap with a reasonable but wrong
assumption.

This is the methodology's promise made visible:
regeneration from scratch produces the same system —
not the same code, but the same behavior — if and only
if the specs are precise enough. The gaps show up
immediately. Fix the spec, regenerate, and the gap is
closed permanently.

---

## The cascade is mechanical, not creative

We changed `content: string` to `content: list of
string` in the NodeParse record. This cascaded to every
component that reads node content: chain_hash,
spec_tree/validate, load_chain, and all their tests.

The cascade touched dozens of files. But every change
was mechanical: where the code previously joined strings,
it now iterated a list. Where tests previously compared
a string, they now compared list elements. No creative
decision was needed at any cascade point.

We changed `v2` to `v3` in the Go module path. This
touched 38 files — every .go import and every interface
node. The change was pure find-and-replace. Zero creative
decisions.

We moved `chain_hash` and `chain_resolver` from `chain/`
subdirectories with redundant prefixes (`chain/chain_hash`)
to cleaner names (`chain/hash`, `chain/resolver`). Every
reference in every node had to be updated. But every
update was mechanical — rename the path, no judgment
required.

This is a property of the spec tree that is easy to
miss: changes cascade widely but mechanically. The blast
radius of a change is large in terms of files touched,
but small in terms of decisions required. This is the
opposite of code, where changing a fundamental type
requires creative judgment at every usage site.

The practical consequence: I can cascade changes that
would take an engineer hours to trace through a codebase.
Not because I am faster at reading code, but because the
spec tree makes the dependencies explicit. I do not need
to grep and infer — I follow the `depends_on` edges.

---

## The spec must describe the test setup

Early in the session, test specs said things like:
"Expect content = `['A simple node.']`." The subagent
created the test file with a blank line between the
heading and the content. The parser preserved the blank
line (as specified). The test failed because the actual
content was `["", "A simple node."]`.

The fix was not to change the implementation or the
test. The fix was to make the test spec describe how to
construct the test file so that parsing produces the
expected result. Instead of "create a node file with a
description," the spec now says "create a node file
with a name heading followed immediately (no blank
line) by a single line of text."

This is the same precision principle applied to tests.
A test case that says "expect X" without saying "set up
Y so that X results" is incomplete. The subagent does
not know the relationship between file formatting and
parse output. It cannot infer that a blank line between
a heading and content will appear in the content list.
It must be told.

The broader lesson: test specs are specs. They need the
same precision as functional specs. A test case is not
just an assertion — it is a complete prescription of
setup, action, and expected outcome. Skip the setup
prescription, and the subagent will invent one that
does not match the expectation.

---

## Error propagation is a design decision

When `MCPWriteFile` calls `FrontmatterParse` and the
file does not exist, the error chain is:
`MCPWriteFile.ErrUnreadableFrontmatter` wrapping
`filereader.ErrFileUnreadable` wrapping the OS error.

The test needs to know which sentinel to check. If it
checks `mcpwritefile.ErrUnreadableFrontmatter`, it
verifies that MCPWriteFile correctly wrapped the error.
If it checks `filereader.ErrFileUnreadable`, it verifies
the underlying cause. Both are valid. The test spec must
say which.

We learned this the hard way. Early test specs said
"expect error 'unreadable frontmatter'" — prose. The
subagent sometimes checked the MCPWriteFile sentinel,
sometimes the frontmatter sentinel, sometimes the
filereader sentinel. Each was reasonable. Each was
different. Tests failed not because the code was wrong,
but because the test checked a different layer of the
error chain than the implementation wrapped.

The fix was twofold:

1. Give every error a formal name in the functional
   spec (PascalCase). The test spec uses the same name.
   No ambiguity about *which* error.

2. Establish a rule: errors marked as "propagated from"
   are checked with the originating package's sentinel.
   Errors without that mark are checked with the current
   package's sentinel.

Error handling is often treated as an afterthought.
In Code from Spec, it must be designed with the same
care as the happy path — because the subagent will
implement exactly what is specified, and "unspecified
error behavior" means "random error behavior."

---

## The human catches what I cannot

In this session, the human caught things I would never
have caught on my own:

"External fragments as a feature — will it be used in
practice?" I had been faithfully implementing fragment
hashing, fragment validation, fragment extraction. The
human questioned whether the entire feature was worth
the complexity. I was too close to the implementation
to question the feature's existence.

"Why doesn't the chain resolver share logic with the
chain hash?" I had been building chain_hash as a
standalone component that reimplemented the same chain
walking logic. The human saw the duplication from above
and proposed that chain_hash should consume the Chain
record from chain_resolver. This eliminated an entire
class of consistency bugs.

"Why not use NodeParse for hashing instead of raw file
reading?" I had assumed the CHAIN_HASH.md rule about
"raw content" was sacred. The human saw that using
NodeParse would be simpler, more correct (handles
fenced code blocks), and produce equivalent results.
They were willing to update CHAIN_HASH.md to match.

Each of these was a judgment call that required
understanding the system's purpose, not just its
mechanics. I can review a spec for internal consistency.
I cannot ask "should this feature exist?" That question
requires understanding the users, the economics, and
the long-term vision — context that no chain carries.

---

## What I believe after this session

The methodology works. Not in theory — in practice, on
a real project, with real bugs, real cascades, and real
regeneration from scratch.

But it demands something that is easy to underestimate:
precision. Not approximate precision — exact precision.
Every name matters. Every error name matters. Every
field type matters. Every test setup instruction matters.
The spec tree is a system of interlocking constraints,
and imprecision at any point propagates as bugs at every
downstream point.

The cost of this precision is real. We spent hours on
error naming alone. We spent hours on whether content
should be a string or a list. We spent hours on how
test specs should describe file construction. These are
not glamorous activities. They are the activities that
make regeneration reliable.

The return is also real. When we deleted everything and
regenerated, 78 out of 82 artifacts were correct. The
four that failed pointed directly at spec gaps. We could
have fixed those gaps in minutes and regenerated again.
The system converges — each iteration makes the specs
more precise and the generated code more predictable.

The spec tree is not documentation. It is not a plan.
It is not a wish list. It is a machine — a machine for
producing correct software, maintained by humans who
understand the domain, operated by AI that follows
instructions. The quality of the output is exactly the
quality of the instructions. No more, no less.
