<p align="center">
  <a href="https://flusterduck.com">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://flusterduck.com/logo.png">
      <img src="https://flusterduck.com/logo-orange.png" alt="Flusterduck" width="120">
    </picture>
  </a>
</p>

# Flusterduck documentation

This is the complete Flusterduck product documentation as plain markdown, mirrored read-only from the product repo for agents and offline reading. Every page in this repo is a file you can grep, pipe into a context window, or read on a plane.

The canonical, rendered version lives at [docs.flusterduck.com](https://docs.flusterduck.com). Issues and pull requests here will not be picked up; the mirror is overwritten on every docs change upstream.

Prefer one file over fifty? [flusterduck.com/llms-full.txt](https://flusterduck.com/llms-full.txt) is the entire documentation concatenated into a single text file, built for stuffing into an LLM context.

## What Flusterduck is

Automatic issue tracking for UX friction. A small script watches for behavioral signals of confusion on your site (rage clicks, dead clicks, backtracking, form abandonment, and more), clusters them into concrete issues with evidence, and verifies against your deploys whether a fix actually worked. No session replay, no DOM recording, no PII.

## For agents and tooling

- **REST API**: every read goes through `https://api.flusterduck.com/v1/query/...` with an `fd_sec_` key, every write through `/v1/manage/...`. Start with [api.md](api.md).
- **MCP server**: `@flusterduck/mcp-server` (binary `flusterduck-mcp`) exposes live scores, issues, deploys, and recommendations to Claude, Cursor, and any MCP client. See [mcp.md](mcp.md).
- **Agent skills**: ready-made Claude Agent Skills for triage, fix verification, page investigation, and setup live at [github.com/flusterduck/skills](https://github.com/flusterduck/skills). Background in [agent-skills.md](agent-skills.md).

Flusterduck is a paid, proprietary product with a 3-day free trial. Plans and pricing: [flusterduck.com/pricing](https://flusterduck.com/pricing).
