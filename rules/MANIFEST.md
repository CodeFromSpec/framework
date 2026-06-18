# Manifest

The manifest tracks the state of every generated artifact.

---

## Staleness

An artifact is stale when its chain has changed since it was last
generated.

### Chain hash

Staleness is determined by comparing hashes. The **chain hash**
is computed from all positions in the chain. See
CHAIN_HASH.md for the full algorithm.

### Staleness check

The `validate_specs` tool (part of `framework-mcp`) computes
the current chain hash for each node that declares `output`
and compares it with the hash recorded in the manifest.
If they differ, the artifact is stale and must be regenerated.

Artifacts whose files do not exist are reported as `missing`
(a special case of staleness).
