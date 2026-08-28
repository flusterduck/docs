# Alerts

Confusion scores fail in two distinct ways. They spike: a deploy breaks something and the score jumps 30 points overnight. Or they drift: friction accumulates gradually until a page that was at 35 six weeks ago is now at 62 and nobody noticed. Most monitoring catches the first type. You need both.

Each rule watches your scores against a condition you define. When the condition's met, Flusterduck fires an alert to whatever channels you've configured. Once an alert is open, it won't fire again for the same rule until the cooldown passes.

Configure rules in the dashboard under Settings > Alert Rules, via the API, or through the MCP server.

## Trigger types

### spike

Fires when a page's raw (pre-normalization) score sits more than `threshold` points above its baseline mean. Requires a baseline to exist, so a brand-new page can't trigger this one; use `new_page` for that.

```json
{
  "trigger_type": "spike",
  "threshold": 60
}
```

Default threshold is 60. Use this for deploy monitoring. If you ship code and the checkout score jumps 60 raw points above its usual level, you want to know immediately.

### anomaly

Fires when a page's score deviates from its baseline by more than a set number of standard deviations. `threshold` encodes that minimum deviation, times 10: a threshold of `20` means the current score has to be at least 2.0 standard deviations above the time-adjusted baseline before the alert fires.

```json
{
  "trigger_type": "anomaly",
  "threshold": 20
}
```

Default threshold is 20 (2.0 sigma). Raise it (`30` for 3.0 sigma) to fire less often; lower it (`10` for 1.0 sigma) to fire on smaller deviations. Useful for pages where the "normal" range shifts with traffic volume or time of day, since the baseline this compares against is already adjusted for hour-of-day and day-of-week.

### new_page

Fires when a previously untracked page (no baseline yet) crosses a confusion score of `threshold` for the first time. Catches new pages that ship with friction already baked in.

```json
{
  "trigger_type": "new_page",
  "threshold": 40
}
```

Default threshold is 40.

### trend

Fires when a page's current score classifies as trending `up` (more than one standard deviation above its time-adjusted baseline) and the score itself is at least `threshold`.

```json
{
  "trigger_type": "trend",
  "threshold": 30
}
```

Default threshold is 30. This is less sensitive to a single bad session than `spike`, because it only fires on the sustained-deviation trend classification, not a one-off jump.

### co_occurrence

Fires when a page's co-occurrence multiplier is active (10 or more distinct users showed friction on the page within the same scoring window) and the number of affected users is at least `threshold`.

```json
{
  "trigger_type": "co_occurrence",
  "threshold": 10
}
```

Default threshold is 10, the same floor where the multiplier itself turns on. Raise it to only fire on larger simultaneous pile-ups (the multiplier steps up again at 20 and 50 concurrent users; see [Scoring internals](./scoring-internals)).

### positive

Fires when a page's raw score drops below `threshold` **and** that raw score is at least 30% below the page's baseline mean. The 30% floor exists so this doesn't fire on ordinary quiet moments, only on a real, sustained improvement. An improvement detector: use it to confirm a fix worked.

```json
{
  "trigger_type": "positive",
  "threshold": 30
}
```

Default threshold is 30. Requires a baseline to exist.

Route positive alerts to a different channel than your incident alerts. They don't belong in `#incidents`. PagerDuty should never wake anyone up at 2am because a score improved.

### budget

Fires when a page has spent `threshold` percent or more of the current period above its confusion budget ceiling. See [Confusion budgets](./scoring) for how the ceiling (`config.target_score`) and the burn threshold relate.

```json
{
  "trigger_type": "budget",
  "threshold": 70,
  "config": { "target_score": 50, "period": "weekly" }
}
```

Default threshold is 70, default `target_score` is 50, default `period` is `weekly`. `threshold` here is a burn percentage (0-100), not a score: it does not mean "fire when the score crosses 70." It means "fire once the page has spent 70% of the period above `target_score`."

### revenue_threshold

Fires when the total revenue at risk across your open issues crosses `threshold` cents. Site-level, not per-page.

```json
{
  "trigger_type": "revenue_threshold",
  "threshold": 500000
}
```

`threshold` is in cents, so a $5,000/mo rule is `500000`, not `5000`. Default threshold is 50000 ($500/mo). Requires open issues with a populated revenue estimate; see [Revenue impact](./revenue).

### stalled_signups

Fires when at least `threshold` new sessions in the trailing 24 hours hit friction and never reached your conversion goal. Site-level. This is the alert half of activation tracking: it watches for warm signups dying on the way in, not for friction on an established page.

```json
{
  "trigger_type": "stalled_signups",
  "threshold": 5
}
```

Default threshold is 5. Uses the conversion goal configured under Settings > Revenue (see [Conversion trigger](./conversion-trigger)) to decide what counts as "reached." One digest alert per cooldown window covers every stalled session found in the window, not one alert per session.

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
  "threshold": 60,
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
| `trigger_type` | string | One of the 9 types above |
| `threshold` | number | Range and default depend on `trigger_type`: `spike`/`anomaly` 0-1000 (default 60/20), `new_page`/`trend`/`positive`/`budget` 0-100 (default 40/30/30/70), `co_occurrence` 0-100000 (default 10), `revenue_threshold` 0-1,000,000,000 cents (default 50000), `stalled_signups` 1-1,000,000 (default 5) |
| `config` | object | Budget rules only: `{ target_score: number, period: "daily" \| "weekly" \| "monthly" }`. Also accepts `slack_channel` on any trigger type as an alternative to the top-level field |
| `cooldown_minutes` | number | 1-1440 |
| `channels` | array | Any combination of supported channel types |
| `page_pattern` | string | Glob pattern, defaults to `/*`. Ignored by the site-level trigger types (`revenue_threshold`, `stalled_signups`) |
| `slack_channel` | string | Optional `#channel-name` or Slack channel ID for Slack alerts |
| `severity_overrides` | object | Optional `{ critical?: number, high?: number, warning?: number }`, in the same units as `threshold`, to override the default severity bands for this rule |
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

1. **Post-deploy spike**: `spike`, threshold 60, all pages, 60-minute cooldown, email + slack
2. **Critical page budget**: `budget`, threshold 70, `config: { target_score: 50 }`, `/checkout*` and `/pricing*`, email + pagerduty
3. **New page check**: `new_page`, threshold 40, all pages, 24-hour cooldown, email
4. **Fix confirmation**: `positive`, threshold 30, all pages, email only (route to a wins channel)
5. **Stalled signups**: `stalled_signups`, threshold 5, 60-minute cooldown, email + slack
