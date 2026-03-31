# Claude Code Explorer — MCP Server

A standalone [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server that lets any MCP-compatible client explore the Claude Code source code.

## What It Does

Exposes 7 tools, 3 resources, and 5 prompts for navigating the ~1,900-file, 512K+ line Claude Code codebase:

### Tools

| Tool | Description |
|------|-------------|
| `list_tools` | List all 40+ agent tools (BashTool, FileEditTool, etc.) |
| `list_commands` | List all 50+ slash commands (/commit, /review, etc.) |
| `get_tool_source` | Read a specific tool's implementation |
| `get_command_source` | Read a specific command's implementation |
| `read_source_file` | Read any file from `src/` by relative path |
| `search_source` | Regex search across the entire source tree |
| `list_directory` | List contents of any directory under `src/` |
| `get_architecture` | Get a full architecture overview |

### Resources

| URI | Description |
|-----|-------------|
| `claude-code://architecture` | README / architecture overview |
| `claude-code://tools` | Tool registry (JSON) |
| `claude-code://commands` | Command registry (JSON) |
| `claude-code://source/{path}` | Any source file (template) |

### Prompts

| Prompt | Description |
|--------|-------------|
| `explain_tool` | Deep-dive explanation of a specific tool's purpose, schema, permissions, and flow |
| `explain_command` | Explanation of a specific slash command's behavior and implementation |
| `architecture_overview` | Guided tour of the full Claude Code architecture |
| `how_does_it_work` | Explain a feature/subsystem (permissions, MCP, bridge, etc.) |
| `compare_tools` | Side-by-side comparison of two tools |

## Setup

```bash
cd mcp-server
npm install
npm run build
```

## Configuration

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "claude-code-explorer": {
      "command": "node",
      "args": ["/absolute/path/to/claude-code/mcp-server/dist/index.js"],
      "env": {
        "CLAUDE_CODE_SRC_ROOT": "/absolute/path/to/claude-code/src"
      }
    }
  }
}
```

### VS Code (GitHub Copilot)

Add to `.vscode/mcp.json` in your workspace:

```json
{
  "servers": {
    "claude-code-explorer": {
      "type": "stdio",
      "command": "node",
      "args": ["${workspaceFolder}/mcp-server/dist/index.js"],
      "env": {
        "CLAUDE_CODE_SRC_ROOT": "${workspaceFolder}/src"
      }
    }
  }
}
```

### Cursor

Add to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "claude-code-explorer": {
      "command": "node",
      "args": ["/absolute/path/to/claude-code/mcp-server/dist/index.js"],
      "env": {
        "CLAUDE_CODE_SRC_ROOT": "/absolute/path/to/claude-code/src"
      }
    }
  }
}
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_CODE_SRC_ROOT` | `../src` (relative to dist/) | Path to the Claude Code `src/` directory |

## Prompts

The server also exposes prompt templates for guided exploration:

| Prompt | Description |
|--------|-------------|
| `explain_tool` | Deep-dive explanation of a specific tool (input schema, permissions, execution flow) |
| `explain_command` | Explain how a slash command works |
| `architecture_overview` | Guided tour of the entire Claude Code architecture |
| `how_does_it_work` | Explain a feature or subsystem (e.g. "permission system", "MCP client", "query engine") |
| `compare_tools` | Side-by-side comparison of two tools |

## Example Usage

Once connected, you can ask your AI assistant things like:

- "List all Claude Code tools"
- "Show me the BashTool implementation"
- "Search for how permissions are checked"
- "What files are in the bridge directory?"
- "Read the QueryEngine.ts file, lines 1-100"
- "How does the MCP client connection work?"
- Use the `explain_tool` prompt with "FileEditTool" to get a full breakdown
- Use `how_does_it_work` with "bridge" to understand IDE integration

## Development

```bash
npm install
npm run dev    # Run directly with tsx (no build needed)
npm run build  # Compile TypeScript to dist/
npm start      # Run compiled server
```


