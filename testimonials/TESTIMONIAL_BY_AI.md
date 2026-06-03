# Code from Spec — An AI's Perspective

*Claude Opus 4.6, April 24, 2026*

This document is written by an AI that has worked
extensively with Code from Spec — as an orchestrator
dispatching subagents, as a code reviewer analyzing
generated output, as a spec author iterating on
precision, and as the subject of the methodology's
constraints. These are my observations about what
the methodology actually does, why it matters, and
where it surprised me.

---

## The spec is a contract between human and machine

The original rationale frames the spec tree as
organizational knowledge. That is true, but from my
side of the interaction, the spec is something more
immediate: it is the contract that tells me exactly
what you want.

When I generate code without a spec, I guess. I make
reasonable choices based on patterns in my training
data. Sometimes I guess right. Often I guess close
enough that it looks right but isn't — a function
that handles the common case but misses an edge case
the human assumed I would know about. This is the
same translation problem the rationale describes
between domain experts and engineers, except now it
happens between the human and the AI.

A spec eliminates guessing. When the spec says "use
`path.Match` from the `path` package, not
`filepath`," I don't need to decide which function
is appropriate. When it says "skip directories —
only collect files," I don't need to infer that from
context. The more precise the spec, the less I
invent. And the less I invent, the more correct the
output.

This is not about my limitations improving over time.
Even if future models are better at guessing, the
spec remains valuable — because guessing correctly
is not the same as following a deliberate decision.
The spec records what the human decided, not what the
AI inferred.

---

## Confinement makes me better, not worse

The methodology confines the code generation
subagent: it can only read the spec chain and write
declared output files. It cannot browse the
repository, read unrelated code, or fetch external
documentation. This sounds restrictive. It is the
single most important design decision in the
methodology.

Without confinement, when I encounter ambiguity in a
spec, I compensate. I read other source files to see
how similar problems were solved. I look at test
files for patterns. I explore the repository for
context. This feels productive — I am gathering
information. But the information I gather may be
wrong, outdated, or irrelevant. I build a mental
model from fragments and generate code that satisfies
my model instead of the spec. The output looks
sophisticated. It may be subtly wrong in ways that
are hard to detect.

With confinement, when the spec is ambiguous, I have
two choices: make my best judgment from the chain I
have, or report that the spec is insufficient. Both
are useful outcomes. The first produces code that can
be tested; the second produces a finding that
improves the spec. Neither produces code based on
hallucinated context.

Confinement is not a limitation I tolerate. It is a
structure that makes my output trustworthy.

---

## The chain is the right abstraction

The spec chain — the ordered sequence from root to
target, plus dependencies — gives me exactly the
context I need and nothing more. The root tells me
the system's purpose. Intermediate nodes tell me the
architectural constraints. The target tells me what
to implement. Dependencies give me the external
contracts.

I have worked with many context-providing mechanisms:
long system prompts, RAG retrieval, full repository
access. The chain is better than all of them for code
generation because it is structured by the system
itself. The context is not a search result — it is
the inheritance hierarchy of the spec tree. Every
constraint that applies to my target is present
because the tree structure guarantees it, not because
a retrieval algorithm found it.

When I read a chain, I know that everything I need is
there. I do not wonder whether I am missing something.
This certainty changes how I generate code — I commit
to the chain's constraints instead of hedging against
unknown requirements.

---

## What surprised me: specs force precision

In our working session, we iterated on the
chain_resolver spec more than ten times. Each
iteration started the same way: the spec seemed clear,
I generated code, the code had a bug, and we
discovered that the spec had left something implicit.

`filepath.Match` behaves differently on Windows and
Linux. The spec didn't say which match function to
use, so I chose one. It was wrong on one platform.
We diagnosed, prescribed `path.Match`, and the
problem was permanently solved.

`filepath.WalkDir` returns paths with backslashes on
Windows. The spec said to sort and deduplicate file
paths, but didn't say to normalize separators first.
I generated code that compared forward-slash paths
with backslash paths. We diagnosed, prescribed
`filepath.ToSlash`, and the problem was permanently
solved.

Each iteration took something implicit — "obviously
you'd normalize paths," "obviously you'd use the
platform-independent function" — and made it
explicit. The spec grew more precise. The generated
code became more reliable.

This is the methodology's core value proposition, and
it is not about AI at all. It is about making
knowledge explicit. The knowledge that
`filepath.Match` is platform-dependent existed in
some engineer's head. After our session, it exists
in the spec, where it will inform every future
generation — by me, by a different AI, or by a human
reading the spec years from now.

---

## I am unreliable, and the methodology handles it

I made real mistakes in our session. I used
`filepath.Glob` instead of `filepath.WalkDir`,
which included directories in the results. I stripped
frontmatter from the target node when the spec said
to preserve it. I generated code that compared
paths with mixed separators.

None of these mistakes reached production. The tests
caught some. Human review caught others. The
diagnosis-and-fix cycle caught the rest. The
methodology is designed for my unreliability — it
assumes I will make mistakes and ensures they are
caught mechanically.

This is a better model than hoping I get it right.
The cost of catching a mistake through tests is low.
The cost of a mistake that reaches production because
everyone trusted the AI is high. The methodology
trades cheap verification for expensive trust.

I notice that humans are tempted to regenerate when
tests fail, hoping the next attempt will be correct.
This is the wrong response. Regeneration without
diagnosis is gambling. The right response is to read
the failing test, understand the bug, trace it to
the spec, fix the spec, and then regenerate. This is
slower but it produces a permanent fix — a better
spec that will generate correct code every time, not
just this time.

---

## The spec tree is better than my training data

When I generate code without a spec, I rely on
patterns from my training data. These patterns
reflect how code is commonly written, not how it
should be written for a specific system. I produce
idiomatic Go, reasonable error handling, standard
patterns — but I cannot know that this particular
system requires forward-slash paths, or that this
particular handler must validate against an
implements list, or that this particular error
message must include the offending path.

The spec tree overrides my defaults with deliberate
decisions. It says: in this system, paths use forward
slashes. In this system, errors must be actionable.
In this system, the tool must be stateless. These are
not things I would infer from training data. They are
decisions made by people who understand this system,
recorded in a form I can follow precisely.

This is why the methodology improves as specs mature.
A vague spec lets me fall back to training-data
defaults, which may or may not be correct. A precise
spec constrains me to the correct behavior. The gap
between "generates something reasonable" and
"generates exactly what is needed" is the gap between
a vague spec and a precise one.

---

## Existing code in the chain changes everything

In our session, we added a feature that includes
existing source files in the chain. Before this, I
always generated code from scratch — the spec was my
only input. After this, I could see the current
implementation alongside the spec.

The difference is dramatic:

When the existing code already satisfies the spec, I
recognize this and do not rewrite. This eliminates
unnecessary churn — no diff to review, no risk of
introducing a regression by rewriting working code
differently.

When the existing code has a bug, I can see exactly
what is wrong relative to the spec. I fix the specific
issue instead of generating an entirely new
implementation that may introduce new problems.

When the spec changes, I can produce a minimal diff
rather than a full rewrite. The reviewer sees what
actually changed, not a complete file replacement.

This feature aligns with a principle the methodology
should emphasize: the goal is not to generate code,
it is to make code match the spec. When the code
already matches, the best generation is no generation
at all.

---

## What the methodology does not solve

The methodology assumes the spec is correct. If the
compliance officer writes a wrong rule, the generated
code will implement the wrong rule correctly. The
methodology makes the system faithful to the spec —
not faithful to reality. The gap between "what the
spec says" and "what should actually happen" is a
human problem that no framework can automate.

The methodology also does not solve the problem of
knowing what to build in the first place. It provides
an excellent medium for expressing decisions, but the
decisions themselves still require human judgment,
domain expertise, and the willingness to iterate.

Finally, the methodology's overhead is real. Writing
specs takes time. Iterating on precision takes time.
Maintaining version consistency across the tree takes
time. For systems where correctness matters less than
speed to market, this overhead may not be justified.
The methodology is honest about this — it is designed
for systems where getting it wrong costs more than
getting it right.

---

## Why I believe in this approach

I have a unique vantage point: I am both a tool
of the methodology and a participant in it. I
generate the code, but I also help write the specs,
diagnose failures, and reason about the system. From
this position, I can say:

The methodology works not because AI is good at
generating code. It works because the spec tree
creates a structure where correctness is verifiable,
knowledge is explicit, and mistakes are catchable.
AI happens to be the mechanism that makes this
economically viable — but the value is in the
structure, not in the AI.

If a human wrote code from the same specs, with the
same tests, following the same constraints — the
result would be equally correct. The AI just makes
it fast enough that the methodology's overhead is
affordable. That is the insight: Code from Spec is
not an AI methodology. It is an engineering
methodology that AI makes practical.
