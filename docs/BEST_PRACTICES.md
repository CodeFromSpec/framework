# Best Practices

Lessons learned from using Code from Spec in practice. These are
not rules — the methodology works without them. They are patterns
that reduce friction and avoid common pitfalls.

---

## Diagnose before regenerating

### The problem

When generated artifacts fail tests, the instinct is to regenerate
immediately — fix the spec, dispatch the subagent, hope it works
this time. This often produces the same bug or a different one,
because the root cause was never understood.

The spec might be correct and the subagent might have made a
reasonable but wrong implementation choice. Regenerating from the
same spec can produce the same wrong choice, or a different wrong
choice, burning tokens without progress.

### The practice

When tests fail after artifact generation, stop and diagnose:

1. **Read the failing test output.** What specifically failed?
   An assertion, a panic, a compilation error?

2. **Read the generated artifact.** Find the line or logic that
   caused the failure. Understand what it is doing and why it's
   wrong.

3. **Trace back to the spec.** Is the spec ambiguous? Missing a
   constraint? Prescribing something that doesn't work? Or is
   the spec correct and the subagent simply implemented it
   incorrectly?

4. **Fix the spec if needed.** If the spec is the problem,
   correct it. Be specific — add the constraint, prescribe the
   approach, clarify the ambiguity. A vague spec fix produces
   vague output.

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

Regeneration is not debugging. The subagent generates artifacts
from the spec it receives. If the spec doesn't address the
problem, no amount of regeneration will fix it. Diagnosis is the
step that turns a failing test into a better spec.

---

## Start every session with the methodology

### The problem

Expecting the AI agent to fetch the methodology from a remote URL
at the start of each session is unreliable. The agent may skip
the read, summarize instead of reading in full, or fail silently
due to network issues. The result is an agent that generates
artifacts without understanding the framework's rules.

### The practice

Keep `CODE_FROM_SPEC.md` at your project root. At the start of
every session, reference it directly:

```
@CODE_FROM_SPEC.md
```

If context gets cluttered during a long session, clear and
re-inject:

```
/clear
@CODE_FROM_SPEC.md
```

This guarantees the agent has the complete methodology in context
before any work begins — no network dependency, no partial reads.
