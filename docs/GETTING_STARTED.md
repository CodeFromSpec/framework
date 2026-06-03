# Getting Started

How to set up a project to use Code from Spec.

---

## 1. Copy the methodology file

Download `CODE_FROM_SPEC.md` and place it at your project root:

```
https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/rules/CODE_FROM_SPEC.md
```

This is the file you will reference at the start of every session
with the AI agent (see Working with the agent below).

---

## 2. Create the spec directory

At the project root, create the spec tree structure:

```
code-from-spec/
  _node.md        <- root spec node
```

The root `_node.md` is the starting point. See
[CODE_FROM_SPEC.md](../rules/CODE_FROM_SPEC.md) for the full
specification structure.

---

## 3. Install tooling

Ask the AI agent to set up the tooling for you. Copy and paste
the following prompt:

````
Set up Code from Spec tooling for this project:

1. Add `/tools/` to `.gitignore` (with leading `/` to match only
   the root directory). Create the file if it does not exist.

2. Download `framework-mcp` for my platform from
   https://github.com/CodeFromSpec/tool-framework-mcp/releases/latest
   and extract it into `tools/`.

3. Download the subagent definition from
   https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/subagents/code-from-spec-artifact-generation.md
   and save it to `.claude/agents/code-from-spec-artifact-generation.md`.

4. Download the artifact generation skill from
   https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/skills/artifact-generation/SKILL.md
   and save it to `.claude/skills/artifact-generation/skill.md`.

5. Register the framework-mcp MCP server. Create or update
   `.mcp.json` in the project root with:

   {
     "mcpServers": {
       "framework-mcp": {
         "type": "stdio",
         "command": "tools/framework-mcp"
       }
     }
   }

   On Windows, use `tools/framework-mcp.exe` as the command.

6. Run `validate_specs` via the MCP server to verify everything
   is wired up.
````

---

## 4. Working with the agent

At the start of each session, reference the methodology file so
the agent has full context:

```
@CODE_FROM_SPEC.md
```

This injects the complete methodology into the conversation
immediately — no network fetch, no waiting, no risk of the agent
skipping the read.

If context gets cluttered during a long session, `/clear` and
`@CODE_FROM_SPEC.md` again to reset.

---

## Available MCP tools

The `framework-mcp` server provides five tools:

| Tool | Purpose |
|------|---------|
| `validate_specs` | Validate spec tree format, detect cycles, check artifact staleness. |
| `load_chain` | Load the complete spec chain for a node (context, input, existing artifact). |
| `write_file` | Write a generated file to disk, validated against the node's output. |
| `chain_hash` | Compute the chain hash for a node. |
| `version` | Print the tool version. |
