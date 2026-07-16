# Slack Integration

Connect once and your team gets friction alerts in Slack, interactive buttons to triage them without opening a dashboard, and a slash command to pull live scores and open issues from any channel.

## Setup

1. Go to Settings > Integrations > Slack in your Flusterduck dashboard.
2. Click "Connect Slack."
3. Authorize the Flusterduck app for your workspace and select the default alert channel.

After connecting, you can route individual alert rules to specific channels. The `/flusterduck` slash command becomes available workspace-wide.

Slack incoming webhooks are tied to the default channel selected during install. Per-rule channel routing uses the Flusterduck bot token and Slack's `chat.postMessage` API instead.

## Slash commands

### /flusterduck scores

Returns your top pages ranked by current confusion score with trend indicators:

```
/flusterduck scores

/checkout        72  ↑  (was 58, +14)
/pricing         41  →
/onboarding      38  ↑  (was 31, +7)
/account         24  ↓  (was 29, -5)
/dashboard       19  →
```

### /flusterduck issues

Lists open and triaged issues ranked by severity:

```
/flusterduck issues

[HIGH]  Dead clicks on #place-order          /checkout    47 signals
[HIGH]  Form abandonment at billing step     /checkout    31 signals
[MED]   Rage clicks on upgrade CTA           /pricing     22 signals
[LOW]   Navigation loop on /settings/team                  8 signals
```

### /flusterduck issues [page]

Scopes the list to a specific page:

```
/flusterduck issues /checkout
```

### /flusterduck help

Prints available commands and syntax.

## Alert messages

When an alert rule fires, Flusterduck posts to the configured channel:

```
ALERT: Checkout confusion spike
Page:       /checkout
Score:      72  (was 46, +26)
Trigger:    spike (+20 threshold)
Rule:       Post-deploy spike
Top signal: rage_click (31 new)

[Acknowledge]  [Start investigating]  [View in dashboard]
```

Each message includes interactive buttons to move the alert through its lifecycle without leaving Slack.

## Interactive buttons

**Acknowledge**: marks the alert `acknowledged`. Escalation stops. Use this the moment someone takes ownership, even before you know the cause.

**Start investigating**: moves the alert to `investigating`. Signals to the rest of the team that it's actively being worked.

**Resolve**: closes the alert. If a deploy fixed the underlying issue, the scoring engine will verify it on the next cycle and mark related issues `verified`.

These buttons require the Flusterduck bot to be a member of the channel the alert routes to. If you add a new private channel or a public channel where your workspace requires membership, invite it: `/invite @Flusterduck`.

## Alert routing

Route individual alert rules to specific channels. Most teams separate noise from urgency:

| Channel | Alert types |
|---|---|
| `#incidents` | Spike alerts on critical pages (`/checkout`, `/pricing`) |
| `#product` | New issue alerts, weekly summaries, positive (improvement) alerts |
| `#eng` | Budget alerts, anomaly alerts, regression alerts |

Set the channel per-rule when creating or editing in Settings > Alert Rules, or via the API:

```json
{
  "name": "Checkout spike",
  "trigger_type": "spike",
  "threshold": 20,
  "channels": ["slack"],
  "page_pattern": "/checkout*",
  "slack_channel": "#incidents"
}
```

Positive alerts should never go to `#incidents`. Route them to a wins channel so they don't get lost in incident noise.

If a rule has `slack` as a channel but no `slack_channel`, Flusterduck falls back to the Slack incoming webhook and posts to the install-selected default channel.

## Bot permissions

The Flusterduck Slack app requests three permissions:

- `chat:write`: post alert messages and slash command responses
- `commands`: respond to `/flusterduck`
- `reactions:write`: react to alert messages when their status changes (a checkmark when an alert is resolved)

It doesn't read message history, access DMs, or join channels it hasn't been invited to.

## Disconnecting

Settings > Integrations > Slack > Disconnect.

Alert rules that had `slack` as a channel will stop delivering to Slack. The rules themselves stay active. If you have other channels configured (email, webhook), those keep firing.

Reconnecting re-enables Slack delivery for all rules that had it configured. No rule changes needed.
