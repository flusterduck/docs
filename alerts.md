# Alerts

Confusion scores fail in two distinct ways. They spike: a deploy breaks something and the score jumps 30 points overnight. Or they drift: friction accumulates gradually until a page that was at 35 six weeks ago is now at 62 and nobody noticed. Most monitoring catches the first type. You need both.

Each rule watches your scores against a condition you define. When the condition's met, Flusterduck fires an alert to whatever channels you've configured. Once an alert is open, it won't fire again for the same rule until the cooldown passes.

Configure rules in the dashboard under Settings > Alert Rules, via the API, or through the MCP server.

## Trigger types

### spike

Fires when a page's confusion score increases by more than `threshold` points in a short window.

```json
{
  "trigger_type": "spike",
  "threshold": 25
}
```

The default window is one scoring cycle. Use this for deploy monitoring. If you ship code and the checkout score jumps 30 points, you want to know immediately.

### anomaly

Fires when a page's score is statistically anomalous relative to its historical baseline. No fixed threshold needed: the engine calculates expected ranges from the page's own history.

```json
{
  "trigger_type": "anomaly",
  "threshold": 80
}
```

`threshold` here is anomaly confidence (0-100). At `80`, the score must be in the top 20% of unusual values before the alert fires. Useful for pages where the "normal" range shifts with traffic volume or time of year.

### new_page

Fires when a previously untracked page crosses a confusion threshold for the first time. Catches new pages that ship with friction already baked in.

```json
{
  "trigger_type": "new_page",
  "threshold": 40
}
```

### trend

Fires when a page's score has been trending upward for a sustained period. Less sensitive than `spike` but catches gradual degradation that spike detection misses.

```json
{
  "trigger_type": "trend",
  "threshold": 50
}
```

If a page climbs from 30 to 50 over three weeks without triggering a spike, `trend` catches it.

### co_occurrence

Fires when two or more signal types cluster on the same element. A button that's getting both dead clicks and rage clicks simultaneously is a different problem than either alone.

```json
{
  "trigger_type": "co_occurrence",
  "threshold": 30
}
```

`threshold` is the signal count that triggers the co-occurrence check.

### positive

Fires when a score drops below `threshold`. An improvement detector: use it to confirm a fix worked.

```json
{
  "trigger_type": "positive",
  "threshold": 20
}
```

Route positive alerts to a different channel than your incident alerts. They don't belong in `#incidents`. PagerDuty should never wake anyone up at 2am because a score improved.

### budget

Fires when a page's confusion score exceeds an absolute ceiling you've set. Unlike `spike` (which is relative to recent history), `budget` doesn't care about rate of change, only whether you've crossed the line.

```json
{
  "trigger_type": "budget",
  "threshold": 60
}
```

Your checkout page probably shouldn't ever be above 50. Set a budget alert and you don't have to remember to check.

## Channels

| Channel | Config |
|---|---|
| `email` | Fires to the addresses configured under Settings > Notifications |
| `slack` | Fires to the channel configured under Settings > Integrations > Slack |
| `webhook` | Delivers a POST to your registered webhook endpoint |
| `pagerduty` | Creates and resolves PagerDuty incidents via your Events API v2 key |

Combine channels. Most teams use `["email", "slack"]` for spike rules and add `"pagerduty"` for budget rules on critical pages.

## Page pattern

Scope a rule to specific pages with glob-style patterns:

```
/checkout           exact match
/checkout*          /checkout and any sub-path
/pricing            exact match
/*                  all pages (default if omitted)
/app/*              all pages under /app
```

## Rule configuration

Full schema for creating or updating a rule:

```json
{
  "name": "Checkout rage click spike",
  "trigger_type": "spike",
  "threshold": 25,
  "cooldown_minutes": 60,
  "channels": ["email", "slack"],
  "page_pattern": "/checkout*",
  "slack_channel": "#incidents",
  "enabled": true
}
```

| Field | Type | Constraints |
|---|---|---|
| `name` | string | Required |
| `trigger_type` | string | One of the 7 types above |
| `threshold` | number | 0-1000 |
| `cooldown_minutes` | number | 1-1440 |
| `channels` | array | Any combination of supported channel types |
| `page_pattern` | string | Glob pattern, defaults to `/*` |
| `slack_channel` | string | Optional `#channel-name` or Slack channel ID for Slack alerts |
| `enabled` | boolean | Defaults to `true` |

## Alert lifecycle

Alerts move through four states: `open`, `acknowledged`, `investigating`, `resolved`.

`open` means the rule fired and nobody's acted on it. You'll keep getting notifications until someone acknowledges it.

`acknowledged` means someone's aware. Escalation stops (no repeat notifications) but the alert stays open. Acknowledge it as soon as someone takes ownership, even before you know the cause.

`investigating` is optional but useful for team coordination. It signals that someone is actively working the problem.

`resolved` closes the alert. If you tagged the deploy that fixed the underlying issue, the scoring engine verifies the fix held and marks related issues as `verified`. If the condition is met again later, a new alert fires.

To suppress a signal you've decided not to fix, use `ignored` status on the underlying issue. Alert rules won't fire for issues in `ignored` status.

## Managing rules

With an MCP key carrying `manage:write` scope, your AI assistant manages rules directly:

```
List all my alert rules and show me which ones are disabled.
Create a rage click spike alert for the pricing page.
Disable the checkout budget alert until the redesign ships.
```

See [API reference](./api) for request examples.

## Suggested starting rules

A set that covers the most common failure modes:

1. **Post-deploy spike**: `spike`, threshold 20, all pages, 60-minute cooldown, email + slack
2. **Critical page budget**: `budget`, threshold 50, `/checkout*` and `/pricing*`, email + pagerduty
3. **New page check**: `new_page`, threshold 40, all pages, 24-hour cooldown, email
4. **Fix confirmation**: `positive`, threshold 20, all pages, email only (route to a wins channel)
