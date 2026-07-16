# Confusion Scores

A confusion score is a number from 0 to 100 that measures how much friction a page generates relative to its session volume. Direct measurement of how often users hit behavioral friction patterns. Not a satisfaction proxy, not survey data.

Most pages sit between 10 and 40. Above 50 is a page worth prioritizing. Above 70 is a page that's actively costing you.

## How scores are calculated

Every signal has a weight based on its tier. Tier 1 signals (rage clicks, dead clicks, form abandonment on a conversion element) carry roughly 3.5x the weight of tier 3 signals (passive drift, hover thrash). The engine sums the weighted signals for a page, normalizes for session volume, and applies recency decay so old signals fade out.

Volume normalization matters because raw counts don't. Ten rage clicks across 20 sessions is a different problem than ten rage clicks across 2,000 sessions. The rate is what counts, not the total.

Recency decay means last week's friction matters more than the same friction pattern from three months ago. The half-life is 14 days. A page that used to be bad but isn't anymore will show that in the score.

## Score components

The engine weighs five factors:

**Signal frequency**: signals per session on this page. The rate, not the total.

**Tier weighting**: tier 1 signals multiply by 3.5x relative to tier 3. A single rage click on your checkout button moves the score more than fifty passive drifts.

**Recency factor**: 14-day half-life. Recent signals are weighted more heavily.

**Session coverage**: the percentage of sessions that contained at least one signal on this page. A page where 40% of sessions produce friction is different from a page where 4% do, even if the total signal count is the same.

**Co-occurrence bonus**: when multiple signal types cluster on the same element (rage clicks and dead clicks both on the same button), the score increases beyond what each signal type would contribute individually. Co-occurrence is usually more meaningful than either signal type alone.

## Baselines

Each page has a rolling 7-day baseline. The baseline is used to calculate:

- `trend`: whether the page is `up`, `down`, or `stable` relative to recent history
- Spike alert thresholds (a `spike` alert fires when the current score exceeds baseline + threshold)
- Anomaly detection (the `anomaly` trigger compares current score to historical variance, not a fixed number)

New pages don't have a baseline until they've accumulated 7 days of data. During that window, relative alerts won't fire for that page. Budget and new-page alerts still will.

## Confusion budgets

A confusion budget is an absolute ceiling for a specific page. Set it and forget it. When the page crosses that number, a `budget` alert fires regardless of whether the score changed recently.

Budgets are set per-page in the dashboard under Settings > Alert Rules, or via the API:

```json
{
  "trigger_type": "budget",
  "threshold": 50,
  "page_pattern": "/checkout*",
  "channels": ["email", "pagerduty"],
  "name": "Checkout budget"
}
```

Most teams set budgets on their highest-value pages. Reasonable starting numbers:

| Page | Budget |
|---|---|
| Checkout | 50 |
| Pricing | 45 |
| Onboarding | 40 |
| Account/billing settings | 35 |

These are starting points, not rules. Set them based on your own baseline data once you've been running for a few weeks.

The budget concept is simpler than anomaly detection. You don't need historical data. You're just defining what "too broken" looks like for a specific page.

## Score interpretation

| Score | What it means |
|---|---|
| 0-20 | Clean. No meaningful friction patterns. |
| 21-40 | Normal. Some friction, nothing alarming. |
| 41-60 | Elevated. At least one issue worth investigating. |
| 61-80 | High. Multiple users hitting the same problems. |
| 81-100 | Critical. Something is broken or completely baffling to users. |

These are rough guides, not absolute thresholds. A score of 45 on a low-traffic onboarding page reads differently than a score of 45 on checkout with 10,000 daily sessions. Use scores for comparison and trend tracking.

## What moves scores quickly

**`rage_click` and `dead_click`** on a critical element will move a page score significantly in a single session. That's by design. Those signals are high-confidence.

**`form_abandonment`** on a conversion form gets treated as tier 1 regardless of the page tier. If users are abandoning your checkout form, the engine weights it accordingly.

**`error_recovery_loop`** and `silent_failure_retry` are also fast movers. They indicate users who hit a broken state and keep trying the same thing, which is a very specific kind of friction.

**Tier 3 signals** (passive drift, thrash hover) need volume to move the needle. Twenty instances on the same element from different sessions will. Five won't.

## Scores via the API

```bash
curl "https://api.flusterduck.com/v1/scores" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

```json
{
  "scores": [
    {
      "page": "/checkout",
      "score": 72,
      "trend": "up",
      "top_signal": "rage_click",
      "issue_count": 3,
      "updated_at": "2026-06-10T14:22:05Z"
    },
    {
      "page": "/pricing",
      "score": 41,
      "trend": "stable",
      "top_signal": "dead_click",
      "issue_count": 1,
      "updated_at": "2026-06-10T14:21:47Z"
    }
  ]
}
```

Get score history for a specific page:

```bash
curl "https://api.flusterduck.com/v1/trends?page=/checkout&days=30" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

## Scores via MCP

```
What pages have the highest confusion scores right now?
How has the /checkout score trended over the past two weeks?
Which pages crossed their confusion budgets this week?
Show me all pages with a score above 50.
```

## How scores relate to issues

Scores are a summary. Issues are the detail. A high score means there's friction on the page. Issues tell you specifically what the friction is, which element it's on, how many users hit it, and what the likely cause is.

A page can have a high score with no open issues if the signals are dispersed across many elements without enough clustering for the engine to create an issue. That's unusual but possible. It usually means several small problems rather than one big one.

A page can also have open issues and a moderate score if the issues are being managed and the fix is in progress. The score reflects live signal volume, not the issue count.

Start with scores to prioritize pages. Then read issues to know what to fix.
