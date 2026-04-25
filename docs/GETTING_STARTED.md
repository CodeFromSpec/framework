# Getting Started

How to set up a project to use Code from Spec.

---

## Quick start

The fastest way to get started is to add an `AGENTS.md` file to
your project root. The agent will read the framework documentation,
create the spec directory, download the tooling, and configure the
MCP server on its own.

Copy the template below into `AGENTS.md` at the project root.
Adjust the framework URLs if you are pinning to a specific version
branch.

````markdown
# Agent Instructions

## Methodology

This project follows the **Code from Spec** methodology. All code
is generated from specifications — the spec tree is the
authoritative source of truth, not the code itself.

Before working on this project, read the framework documentation:

- Overview: `https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/README.md`
- Specification structure and versioning: `https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/rules/CODE_FROM_SPEC.md`
- Code generation rules: `https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/rules/CODE_GENERATION.md`

## Key Rules

- The spec tree under `code-from-spec/` drives all
  implementation decisions.
- Only leaf nodes generate code. Each source file contains a
  spec comment referencing its spec node and version.
- Never change code; correct the corresponding spec node and
  regenerate.
- When a spec version changes, the code is stale and must be
  regenerated to match.

## Tooling Bootstrap

**You MUST ensure tooling is properly installed if you have not
already done so:**

1. Verify that `/tools/` is present in `.gitignore`. If it is not,
   add it. Use the leading `/` to match only the root `tools/`
   directory.

2. Verify that `tools/staleness-check` (or `tools/staleness-check.exe`
   on Windows) exists. If it does not, download the latest release
   for your platform:

   | Platform | Download URL |
   |---|---|
   | Windows amd64 | `https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_windows_amd64.zip` |
   | Windows arm64 | `https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_windows_arm64.zip` |
   | Linux amd64 | `https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_linux_amd64.tar.gz` |
   | Linux arm64 | `https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_linux_arm64.tar.gz` |
   | macOS arm64 | `https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_darwin_arm64.tar.gz` |
   | macOS amd64 | `https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_darwin_amd64.tar.gz` |

   Extract the binary into `tools/`.

3. Verify that `tools/subagent-mcp` (or `tools/subagent-mcp.exe`
   on Windows) exists. If it does not, download the latest release
   for your platform:

   | Platform | Download URL |
   |---|---|
   | Windows amd64 | `https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_windows_amd64.zip` |
   | Windows arm64 | `https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_windows_arm64.zip` |
   | Linux amd64 | `https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_linux_amd64.tar.gz` |
   | Linux arm64 | `https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_linux_arm64.tar.gz` |
   | macOS arm64 | `https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_darwin_arm64.tar.gz` |
   | macOS amd64 | `https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_darwin_amd64.tar.gz` |

   Extract the binary into `tools/`.

4. Verify that `.claude/agents/code-from-spec-code-generation.md`
   exists. If it does not, download it from the framework repository:

   ```
   https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/subagents/code-from-spec-code-generation.md
   ```

   Save the contents to
   `.claude/agents/code-from-spec-code-generation.md`.

5. Verify that the `subagent-mcp` MCP server is configured.
   Check both `.mcp.json` and `.claude/settings.json`. If
   neither contains the configuration, create or update
   `.mcp.json` in the project root with:

   ```json
   {
     "mcpServers": {
       "subagent-mcp": {
         "type": "stdio",
         "command": "tools/subagent-mcp"
       }
     }
   }
   ```

   On Windows, use `tools/subagent-mcp.exe` as the command.

   If you created or modified the MCP configuration, inform
   the user that a restart of Claude Code is required for the
   new MCP server to become available.

## Workflow Rules

- **Do not** run staleness checks, resolve staleness, or
  generate code unless the user explicitly requests it.
- Reading the methodology documents (the three URLs above)
  remains a prerequisite before any of these actions.
````

---

## Manual setup

If you prefer to set things up yourself, or want to understand
what the AGENTS.md template automates, follow these steps.

### 1. Create the spec directory

At the project root, create the spec tree structure:

```
code-from-spec/
  _node.md        <- root spec node
```

The root `_node.md` is the starting point. See
[CODE_FROM_SPEC.md](../rules/CODE_FROM_SPEC.md) for the full
specification structure.

### 2. Install tooling

Create a `tools/` directory at the project root and add `/tools/`
to `.gitignore` (with the leading `/` to match only the root
directory):

```
/tools/
```

#### staleness-check

Download the latest release for your platform and extract it into
`tools/`:

| Platform | Download |
|---|---|
| Windows amd64 | [staleness-check_windows_amd64.zip](https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_windows_amd64.zip) |
| Windows arm64 | [staleness-check_windows_arm64.zip](https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_windows_arm64.zip) |
| Linux amd64 | [staleness-check_linux_amd64.tar.gz](https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_linux_amd64.tar.gz) |
| Linux arm64 | [staleness-check_linux_arm64.tar.gz](https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_linux_arm64.tar.gz) |
| macOS arm64 | [staleness-check_darwin_arm64.tar.gz](https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_darwin_arm64.tar.gz) |
| macOS amd64 | [staleness-check_darwin_amd64.tar.gz](https://github.com/CodeFromSpec/tool-staleness-check/releases/latest/download/staleness-check_darwin_amd64.tar.gz) |

#### subagent-mcp

Download the latest release for your platform and extract it into
`tools/`:

| Platform | Download |
|---|---|
| Windows amd64 | [subagent-mcp_windows_amd64.zip](https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_windows_amd64.zip) |
| Windows arm64 | [subagent-mcp_windows_arm64.zip](https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_windows_arm64.zip) |
| Linux amd64 | [subagent-mcp_linux_amd64.tar.gz](https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_linux_amd64.tar.gz) |
| Linux arm64 | [subagent-mcp_linux_arm64.tar.gz](https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_linux_arm64.tar.gz) |
| macOS arm64 | [subagent-mcp_darwin_arm64.tar.gz](https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_darwin_arm64.tar.gz) |
| macOS amd64 | [subagent-mcp_darwin_amd64.tar.gz](https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest/download/subagent-mcp_darwin_amd64.tar.gz) |

### 3. Install the code generation agent

Download the subagent definition from the framework repository:

```
https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/subagents/code-from-spec-code-generation.md
```

Save the contents to
`.claude/agents/code-from-spec-code-generation.md`.

### 4. Configure the MCP server

Register the subagent-mcp server in `.mcp.json` at the project
root (or in `.claude/settings.json`):

```json
{
  "mcpServers": {
    "subagent-mcp": {
      "type": "stdio",
      "command": "tools/subagent-mcp"
    }
  }
}
```

On Windows, use `tools/subagent-mcp.exe` as the command.

### 5. Verify

Run the staleness check to confirm everything is wired up:

```bash
./tools/staleness-check        # Linux / macOS
./tools/staleness-check.exe    # Windows
```

If the tool runs and reports `spec_staleness: []`,
`test_staleness: []`, and `code_staleness: []`, the project is
clean. Otherwise, follow the
[staleness resolution process](../rules/CODE_FROM_SPEC.md#staleness-resolution).
