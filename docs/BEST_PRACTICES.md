# Best Practices

Lessons learned from using Code from Spec in practice. These are
not rules — the methodology works without them. They are patterns
that reduce friction and avoid common pitfalls.

---

## Batch spec changes before resolving staleness

### The problem

Every spec change triggers a staleness cascade. If you edit a
leaf node, its dependents become stale, their dependents become
stale, and so on. Resolving staleness after each individual edit
means walking through the cascade multiple times — most of it
mechanical version bumps with no content change.

For example, suppose you need to change two leaf nodes that
share a common dependent — say a library node and a handler
node that both feed into a server node. If you edit the library
first and resolve staleness immediately, the cascade propagates
through the server and its test nodes. Then you edit the
handler — and the server and its test nodes cascade again. Two
rounds of mechanical bumps for work that could have been one.

### The practice

Complete all related spec edits before running `staleness-check`.
Design the changes, write them into the spec nodes, and only then
start the resolution process.

The framework's resolution rules do not require immediate
resolution after each edit. They require that staleness is
resolved in order (specs before tests, tests before code) and
top-down within each layer. Nothing prevents you from
accumulating changes first.

### When to resolve

A good trigger is: "I'm done changing specs and ready to
generate code." At that point, run `staleness-check` once,
resolve the cascade once, and proceed to code generation.

If you're unsure whether you're done — keep editing. A premature
resolution round is wasted effort if you end up changing another
spec five minutes later.

### The exception

If a spec change introduces ambiguity or you need to validate
your direction before continuing, it's fine to resolve staleness
mid-stream. The practice is about avoiding *unnecessary*
intermediate rounds, not about never resolving until the very
end.

---

## Diagnose before regenerating

### The problem

When generated code fails tests, the instinct is to regenerate
immediately — fix the spec, dispatch the subagent, hope it works
this time. This often produces the same bug or a different one,
because the root cause was never understood.

The spec might be correct and the subagent might have made a
reasonable but wrong implementation choice. Regenerating from the
same spec can produce the same wrong choice, or a different wrong
choice, burning tokens without progress.

### The practice

When tests fail after code generation, stop and diagnose:

1. **Read the failing test output.** What specifically failed?
   An assertion, a panic, a compilation error?

2. **Read the generated code.** Find the line or logic that
   caused the failure. Understand what the code is doing and
   why it's wrong.

3. **Trace back to the spec.** Is the spec ambiguous? Missing a
   constraint? Prescribing something that doesn't work? Or is
   the spec correct and the subagent simply implemented it
   incorrectly?

4. **Fix the spec if needed.** If the spec is the problem,
   correct it. Be specific — add the constraint, prescribe the
   approach, clarify the ambiguity. A vague spec fix produces
   vague code.

5. **Regenerate.** Now that you understand the problem and the
   spec addresses it, regeneration is targeted rather than
   hopeful.

### A real example

In one session, generated code used a standard library function
(`filepath.Match`) that behaved differently across operating
systems. Tests passed on Windows but failed on Linux. The spec
was not wrong — it simply didn't prescribe which function to
use, so the subagent chose one that seemed reasonable.

The first instinct was to regenerate. Instead, the team
investigated: they read the test output, traced the failure to
the function's platform-dependent behavior, identified that the
spec needed to prescribe a platform-independent alternative
(`path.Match`), updated the spec, and regenerated. The fix was
permanent.

Had they regenerated without diagnosing, the subagent might have
chosen the same function again — or a different one with its own
problems.

### The principle

Regeneration is not debugging. The subagent generates code from
the spec it receives. If the spec doesn't address the problem,
no amount of regeneration will fix it. Diagnosis is the step
that turns a failing test into a better spec.
