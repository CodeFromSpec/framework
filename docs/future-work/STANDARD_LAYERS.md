# Standard Language Layers

Today, adopting Code from Spec for a Go project
requires writing the entire `golang/` layer —
translation rules, interface conventions, test
patterns. This is project-specific work that is
largely the same across projects.

A standard "golang layer pack" would provide the
intermediate nodes (translation rules, error
conventions, test patterns) as a reusable package.
A non-programmer could write functional specs, drop
in the layer pack, and generate Go code.

---

## What is needed

- Extract generic translation rules from
  project-specific configuration (module path,
  dependencies).
- A configuration mechanism at the layer root for
  project-specific values.
- A scaffolding tool that generates leaf nodes from
  the functional tree structure (each functional node
  gets a corresponding interface, implementation, and
  test node).
- Validation with a second project to confirm the
  rules are truly generic.

This extends to other languages: a `python/` layer,
a `typescript/` layer, each with their own translation
rules and conventions.
