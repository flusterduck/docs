# What is Flusterduck?

Flusterduck is automatic issue tracking for UX friction.

It watches real user behavior, detects where people get stuck, clusters repeated friction into fixable issues, and verifies whether each fix worked after you ship. No session replay. No watching recordings. No manual digging.

## The problem it solves

Most analytics tools tell you where users dropped off. Session replay shows you recordings you have to watch. Neither one automatically tells you what's wrong, why, or what to fix first.

There's a gap between "conversion dropped 8% on checkout" and "the continue button looks disabled because of the grey border, and 140 users have dead-clicked it in the last two weeks." Flusterduck fills that gap.

## How it works

The SDK runs in the browser. It listens for behavioral signals, 131 types in all (rage clicks, dead clicks, form abandonment, scroll patterns, keyboard confusion, mobile tap misses, navigation loops, and more), and sends them to the scoring engine. It never records keystrokes, form values, or page text.

The scoring engine does four things with those signals:

- Computes a confusion score for each page based on signal frequency, weight, and recency
- Clusters repeated signals from different users into UX issues with session evidence
- Estimates revenue impact when you wire conversion events with `track()`
- Runs verification after each deploy to check whether a fix actually reduced friction

## What you get

**Issues** are the main output. Each one has a title, root cause hypothesis, affected element, evidence sessions, severity score, and status. They work like Linear tickets: assign them, triage them, mark them resolved. The scoring engine re-checks them after each deploy to confirm the fix held.

**Scores** give you a per-page confusion rating. Useful for prioritizing what to look at first, and for tracking whether a page is getting better or worse over time.

**Alerts** fire when a score spikes after a deploy, when a new friction pattern emerges, or when a page crosses a threshold you set. You can route them to email, Slack, PagerDuty, or a webhook.

**MCP tools** give your AI assistant direct access to your friction data. Instead of opening a dashboard, you ask Claude what's broken. It queries live scores, pulls open issues, and answers you. You can also run post-deploy checks and generate weekly summaries without touching a UI.

## What Flusterduck never does

No session replay. No DOM recording. No form values. No text content. No keystroke capture. No raw IP storage. No PII of any kind. It measures behavior patterns, not personal data.

The SDK's privacy constraints are enforced at the code level, not just by policy. It cannot record what users type because it doesn't attach the kind of listeners that would capture it.

## Pricing

Paid product. No free tier. No open-source version.

| Plan | Monthly | Sessions | Sites | Key features |
|---|---|---|---|---|
| Grow | $99 | Up to 50,000 | 1 | Core friction detection, issues, alerts, MCP access, 2 autofixes a month |
| Scale | $249 | Up to 250,000 | 5 | Everything in Grow, plus team members, Slack integration, 10 autofixes a month |
| Pro | $499 | Up to 1,000,000 | 10 | Everything in Scale, plus unlimited members, revenue estimates, priority support, 30 autofixes a month |
| Enterprise | Custom | Custom | Unlimited | Custom volume, SSO, dedicated support, SLA |

Session limits are pooled across all sites in the org, not per-site. All plans include a 3-day free trial with a card on file. You are not charged until the trial ends, and you can cancel any time before then.
