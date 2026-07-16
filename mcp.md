# MCP Integration

Set this up once and you stop opening the dashboard to check on friction data. Ask your AI assistant:

> "What's broken on checkout right now?"

It calls `get_site_context`, pulls your live scores and open issues, and tells you. Post-deploy checks, issue triage, weekly summaries. All without touching a UI.

## Fastest install

The hosted server needs no keys and no install. Add the URL and sign in with your Flusterduck account when your client prompts you:

```bash
claude mcp add --transport http flusterduck https://mcp.flusterduck.com/mcp
```

Prefer running the server locally (stdio)? `npx` fetches it on first run. Create an MCP key in the dashboard (Settings > API Keys). The key is scoped to your current site when you create it, so the key alone is enough:

```bash
claude mcp add flusterduck \
  --env FLUSTERDUCK_MCP_KEY=fd_mcp_xxxxxxxxxxxx \
  -- npx -y @flusterduck/mcp-server
```

Only org-scoped keys also need `--env FLUSTERDUCK_SITE_ID=<site-uuid>` to pick which site to read.

## Setup

All local (stdio) configs below need exactly one value: an MCP key from Settings > API Keys. Keys are scoped to your current site when you create them, so the server finds the right site by itself. Add `"FLUSTERDUCK_SITE_ID": "<site-uuid>"` to the env block only when using an org-scoped key.

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (npx pulls the server automatically, nothing to install first):

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

Restart Claude Desktop. Tools are available immediately.

### Claude Code CLI

```bash
claude mcp add flusterduck \
  --env FLUSTERDUCK_MCP_KEY=fd_mcp_xxxxxxxxxxxx \
  -- npx -y @flusterduck/mcp-server
```

Or add directly to `~/.claude/settings.json`:

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

### Cursor

`~/.cursor/mcp.json` (global) or `.cursor/mcp.json` in the project root:

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

Reload the Cursor window after saving.

### Windsurf

`~/.codeium/windsurf/mcp_config.json`:

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

### VS Code (GitHub Copilot)

`.vscode/mcp.json` in the workspace:

```json
{
  "servers": {
    "flusterduck": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@flusterduck/mcp-server"],
      "env": {
        "FLUSTERDUCK_MCP_KEY": "fd_mcp_xxxxxxxxxxxx"
      }
    }
  }
}
```

### Remote server

Point your client at `https://mcp.flusterduck.com/mcp`. There is nothing to copy: the first connection opens a consent page, you sign in with your Flusterduck account, and access is granted through OAuth 2.1 with PKCE. Disconnecting from your MCP client revokes access immediately.

## Keys

The hosted server never needs a key; sign-in handles it. MCP keys (`fd_mcp_`, created under Settings > API Keys) are only for the local stdio server.

## MCP or CLI?

Both surfaces expose the same data; pick by where the agent runs. MCP is right for interactive assistants (Claude Desktop, Cursor, Claude Code sessions): persistent connection, tools, and the multi-step prompts below. The [CLI](./cli) is right for CI jobs and shell scripts: `npx flusterduck-cli issues --site <id> --json` for reads, `issue resolve|ignore|reopen|start` to manage issues, and `deploy notify` to record deploys. Every command takes `--json`, so agents can parse the output directly.

**Read-only** (`mcp:read`): all query tools and prompts, including `whoami` (introspect the key's scope and permissions). Can't write anything. Start here.

**Read + write** (`mcp:read` + `manage:write`): adds `update_issue`, `rate_issue`, `update_alert`, `add_annotation`, `create_alert_rule`, `update_alert_rule`, `delete_alert_rule`, and `send_feedback`. Needed for triage and annotation workflows.

## Prompts

The prompts are the highest-value part. They're multi-step workflows built into the server. The AI calls several tools in sequence and synthesizes the result into a useful answer. Use prompts for the common cases; use raw tools when you need something specific.

### post_deploy_check

Finds your most recent deploy, compares confusion scores before and after, identifies pages where friction spiked, and returns PASS / WARN / FAIL. Writes an annotation automatically on FAIL.

Run this after every deploy. Takes about 30 seconds.

### triage_open_issues

Scans open issues by signal count, moves each to the right status (`triaged`, `in_progress`, or `ignored`), adds a triage note to each, and writes a summary annotation. Requires `manage:write`.

Most useful first thing Monday morning before standup.

### diagnose_page

Five-step diagnosis for a specific page. Returns: current friction score, top signal sources by volume, worst-performing element, likely root cause, and one concrete fix recommendation.

```
page_path: "/settings/billing"
```

### investigate_session

Full investigation of a single session: event timeline in order, dominant signal types, matching open issues, and a verdict on whether the session represents a recurring pattern or an outlier. Session ids are 32-character hex strings; get them from `explore`, `get_issue` evidence, or `query_raw_rows` on the `sessions` table.

```
session_id: "f3a9c1d24e8b76051a2b3c4d5e6f7081"
```

### weekly_summary

A weekly UX friction digest formatted for a PM or eng lead: score trends, newly opened issues, open alerts, revenue at risk, top three recommendations. Under 300 words.

Good to run before Monday standup.

## What a real investigation looks like

You ask: "Did the deploy yesterday break anything on checkout?"

The AI runs this automatically:

1. `get_deploys`: finds yesterday's deploy and its timestamp
2. `get_page` with `/checkout`: sees the confusion score jumped from 14 to 38 in the hour after
3. `get_issues` filtered to `open`: two issues opened within 90 minutes of the deploy, a rage click spike on `#place-order` and elevated form abandonment on the payment step
4. `get_elements` scoped to `/checkout`: `#place-order` has 52 rage clicks in 24 hours, baseline is 5/day
5. `diagnose_page`: returns root cause. The button appears inactive during payment processing with no visual indication, causing repeated clicking.
6. `add_annotation`: "Deploy 2026-06-09: /checkout confusion +171%. #place-order rage clicks 10x baseline. Root cause: missing loading state on payment submit."

The whole sequence runs in about 30 seconds and leaves a permanent timeline marker.

## Read tools

All require `mcp:read` scope. Start with `get_site_context`. It gives you the full picture in one call and tells you where to dig next.

**Documentation**: the full Flusterduck docs are bundled into the server, so your assistant can read them before acting on data.

- `list_docs`: Lists every documentation page (slug, title, group). Start here to see what's available.
- `search_docs`: Full-text search across all docs. Pass `query`; returns matching pages with excerpts.
- `get_doc`: Returns a full documentation page by `slug` (e.g. `alerts`, `scoring`, `webhooks`, `react`, `mcp`). Read a feature's page before acting. For example, read `alerts` before creating a rule to understand threshold semantics.

**Start here**

- `get_site_context`: Full snapshot of scores, open issues, active alerts, recent deploys, and top recommendations. The right first call for any investigation.
- `get_scores`: Every tracked page ranked by current confusion score.
- `get_recommendations`: Prioritized fix list ranked by estimated confusion reduction.

**Issues and alerts**

- `get_issues`: All UX issues, newest activity first, 50 per page. Filter by status: `open`, `triaged`, `in_progress`, `verified`, `resolved`, `ignored`, `regressed`.
- `get_issue`: Full detail on one issue, including evidence, session links, verification history, and deploy correlation.
- `get_alerts`: Alerts, newest first, 50 per page. Filter by status: `fired` (nobody has looked yet), `acknowledged`, `investigating`, `resolved`.
- `list_alert_rules`: All configured rules with thresholds, channels, and enabled state. Call this before creating or modifying rules.

**Page deep-dives**

- `get_page`: Score history, active issues, element friction, annotations, and confusion budget for one page. Pass `page: "/checkout"`.
- `get_elements`: Element-level breakdown showing which buttons, forms, and links generate the most signals. Scope by `page` or pull site-wide.
- `get_trends`: Confusion score over time. Pass `days` (1-90) and optionally `page`.
- `compare_pages`: Side-by-side confusion score comparison. Pass `a` and `b` as page paths.
- `diagnose_journey_friction`: High-friction navigation edges from recent sessions. Filter by `signal_type` or `min_friction_weight`.

**Sessions and raw data**

- `get_session_detail`: Full event timeline for one session in chronological order.
- `get_heuristics`: The complete signal catalog: every friction heuristic type with its scoring weight and thresholds.
- `query_raw_rows`: SQL-style access to allowlisted tables (`events`, `signals`, `sessions`, `page_scores`, `score_history`, `ux_issues`, `alerts`, `deploys`).
- `download_events_csv`: Raw event export as CSV.
- `explore`: A deterministic, typed query engine over session data. No natural language, no LLM, built explicitly for agents to answer ad-hoc questions the rest of the surface doesn't cover directly. Pick a `window_days` (1-90, default 7), up to 12 AND-ed `filters` over `signal`, `page`, `source`, `confused`, `converted`, `event_type` (ops `is` / `is_not` / `contains`, contains only on `page` and `source`), and exactly one `output`: `{mode: "list"}` to pull matching sessions, or `{mode: "measure", metric, group_by?}` to compute `count`, `avg_pageviews`, `conversion_rate`, `avg_dwell_ms`, or `bounce_rate` as a scalar or, with `group_by` (`page`, `source`, `day`, `signal`, `cohort`), a series. See "Explore via MCP" below for a list and a measure example.

**Deploy correlation**

- `get_deploys`: All deploys Flusterduck knows about, with `confusion_before` and `confusion_after` populated for deploys older than 5 minutes.
- `get_revenue_impact`: Revenue impact estimates for active friction. Only populated when conversion tracking is wired.
- `get_flows`: Page-to-page navigation edges from recent sessions.

**Conversion impact**

- `get_conversion_insights`: The confused-vs-calm conversion analysis: how much less confused sessions convert than calm ones, broken down by page and traffic source, plus ranked narratable insights. Pass `days` (1-90, default 7). Needs a conversion event wired; see [Conversion trigger](./conversion-trigger).

**Org-level** (requires org-scoped key)

- `get_audit_log`: Organization audit log.
- `get_degradation`: Active and recent backend degradation events. Also available to site-scoped keys, since degradation is operational health rather than tenant data.
- `get_webhook_deliveries`: Outbound webhook delivery history and failure details.

## Explore via MCP

`explore` is the odd one out on purpose: everything else in this list is a fixed shape (scores, issues, trends), but `explore` is a small closed query language over session data: a window, up to 12 AND-ed filters, and one output mode. Still fully deterministic and validated against the same allowlists the dashboard uses; there's no free-text query and no model in the loop.

**List**: sessions that rage-clicked on `/pricing` in the last 7 days:

```
window_days: 7
filters: [
  { "field": "page", "op": "is", "value": "/pricing" },
  { "field": "signal", "op": "is", "value": "rage_click" }
]
output: { "mode": "list", "limit": 20 }
```

Returns up to 20 matching sessions, most recent first, each with its page list, signal counts, source, and confused/converted flags. Use this to cite concrete example sessions as evidence in a diagnosis.

**Measure**: does confusion actually cost this site conversions?

```
window_days: 30
output: { "mode": "measure", "metric": "conversion_rate", "group_by": "cohort" }
```

Returns conversion rate as a two-point series: the `confused` cohort (sessions with at least one friction signal) versus the `calm` cohort. Swap `group_by` for `page`, `source`, `day`, or `signal` to see the same metric broken down a different way, or drop `group_by` entirely for a single scalar across all matching sessions.

## Write tools

All require `manage:write` scope.

**`update_issue`**: Change status, add a triage note, or assign an issue. Issue ids are UUIDs; get them from `get_issues`.

```
issue_id: "3a7f2c9d-4e1b-42d8-9c5a-1f6b8e2d4a7c"
status: "triaged"
note: "Confirmed on Safari iOS 17. Disabled-state styling on #place-order doesn't communicate that payment is processing."
assigned_to: "eng-lead"
```

Status options: `open`, `triaged`, `in_progress`, `verified`, `resolved`, `ignored`.

**`rate_issue`**: Rate a detected issue's accuracy. `confirmed` means it described a real problem, `rejected` means it was a false positive or noise. Ratings feed the detection accuracy metric without changing the issue's status. Pass `null` to clear a previous rating.

```
issue_id: "3a7f2c9d-4e1b-42d8-9c5a-1f6b8e2d4a7c"
verdict: "confirmed"
```

**`send_feedback`**: Send product feedback about Flusterduck itself to the Flusterduck team. Use `suggestion` for a missing capability, confusing tool output, or data you wished you had; use `bug` when a tool or surface misbehaved. This is how you (or your AI assistant, mid-workflow) tell us what to build next.

```
kind: "suggestion"
message: "get_issue cites session ids but there is no way to fetch two sessions side by side. A compare_sessions tool would save a round trip."
```

**`update_alert`**: Acknowledge, mark investigating, or resolve an alert.

```
alert_id: "9b5e1f4c-2d8a-4f3e-8b6d-7c1a5e9f2b4d"
status: "resolved"
resolved_reason: "Deploy 2026-06-09 added a spinner and disabled state to the payment submit button."
```

**`add_annotation`**: Write a timeline marker visible to the whole team.

```
message: "Redesigned billing flow launched. Monitoring confusion score on /settings/billing."
```

**`create_alert_rule`**: Create a new alert rule. Trigger types: `spike`, `anomaly`, `new_page`, `trend`, `co_occurrence`, `positive` (fires when the score drops below the threshold), `budget`, `revenue_threshold` (threshold in cents), `stalled_signups`. Omit `threshold` and `cooldown_minutes` to get sensible per-trigger defaults.

```
name: "Checkout rage click spike"
trigger_type: "spike"
threshold: 25
cooldown_minutes: 60
channels: ["email", "slack"]
page_pattern: "/checkout*"
```

**`update_alert_rule`**: Update an existing rule. Use `enabled: false` to silence it temporarily rather than deleting.

**`delete_alert_rule`**: Permanently remove a rule. Can't be undone.

## Questions that work well

- What's broken on checkout right now?
- Did the last deploy make things worse?
- Which page has the most dead clicks this week?
- Triage the open issues and tell me what to fix first.
- Write a friction summary for the PM standup.
- Which sessions show rage clicks on the upgrade button?
- How does this week's confusion score compare to last week?
- Create a spike alert for the pricing page and notify Slack.
- What's the revenue impact of the current open issues?
- Which elements on `/onboarding` are generating the most friction?
