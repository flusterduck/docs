# Revenue Impact

Wire conversion events with `track()` and Flusterduck starts putting dollar amounts on UX issues. Instead of "47 users are rage-clicking the checkout button," you get "47 users are rage-clicking the checkout button and we estimate $5,200/month in lost revenue from sessions with that pattern."

That's the number that gets engineers to prioritize UX bugs over feature work.

## What to track

Three events power revenue estimates:

```ts
import { track } from 'flusterduck'

// User selects a plan or clicks an upgrade CTA
track('plan_intent', {
  plan_id: 'scale',
  amount_cents: 9900,
  billing: 'monthly',
})

// Purchase completed
track('subscription_started', {
  plan_id: 'scale',
  amount_cents: 9900,
  billing: 'monthly',
})

// Checkout started but not completed
track('checkout_abandoned', {
  plan_id: 'grow',
  amount_cents: 3900,
})
```

Where each one belongs in your code:

**`plan_intent`**: when a user clicks a pricing CTA or selects a plan tier. Before the payment form. This tells the engine what was at stake in sessions that didn't convert.

**`subscription_started`**: in your post-payment success handler or your Stripe webhook. This establishes the baseline conversion rate.

**`checkout_abandoned`**: when a user leaves the checkout flow. Use your payment provider's abandon detection or wire it to `onbeforeunload` on your checkout page.

## Safe properties

Never pass PII, form values, or any user-typed content. The engine only needs these:

| Property | Type | Notes |
|---|---|---|
| `plan_id` | string | Your internal plan identifier (`"grow"`, `"scale"`, `"enterprise"`) |
| `amount_cents` | number | Integer. In cents, not dollars. `9900` = $99.00 |
| `billing` | string | `"monthly"` or `"annual"` |
| `currency` | string | ISO 4217 code. Defaults to `"USD"` |
| `quantity` | number | Seat count, unit count |
| `product_id` | string | For multi-product apps |

## How the estimate works

The engine correlates friction patterns with conversion outcomes:

1. For each open issue, it identifies the sessions containing that friction pattern.
2. It compares the conversion rate of those sessions against sessions on the same page with no friction signals.
3. The conversion gap gets monetized using the average `amount_cents` from recent `subscription_started` events.
4. That's projected across the monthly session volume for that page.

The result is directionally accurate, not exact. It assumes the friction is causing the conversion gap, which is usually right but not always. A page that's down or slow will produce high friction signals and low conversions, but the fix isn't a UX change. Use the revenue estimate as a prioritization signal, not a forecast.

## Reading revenue estimates

```bash
curl "https://api.flusterduck.com/v1/revenue" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

```json
{
  "total_at_risk_monthly": 8400,
  "currency": "USD",
  "issues": [
    {
      "issue_id": "iss_xxxxxxxxxxxx",
      "title": "Dead clicks on complete purchase button",
      "page": "/checkout",
      "revenue_at_risk_monthly": 5200,
      "affected_sessions_pct": 14,
      "conversion_gap_pct": 8.3
    },
    {
      "issue_id": "iss_xxxxxxxxxxxx",
      "title": "Form abandonment at billing step",
      "page": "/checkout",
      "revenue_at_risk_monthly": 3200,
      "affected_sessions_pct": 9,
      "conversion_gap_pct": 5.1
    }
  ]
}
```

`affected_sessions_pct` is the percentage of checkout sessions that contain this friction pattern. `conversion_gap_pct` is the difference in conversion rate between affected and unaffected sessions.

Revenue estimates are only populated when:
- You've called `track('subscription_started', ...)` at least 50 times in the past 30 days (the engine needs enough data to establish a baseline conversion rate)
- There are open issues on pages where conversion events were tracked
- The issue's affected sessions have a statistically meaningful conversion gap

If the `/revenue` endpoint returns empty `issues`, you either haven't tracked enough conversions yet or there aren't enough sessions to compute a meaningful gap.

## Revenue via MCP

```
What's the revenue impact of the current open issues?
Which issue is costing us the most?
Rank the open issues by revenue at risk.
```

The `get_revenue_impact` tool returns the same data as the API endpoint, formatted for your AI assistant to reason about.

## Annual vs monthly

The estimate uses `billing` to annualize correctly. If most of your conversions are annual, the per-session value is higher and the revenue-at-risk number will reflect that. Pass `billing: "annual"` in `subscription_started` events for annual subscribers and `billing: "monthly"` for monthly subscribers.

If you don't pass `billing`, the engine assumes monthly.

## When estimates look too low or too high

**Too low**: You may not be calling `track('checkout_abandoned', ...)`. Without explicit abandonment events, the engine has to infer them from sessions that had `plan_intent` but no `subscription_started`. That inference is less precise than an explicit event.

**Too high**: Check whether your `subscription_started` events are firing in test or CI sessions. If they are, the engine's baseline conversion rate will be inflated and the gap will appear larger than it is. Pass `environment: "development"` to your SDK init in non-production environments to keep test data out of production scores.

## See also

The same conversion events power the [Conversion trigger](./conversion-trigger): the confused-vs-calm analysis that shows, site-wide, how much less confused sessions convert than calm ones. Revenue impact dollarizes individual issues; the conversion trigger proves confusion is costing conversions and pinpoints where. Wire the event once and both get sharper.
