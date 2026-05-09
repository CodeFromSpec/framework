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

2. Download `staleness-check` for my platform from
   https://github.com/CodeFromSpec/tool-staleness-check/releases/latest
   and extract it into `tools/`.

3. Download `subagent-mcp` for my platform from
   https://github.com/CodeFromSpec/tool-subagent-mcp/releases/latest
   and extract it into `tools/`.

4. Download the subagent definition from
   https://raw.githubusercontent.com/CodeFromSpec/framework/refs/heads/main/subagents/code-from-spec-code-generation.md
   and save it to `.claude/agents/code-from-spec-code-generation.md`.

5. Register the subagent-mcp MCP server. Create or update
   `.mcp.json` in the project root with:

   {
     "mcpServers": {
       "subagent-mcp": {
         "type": "stdio",
         "command": "tools/subagent-mcp"
       }
     }
   }

   On Windows, use `tools/subagent-mcp.exe` as the command.

6. Run `staleness-check` to verify everything is wired up.
````

---

## Working with the agent

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
