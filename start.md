# Your First Data

The SDK is installed and signals are flowing. Here's what to do with them.

## Read your scores

Open the dashboard. Every tracked page has a confusion score from 0 to 100. Higher is worse. A page at 15 is fine. A page at 60 has real friction. A page at 80 is actively costing you conversions.

The score accounts for signal type, frequency, and recency. A single `rage_click` on your checkout button today matters more to the score than 10 `passive_drift` signals from last month.

New sites take 24-48 hours to produce meaningful scores. You need enough signal volume for the clustering to kick in.

## Look at your first issues

Issues are what you act on. Each one has a title, an affected element, a signal count, and a severity score. Sort by severity first. Those are the ones where friction is concentrated on a specific element, not just diffuse across a page.

The diagnosis on each issue says what the cited evidence supports. When the evidence doesn't prove a cause, the issue leaves the cause unknown instead of filling in a guess.

Issues work like tickets. Assign them, update the status as you investigate, mark them resolved after you deploy a fix. The scoring engine re-checks resolved issues after each deploy to confirm the fix held.

## Wire conversion data

If you want revenue impact estimates, add `track()` calls around your key conversion events:

```ts
track('plan_intent', { plan_id: 'scale', amount_cents: 9900, billing: 'monthly' })
track('subscription_started', { plan_id: 'scale', amount_cents: 9900, billing: 'monthly' })
track('checkout_abandoned', { plan_id: 'grow', amount_cents: 3900 })
```

The scoring engine correlates friction signals with abandonment and estimates how much revenue each open issue is likely costing. It takes a few days of data to produce stable estimates.

## Set up at least two alerts

Don't wait until users complain. Set up a spike alert for your most critical page, and a budget alert as a ceiling:

**Spike alert** fires when a page score increases sharply after a deploy. Catches regressions before they compound.

**Budget alert** fires when a page crosses a score threshold you set. Your checkout page probably shouldn't ever be above 50.

Both take 2 minutes to configure under Settings > Alert Rules. Route the spike alert to Slack or email. Add PagerDuty to the budget alert if the page is critical enough.

## Record your deploys

When you ship a fix, tell Flusterduck a deploy happened. That's what triggers the before/after comparison. Run this from CI:

```bash
npx flusterduck deploy notify --site <site_id> --key fd_sec_xxxxxxxxxxxx
```

It auto-detects the commit hash, message, and author on GitHub Actions, Vercel, GitLab, and Bitbucket, so a bare call with no flags works in most pipelines. Use `--commit`, `--message`, and `--author` to override, and `--env` if you're deploying somewhere other than production.

Flusterduck captures the confusion score at the moment of the deploy, then again once the deploy is a few minutes old. This is how you confirm a fix worked: the scoring engine shows you the before/after delta and verifies that the affected issues actually resolved. If you're on GitHub, deploys are often detected automatically from your deployment history, but calling `deploy notify` explicitly is the reliable path and works with any host.

## Connect your AI assistant

Once your site has a few days of data, the MCP integration pays off fast. Instead of checking the dashboard, ask Claude:

> "What's the worst-performing page right now, and what does the evidence prove?"

See [MCP setup](./mcp) for a 5-minute configuration walkthrough.

## Share access

Add your teammates under Settings > Members. Engineers can triage issues. PMs can read scores and run weekly summaries. Admins can manage alert rules and integrations.

If your team uses the AI assistant heavily, create a read-only `fd_mcp_` key so teammates can query friction data without access to your write operations.

That's the whole setup. Everything past this point is the duck's problem.
