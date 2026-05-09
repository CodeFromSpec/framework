# Future Work

Ideas and plans that are not yet part of the framework.

---

## Spec visualizer

A dedicated application for browsing and editing specs visually.
Instead of navigating the filesystem, users would see a node
navigator that displays the spec tree with proper hierarchy,
search, and inline editing.

### Node ordering

The filesystem sorts directories alphabetically, which does not
always reflect the intended reading order — especially for layers
(e.g., `database/` appears before `domain/`).

The visualizer would support an `order` field in the frontmatter
to control display order among sibling nodes:

```yaml
---
order: 10
depends_on:
  - ROOT/external/payments-api
artifacts:
  - id: main
    path: internal/transfers/transfers.go
---
```

Lower values appear first. Nodes without `order` are sorted
alphabetically after ordered nodes. The framework ignores
unknown frontmatter fields, so `order` can be added to specs
today without breaking anything — the visualizer will use it
when it exists.
