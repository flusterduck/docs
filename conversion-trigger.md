# Conversion trigger

Flusterduck already knows where people get confused. The conversion trigger tells it what *success* looks like on your site: the single event that means a visitor did the thing you wanted. Once that's wired, Flusterduck can prove the part that matters to the business: **confused sessions convert less than calm ones**, and it can put a number on the gap.

This page covers the two steps to set it up, then the confused-vs-calm insight it unlocks.

## Why this exists

Confusion scores tell you *where* friction is. They don't, on their own, tell you whether that friction costs you anything. A page can be a little confusing and still convert fine; another can look calm and quietly leak sign-ups.

The conversion trigger closes that loop. When Flusterduck knows which event counts as a conversion, it splits every session into two cohorts, **confused** (produced at least one friction signal) and **calm** (zero friction signals), and compares how often each cohort converts. The difference is the money story: "confused sessions convert 12 points lower than calm ones, and it's worst on `/checkout`."

## Step 1: Fire the conversion event from the SDK

Conversions are ordinary business events, sent with `track()`. Call it the moment a conversion actually completes: in your post-payment success handler, your thank-you page, or wherever you already know the visitor succeeded.

```ts
import { track } from 'flusterduck'

// In your success handler, once the purchase is confirmed
track('purchase_completed', {
  plan_id: 'scale',
  amount_cents: 9900,
  billing: 'monthly',
})
```

Flusterduck recognizes three realized-purchase events out of the box:

| Event | Fire it when |
|---|---|
| `checkout_completed` | An order or checkout finishes |
| `purchase_completed` | A one-time purchase is confirmed |
| `subscription_started` | A subscription or trial-to-paid conversion completes |

Pick the one that best names your success moment. If you have several (a store with both one-off orders and subscriptions), you can fire more than one. Step 2 lets you choose which counts.

The same privacy rules as [Revenue impact](./revenue) apply: never pass PII, form values, or typed content. Safe properties are `plan_id`, `amount_cents` (integer cents), `billing`, `currency`, `quantity`, `product_id`.

> The SDK sends `track('purchase_completed', …)` as a business event named `purchase_completed`. That name is exactly what you'll reference in Step 2.

## Step 2: Choose which event counts as the conversion

Tell Flusterduck which business event is *the* conversion for this site. Open **Settings → Revenue** and set the **Conversion goal** field to the event name you fired in Step 1 (for example, `purchase_completed`).

Under the hood this is stored as `revenue_config.conversion_goal` on the site's production SDK config. If you manage config through the API, set it with the `manage` write API:

```bash
curl -X PATCH "https://api.flusterduck.com/v1/manage/sdk-config" \
  -H "Authorization: Bearer <jwt>" \
  -H "Content-Type: application/json" \
  -d '{
    "site_id": "<site_id>",
    "environment": "production",
    "revenue_config": { "conversion_goal": "purchase_completed" }
  }'
```

**Leaving the goal unset is a valid choice.** With no `conversion_goal`, Flusterduck counts *any* realized-purchase event (`checkout_completed`, `purchase_completed`, or `subscription_started`) as a conversion. Set an explicit goal when you fire several of those events but only one of them is the outcome you care about. The goal narrows the definition to exactly that event.

## The insight it unlocks

With a conversion firing, the confused-vs-calm analysis becomes available. It reports:

- **Cohort comparison**: conversion rate, average session duration, pages per session, and bounce rate for confused vs calm sessions, plus the deltas between them.
- **By page**: where the conversion gap is widest, so you know which confusing page costs the most conversions.
- **By source**: which traffic sources (UTM source / referrer host) lose the most conversions to confusion.
- **Ranked insights**: pre-written, narratable findings such as "Confused visitors convert 12 points lower." Each carries a confidence flag; a cohort under the minimum sample size is marked low-confidence so you don't over-read early data.

### Read it three ways

**REST**: `GET /v1/query/insights?site_id=<id>&days=7` (window is 1–90 days, default 7):

```bash
curl "https://api.flusterduck.com/v1/query/insights?site_id=<site_id>&days=30" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

**CLI**: the [CLI](./cli) prints a readable summary: the headline gap, the top per-page gaps, the hardest-hit sources, and the ranked insights.

```bash
flusterduck insights --site <site_id> --days 30
```

**MCP**: the `get_conversion_insights` tool (local [MCP server](./mcp) and the remote Cloudflare worker) returns the same analysis for your AI assistant, so you can just ask:

```
How much less do confused sessions convert than calm ones?
Which page loses the most conversions to confusion?
Which traffic source is hit hardest by confusion?
```

## Sampling and confidence

The engine needs a floor of sessions in **each** cohort before it will make a confident claim (the `min_sample` in the response, 30 by default). Until both cohorts clear it, the analysis still returns, but the headline insight reads "Still gathering data" and cohorts are flagged `low_confidence`. This is deliberate: a 3-session cohort can show a wild, meaningless gap. Give it real traffic before acting on the number.

## Relationship to revenue impact

The conversion trigger and [Revenue impact](./revenue) are complementary. Revenue impact attaches dollar figures to individual open issues using the `amount_cents` you pass. The confused-vs-calm insight is the broader, site-wide picture: it doesn't need dollar amounts (only a conversion event) to show that confusion is depressing conversions and where. Wire the conversion event once and both get sharper.
