# Rename Tool

Renaming a logical name today requires manually updating
every reference across the spec tree: `depends_on`
entries, `ARTIFACT/` references, `input` fields, examples
in body text, and the node's own directory path.

A `rename_logical_name` MCP tool that takes an old and
new logical name, scans the spec tree, and updates all
references would eliminate the most tedious and
error-prone part of reorganizations.
