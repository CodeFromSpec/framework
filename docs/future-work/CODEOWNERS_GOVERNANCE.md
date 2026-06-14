# CODEOWNERS Governance

Status: idea, not designed. Captured from a brainstorm on
2026-06-12. To be investigated in the future.

---

## The insight

The spec tree, when organized by layer, aligns each subtree
with one team (see the layer-ownership model). CODEOWNERS maps
path patterns to required reviewers. Because path equals
owner in such a tree, CODEOWNERS becomes a direct,
non-contrived governance mechanism:

```
/code-from-spec/functional/     @analysts
/code-from-spec/golang/         @engineers
/code-from-spec/integrations/   @platform-team
```

A PR touching a layer auto-requests that layer's owner, and
with branch protection ("require review from code owners")
cannot merge without their approval.

## Why it fits Code from Spec specifically

- **Complements guard nodes, does not duplicate them.** A
  guard node protects the *generation* (the agent cannot
  produce code that violates an inherited constraint).
  CODEOWNERS protects the *constraint itself* (a human cannot
  weaken the guard node without its owner approving). Guard
  node = structural protection of the output; CODEOWNERS =
  procedural protection of the rule. Without the latter,
  someone edits a guard node, every descendant regenerates
  silently compromised — structurally valid, semantically
  broken.

- **Operationalizes "the PR becomes an organizational gate."**
  The finer-grained, continuous counterpart to the site's
  main→prod governance gate: routes the right reviewer to
  every spec change, not just at deploy time.

## Open questions

- **File-level granularity.** CODEOWNERS protects whole files.
  A `_node.md` mixing authors (e.g. business `# Public` and
  engineering `# Agent`) cannot be split. This argues that
  authorship boundaries must fall on layer/directory
  boundaries, never within a node — which the layer-ownership
  model already implies.

- **Who owns generated code?** No one should review derived
  artifacts. Output paths either stay out of CODEOWNERS or
  carry a rule marking them as derived/not-reviewed. If anyone
  needs to own generated code, the authorship boundary is in
  the wrong place.

- **Forge dependency.** CODEOWNERS is a GitHub/GitLab feature
  and assumes the org structure (teams, assignments) already
  exists. It is a practice Code from Spec enables, not part of
  the framework.
