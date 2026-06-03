---
name: cfs-init
description: Initialize a new project for Code from Spec. Creates the spec directory, downloads tooling, configures the MCP server, and installs the skill and subagent definitions.
---

# Initialize Code from Spec

Set up a new project to use Code from Spec.

## When invoked

Run this skill when the user asks to initialize Code from
Spec in a project, or when starting a new project that
will use the methodology.

## Algorithm

Each step checks if its target already exists and skips
if so. This makes the skill safe to re-run — it fills
in whatever is missing without overwriting what is
already in place.

1. **Create the spec root.** If `code-from-spec/_node.md`
   does not exist, create it with minimal content:

   ```markdown
   # ROOT
   ```

3. **Download the methodology file.** Download
   `CODE_FROM_SPEC.md` from
   `https://raw.githubusercontent.com/CodeFromSpec/framework/main/rules/CODE_FROM_SPEC.md`
   and save it to the project root.

4. **Download the MCP server.** Detect the platform
   (OS + architecture) and download the appropriate
   `framework-mcp` binary from
   `https://github.com/CodeFromSpec/tool-framework-mcp/releases/latest`
   into `tools/`. On Windows, the binary is
   `framework-mcp.exe`.

5. **Configure .gitignore.** Add `/tools/` to `.gitignore`
   (with leading `/` to match only the root directory).
   Create the file if it does not exist. Do not duplicate
   if the entry already exists.

6. **Configure the MCP server.** Create or update
   `.mcp.json` in the project root:

   ```json
   {
     "mcpServers": {
       "framework-mcp": {
         "type": "stdio",
         "command": "tools/framework-mcp"
       }
     }
   }
   ```

   On Windows, use `tools/framework-mcp.exe` as the
   command. If `.mcp.json` already exists and has other
   servers, merge — do not overwrite.

7. **Install the subagent definition.** Download from
   `https://raw.githubusercontent.com/CodeFromSpec/framework/main/subagents/cfs-artifact-generation.md`
   and save to
   `.claude/agents/cfs-artifact-generation.md`.
   Create the directory if needed.

8. **Install skills.** Download the following skills and
   save them to `.claude/skills/<name>/skill.md`:

   - `cfs-generate` from
     `https://raw.githubusercontent.com/CodeFromSpec/framework/main/skills/cfs-generate/SKILL.md`
   - `cfs-verify` from
     `https://raw.githubusercontent.com/CodeFromSpec/framework/main/skills/cfs-verify/SKILL.md`
   - `cfs-check-meta-language` from
     `https://raw.githubusercontent.com/CodeFromSpec/framework/main/skills/cfs-check-meta-language/SKILL.md`

   Create directories as needed.

9. **Verify.** Connect to the MCP server and call
   `validate_specs`. Expect a report with the root node
   and no errors. Report success to the user.

## Rules

- Do not overwrite existing files without asking.
- If any download fails, report the error and continue
  with the remaining steps.
- Report each step as it completes so the user can see
  progress.
