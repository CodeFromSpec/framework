# Artifact Pinning

When an artifact is correct and tested, regenerating it
because an ancestor changed a comment or added an
unrelated constraint is wasteful. A pinning mechanism
would let the user mark an artifact as verified:

```yaml
pinned: true
```

A pinned artifact is excluded from regeneration even
when its chain hash changes. The staleness report would
still flag it — but as "stale (pinned)" rather than
"stale", so the user knows the hash diverged. Unpinning
and regenerating would be explicit.

This trades automatic consistency for reduced churn.
It is most useful for stable leaf nodes deep in the
tree where ancestor changes rarely affect the actual
generated code.
