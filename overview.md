# What is Flusterduck?

Flusterduck is automatic issue tracking for UX friction.

It watches real user behavior, finds repeated friction, investigates the supporting sessions, and verifies whether each fix worked after you ship. No session replay. No watching recordings. No manual digging.

## The problem it solves

Most analytics tools tell you where users dropped off. Session replay shows you recordings you have to watch. Neither one turns repeated behavior into a located issue with cited sessions and checked claims.

There's a gap between "conversion dropped 8% on checkout" and "140 users dead-clicked the continue button in the last two weeks." Flusterduck fills that gap without guessing why it happened.

## How it works

The SDK runs in the browser. It listens for behavioral signals, 132 types in all (rage clicks, dead clicks, form abandonment, scroll patterns, keyboard confusion, mobile tap misses, navigation loops, and more), and sends them to the scoring engine. It never records keystrokes or form values. For an element involved in a signal, it may send a short, PII-redacted accessible label so the evidence is readable.

Flusterduck does four things with those signals:

- Computes a confusion score for each page based on signal frequency, weight, and recency
- Sends repeated patterns to the AI investigator, which can file an issue only after its evidence passes the server's checks
- Estimates revenue impact when you wire conversion events with `track()`
- Runs verification after each deploy to check whether a fix actually reduced friction

## What you get

**Issues** are the main output. Each one has a checked description, affected element, cited sessions, severity score, and status. They work like Linear tickets: assign them, triage them, mark them resolved. Flusterduck re-checks them after each deploy to confirm the fix held.

**Scores** give you a per-page confusion rating. Useful for prioritizing what to look at first, and for tracking whether a page is getting better or worse over time.

**Alerts** fire when a score spikes after a deploy, when a new friction pattern emerges, or when a page crosses a threshold you set. You can route them to email, Slack, PagerDuty, or a webhook.

**MCP tools** give your AI assistant direct access to your friction data. Instead of opening a dashboard, you ask Claude what's broken. It queries live scores, pulls open issues, and answers you. You can also run post-deploy checks and generate weekly summaries without touching a UI.

## What Flusterduck never does

No session replay. No DOM recording. No form values. No user-typed text. No keystroke capture. No raw IP storage. It measures behavior patterns through structured, pseudonymous events rather than recordings.

The SDK's privacy constraints are enforced in code and policy. It cannot record what users type because it doesn't attach the kind of listeners that would capture it.

## Pricing

Paid product. No free tier. No open-source version.

| Plan | Monthly | Sessions | Sites | Team | AI |
|---|---|---|---|---|---|
| Grow | $99 | Up to 50,000 | 1 | 3 members | AI diagnosis, 2 autofixes a month |
| Scale | $249 | Up to 250,000 | 5 | 10 members | AI diagnosis, 10 autofixes a month |
| Pro | $499 | Up to 1,000,000 | 10 | Unlimited | AI diagnosis, 30 autofixes a month |
| Enterprise | Custom | Custom | Unlimited | Unlimited | AI diagnosis, unlimited autofixes |

Every plan gets the full detection engine: all 132 friction signals, issues, alerts, integrations, and MCP access. The tiers differ on volume, sites, seats, and AI allowances, not on which detectors you're allowed to run.

Session limits are pooled across all sites in the org, not per-site. Each organization gets one 3-day self-serve trial on Grow, Scale, or Pro. A card is required, you pay $0 today, and billing starts after 3 days unless you cancel. Enterprise terms are arranged separately.

The trial includes capped AI detection, AI diagnosis and composition, Guide explanations, and one manual Autofix. Those allowances cover the three-day trial and do not reset if the month changes. Scheduled AI triage and Autofix Autopilot require paid billing.
