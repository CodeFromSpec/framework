# Historical Background

How spec-driven development evolved from a promising idea
to an abandoned practice to a practical methodology — and
why AI is the missing piece that makes it work.

---

## The problem software has always had

Software is written by people who hold context in their
heads. The engineer receives requirements, translates
them into code, and in that translation makes hundreds of
small decisions that are never recorded anywhere. When
the engineer leaves, the decisions leave too. What
remains is code that works — until it doesn't — and that
nobody fully understands anymore.

Code is not a good medium for expressing intent. It
expresses mechanism. You can read code and understand
what it does; you cannot read it and understand why it
does that, what alternatives were considered, or what
constraints it is silently respecting.

The industry built compensating mechanisms: comments,
wikis, ADRs, runbooks, onboarding docs. None of them
work at scale because they are separate from the system.
They describe a system that exists independently. As the
system changes, they drift. Eventually they describe a
system that no longer exists, and the team stops trusting
them. The knowledge returns to people's heads, and the
cycle repeats.

---

## The translation problem

Every organization that builds software has two kinds of
knowledge: domain knowledge (compliance officers,
accountants, legal teams, product managers) and technical
knowledge (engineers who know how to build systems).

Traditionally, the engineer was the only author of
software. They received domain knowledge as
requirements — documents, meetings, conversations — and
translated it into code. This translation was lossy.
Every handoff lost information. Every interpretation
introduced error. The more intermediaries between the
domain expert and the code, the more the implementation
drifted from the original intent.

The result was systems that implemented the domain
approximately. Compliance rules that were almost right.
Business logic that handled the common case but missed
the edge case that the domain expert knew about but
never thought to mention.

This was not a failure of individual engineers. It was
the inevitable consequence of a process where domain
knowledge had to pass through a single narrow channel —
the engineer's interpretation — before becoming software.

---

## The programmer bottleneck

The software industry has always known that the
bottleneck is not typing code — it is understanding what
to build. The response was to split the role: analysts
who understood the domain, and programmers who wrote the
code. This formalized the translation problem, not its
solution. It added a handoff instead of removing one.

The cost of producing programmers is enormous. Years of
formal education, years of practical experience,
continuous learning as platforms and languages evolve.
And the output of this investment is a person who
translates — who takes someone else's understanding and
re-expresses it in a language machines can execute.

The industry tried to close this gap from multiple
directions. No-code and low-code platforms attempted to
let non-programmers build software directly. They
succeeded for narrow cases: simple workflows, forms,
dashboards. They failed for anything complex enough to
require real engineering judgment: error handling,
concurrency, security, integration with other systems.

The fundamental issue persisted: someone had to translate
domain knowledge into something executable. The
programmer remained indispensable — expensive to train,
scarce in supply, and the sole bridge between what the
organization knew and what the software did.

---

## What the 1980s understood and abandoned

The software engineering community recognized this
problem decades ago. Structured analysis, stepwise
refinement, formal specifications — the 1970s and 1980s
produced rigorous methods for capturing domain knowledge
before writing code. The goal was exactly what Code from
Spec describes: a structured artifact that expressed
intent, could be reviewed by domain experts, and guided
implementation.

The methods failed not because they were wrong but
because they were expensive. Maintaining a specification
in sync with evolving code required constant manual
effort. The spec drifted. The team stopped trusting it.
The cost of maintaining the spec exceeded the cost of
fixing the bugs it would have prevented.

The industry didn't abandon specifications because they
were wrong. It abandoned them because the cost of
keeping them current was too high. The constraint was
economic, not fundamental.

---

## Agile: what it solved and what it conceded

Agile, lean, and continuous delivery solved real
problems: fast feedback loops, incremental delivery,
prioritization under uncertainty, responsiveness to
change. These remain valuable. Code from Spec does not
replace them.

Agile's subtlest contribution was removing the
bottleneck that formal methods had created: the
separation between "people who specify" and "people who
code." Agile trusted the programmer to make domain
decisions at the point of implementation — to talk
directly to the user, to understand the business
context, to specify and build in the same motion. This
democratized authorship in a way that the 1980s never
achieved.

Agile compensated for the risk with short cycles. When
the programmer made a wrong domain decision, the
delivery cycle was short enough that the user saw the
result quickly and corrected it. This worked as implicit
domain feedback — not a spec review, but "I saw it
running, that's wrong, fix it."

This worked well when the end user was the domain
expert. It worked less well for domains where the person
who sees the demo is not the person who knows the rules.
A product demo shows screens and flows — it does not
show that the provisioning calculation uses the wrong
cutoff date, or that the settlement logic violates a
regulatory constraint. The compliance officer, the
accountant, the legal analyst — they are not in the
demo. Or if they are, they cannot tell from a demo
whether the underlying logic is correct.

Agile solved the bottleneck by removing the spec. The
knowledge that would have been in the spec became
invisible — encoded in code that only the programmer
could read. When the programmer left, the knowledge
left too.

---

## AI changes the economics

AI inverts the cost structure. Code generation is now
cheap. An agent can implement a well-specified component
in seconds. The scarce resource is no longer writing the
code — it is knowing what to write.

More importantly: when code is generated from spec,
synchronization is automatic by construction. The spec
does not drift from the code because the code is derived
from the spec. There is no separate maintenance burden.
The argument that killed formal specification in the
1980s no longer applies.

This is not a marginal improvement. It is the removal of
the constraint that made formal specifications
impractical for forty years.

Code from Spec restores what agile conceded — without
reintroducing the bottleneck. The spec tree provides the
structured specification that formal methods promised,
kept current by construction because code is derived
from it. But unlike the 1980s, authorship is not
restricted to a special role. Everyone contributes —
just as agile intended — but the contributions are
recorded in the tree, not lost in the code.

The short cycles, incremental delivery, and feedback
loops of agile remain — but now each iteration produces
a spec change and a regeneration, not an ad-hoc code
change that drifts from an outdated document.

The two are complementary. Agile provides the process.
Code from Spec provides the artifact.

---

## The return to engineering

For decades, the engineering profession was reduced to
typing. The engineer's job was to write code — to
translate someone else's decisions into syntax. This was
always a misallocation. Engineering is analysis and
design: understanding the problem, structuring the
solution, placing constraints, resolving ambiguities.

Code from Spec restores the practice. The engineer's job
is structuring the spec tree, placing guard nodes,
resolving ambiguities, reviewing contributions from
domain experts. The agent types. The engineer thinks.

This is not a demotion of engineering. It is a return to
what engineering was always supposed to be — and now can
be, because the constraint that prevented it has been
removed.

The engineer is not hired to write code. The engineer is
hired because they are intelligent people who solve
problems. The code was always a side effect. Now it is
an automated side effect.
