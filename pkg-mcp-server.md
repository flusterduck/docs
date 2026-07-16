# @flusterduck/mcp-server

The local stdio MCP server. It connects an AI assistant to your Flusterduck data so it can read scores, issues, journeys, and revenue leakage on your behalf.

## Install

```bash
npx -y @flusterduck/mcp-server
```

No install step is required. The `npx` command runs the server over stdio. The published binary is `flusterduck-mcp`.

## Usage

Add it to your assistant's MCP configuration. The server authenticates with a scoped MCP key:

```json
{
  "mcpServers": {
    "flusterduck": {
      "command": "npx",
      "args": ["-y", "@flusterduck/mcp-server"],
      "env": {
        "FLUSTERDUCK_API_KEY": "fd_mcp_xxxxxxxxxxxx"
      }
    }
  }
}
```

Once connected, your assistant can query confusion scores, open issues, user journeys, and revenue impact through the server's tools.

## Links

Published on npm as `@flusterduck/mcp-server`. Install pulls the latest published version.
