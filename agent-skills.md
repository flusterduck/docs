# Agent Skills

We publish four Claude Agent Skills that teach an AI assistant how to do real product work with Flusterduck: triage what to fix, verify whether a fix worked, investigate an underperforming page, and set up the SDK and MCP. Each skill is a single markdown file you drop into your agent's skills directory. No package to install, no build step.

A skill is a `SKILL.md` file: YAML frontmatter with a trigger-rich description, followed by the procedure the agent runs once a request matches. Your agent reads only the description until a request lines up with it, then loads the full body. The skills work in Claude Code, Claude Desktop, and any agent that reads a skills directory.

## The four skills

| Skill | What it does | Triggers on |
|---|---|---|
| `flusterduck-triage` | Ranks open UX issues by confidence, affected users, and revenue at risk, then splits them into "Fix this" and "Watching" | "what should I fix", "what's broken", "what's hurting conversion" |
| `flusterduck-verify-fix` | Compares page confusion before and after a deploy and reports HELD, REGRESSED, or PENDING with a next action | "did my fix work", "did the deploy help", "is it resolved" |
| `flusterduck-investigate-page` | Deep single-page diagnosis: score, dominant signals, worst elements, deploy timing, and concrete recommended changes | "why is /checkout underperforming", "investigate /calculator" |
| `flusterduck-setup` | The real install path: the SDK, framework wrappers, publishable key, and connecting an assistant to the Flusterduck MCP | "install Flusterduck", "connect Flusterduck to Claude" |

Each file is served at `https://flusterduck.com/skills/<name>.md`, and all four are listed with sha256 hashes in the machine-readable index at `https://flusterduck.com/.well-known/agent-skills/index.json`.

## Skills vs MCP vs REST

These three layers do different jobs, and they stack.

**Skills are the playbook.** A skill carries no credentials and returns no data. It tells the assistant how to use Flusterduck well: which tools to call, in what order, how to rank issues, when to say "pending" instead of declaring victory. Use skills when you want the assistant's answers to follow our procedures instead of improvising.

**MCP is the data.** The [MCP server](/mcp) gives an assistant authenticated read access to your live scores, issues, deploys, and recommendations. The triage, verify-fix, and investigate-page skills all read through MCP. Use MCP whenever an AI assistant needs your actual friction data, with or without the skills on top.

**REST is for code.** The [REST API](/rest-api) exposes the same data to scripts, CI pipelines, and backends. No assistant involved. Use it when a cron job posts a weekly summary or a deploy script checks confusion before promoting a release.

In practice: install the skills so the assistant knows the procedure, connect MCP so it has the data, and reach for REST when there is no assistant in the loop.

## Install

The fastest path is the skills CLI, pointed at our public mirror repo [github.com/flusterduck/skills](https://github.com/flusterduck/skills):

```bash
npx skills add flusterduck/skills
```

That installs all four. To install just one:

```bash
npx skills add flusterduck/skills --skill flusterduck-triage
```

The mirror is read-only and maintained automatically from the product repo, so it always matches what we serve from flusterduck.com.

No skills CLI? Each skill lives in its own directory named after the skill, containing one `SKILL.md`. For Claude Code, that is `~/.claude/skills/` for personal use or `.claude/skills/` inside a project:

```bash
mkdir -p ~/.claude/skills/flusterduck-triage
curl -o ~/.claude/skills/flusterduck-triage/SKILL.md https://flusterduck.com/skills/flusterduck-triage.md
```

Repeat for the others: `flusterduck-verify-fix`, `flusterduck-investigate-page`, `flusterduck-setup`. Other agents that read a skills directory follow the same shape; check your agent's docs for where that directory lives.

To verify a download, compare its sha256 against the hash published in `/.well-known/agent-skills/index.json`:

```bash
shasum -a 256 ~/.claude/skills/flusterduck-triage/SKILL.md
```

## Use

Nothing to invoke by hand. Ask a question that matches a skill's triggers and the agent picks it up:

> "What's the worst friction on my site right now?"

The triage skill loads, pulls your open issues through MCP, and returns a ranked plan.

Two prerequisites for the data-reading skills:

1. The Flusterduck SDK is installed on your site, so there is data to read.
2. The assistant is connected to the Flusterduck MCP server with an `fd_mcp_` key. See the [MCP integration](/mcp) page, or just ask the assistant to set it up: the `flusterduck-setup` skill walks it through the whole flow, and the other three fall back to it automatically when Flusterduck is not connected yet.

The skills only read by default. Write actions like resolving an issue require a `manage:write` scoped key and run only when you explicitly ask.
