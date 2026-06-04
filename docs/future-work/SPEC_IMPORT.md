# Spec Import from External Repositories

Guard nodes, conventions, and platform standards could
live in a shared repository and be imported into
project-specific spec trees. This would allow
organization-wide standards to propagate automatically.

Example: a `platform-standards` repository with guard
nodes for security, compliance, and coding conventions.
Each project imports them and depends on them. When the
standard changes, all projects detect staleness.

This connects to standard language layers — a Go layer
pack could be distributed as an importable repository.
