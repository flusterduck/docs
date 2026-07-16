# @flusterduck/mcp-server

The local stdio MCP server. It connects an AI assistant to your Flusterduck data so it can read scores, issues, journeys, and revenue leakage on your behalf, and (with a write-scoped key) triage issues and manage alerts.

## Install

```bash
npx -y @flusterduck/mcp-server
```

No install step is required. The `npx` command runs the server over stdio. The published binary is `flusterduck-mcp`.

## Usage

Add it to your assistant's MCP configuration. The server needs one value: an MCP key from the dashboard (Settings > API Keys). Keys are scoped to your current site when created, so the server finds the right site by itself.

```json
{
  "mcpServers": {
    "flusterduck": {
      "command": "npx",
      "args": ["-y", "@flusterduck/mcp-server"],
      "env": {
        "FLUSTERDUCK_MCP_KEY": "fd_mcp_xxxxxxxxxxxx"
      }
    }
  }
}
```

| Variable | Required | Description |
|---|---|---|
| `FLUSTERDUCK_MCP_KEY` | yes | An `fd_mcp_` key from Settings > API Keys. `FLUSTERDUCK_API_KEY` is accepted as an alias and also takes `fd_sec_` keys. |
| `FLUSTERDUCK_SITE_ID` | no | A site UUID. Only needed for org-scoped keys, to pick which site to read; site-scoped keys already know their site. |
| `FLUSTERDUCK_ORG_ID` | no | Pins org-level reads to a specific org. Normally unnecessary: the key's own org is used automatically. |
| `FLUSTERDUCK_API_URL` | no | Override the API base URL. Defaults to the hosted Flusterduck API. |

Once connected, your assistant can query confusion scores, open issues, user journeys, and revenue impact through the server's tools. Write tools (issue triage, alert management, annotations) additionally need a key with `manage:write` scope.

Prefer zero configuration? The hosted server at `https://mcp.flusterduck.com/mcp` needs no keys at all: you sign in with your Flusterduck account on first use. See the [MCP integration guide](./mcp) for both paths and the full tool list.

## Links

Published on npm as `@flusterduck/mcp-server`. Install pulls the latest published version.
