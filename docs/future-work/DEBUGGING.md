# Debugging in Code from Spec

The gap between a failing test and the root cause in a
spec is the least tooled part of the methodology today.

---

## The problem

When a test fails, the error message is a **symptom**:

```
expected ErrCelcoinAPI, got AccountNotFoundError
```

The cause could be in any of:

- The implementation spec (wrong logic)
- The test spec (wrong setup or assertion)
- The error spec (wrong error code mapping)
- A dependency spec (missing or incorrect interface)
- The frontmatter (missing `depends_on`, so the
  subagent never saw a required interface)

Tracing from symptom to cause requires **6 manual
jumps**:

1. Read the test output
2. Find the failing line in the generated file
3. Look up the file in the manifest to find the node
4. Read the node's chain (ancestors + dependencies)
5. Identify which spec is ambiguous or wrong

Today, steps 3–6 are entirely manual. The human greps,
opens files, and mentally reconstructs the chain. The
AI assistant can help when the problem is mechanical
("the import is wrong") but struggles when the problem
is semantic ("the spec said A, the subagent interpreted
B, and B is wrong in a way that produces error C in
test D").

---

## What exists today

The framework already has the information needed:

- **File → node**: the manifest maps each output path
  to the node that generated it.
- **Node → chain**: `load_chain` assembles the full
  context for any node.
- **Node → dependencies**: `depends_on` in the
  frontmatter lists every spec the node imports.

The missing piece is a tool that connects **test
failure → file → node → chain → related specs** in
one step.

---

## Proposed: diagnose tool

Given a file path, resolve the full diagnostic context:

```
$ cfs diagnose internal/accountclose/accountClose_test.go

Artifact: internal/accountclose/accountClose_test.go
Node:     SPEC/.../account-close/test
Hash:     w7NAAzpoUS-XI2VQAS6rr9WyLK0

Chain (6 ancestors):
  architecture → backend → internal →
  api → operations → account-close → test

Dependencies (5):
  - testutils
  - celcoin-gateway(Interface)
  - errors(Errors)
  - celcoin-api/api/account-close
  - database

Related implementation:
  SPEC/.../account-close/implementation
  → internal/accountclose/accountClose.go
```

This is **language-agnostic** — it operates on file
paths and spec metadata, not on test output formats.
The language-specific part (parsing `go test` output
to extract the failing file:line) stays outside the
framework, handled by the human or the AI assistant
in conversation.

---

## Why this matters

In traditional development, debugging is: read the
code, understand the bug, fix the code. The developer
has full context because they wrote the code.

In Code from Spec, the developer did not write the
code. The code is a generated artifact. Debugging
means understanding what the **spec** said that led
to the wrong code — a fundamentally different skill
that requires navigating the spec tree, not the
source tree.

The Spec IDE (see SPEC_IDE.md) helps with navigation.
The diagnose tool helps with the specific case of
"a test failed, where do I look?" Together, they
close the gap between the methodology's promise
(all knowledge is in the spec tree) and the
developer's experience (I'm staring at a Go stack
trace).
