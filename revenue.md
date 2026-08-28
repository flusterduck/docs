# Revenue Impact

Confusion scores tell you a page is broken. Revenue impact tells you what it's costing. Flusterduck gives you two separate ways to see that, and they answer different questions.

**Per-issue estimates** turn "47 users are rage-clicking the complete purchase button" into "and it's costing about $2,400-$3,800/mo." Every issue carries one, driven by published conversion-impact research for that signal type, sharpened by your own numbers if you provide them.

**Revenue opportunities** look for a more direct kind of leak: sessions where a visitor showed intent to pay one amount and ended up paying less, or nothing. This is the `/v1/revenue` endpoint, and it's a session-level accounting, not a friction-pattern estimate.

Both read from the same `track()` events. Wire them once.

## What to track

```ts
import { track } from 'flusterduck'

// User clicks a pricing CTA, selects a plan tier, or adds to cart.
// Before the payment form.
track('plan_intent', {
  plan_id: 'scale',
  amount_cents: 9900,
  billing: 'monthly',
})

// Purchase or subscription completed. In your post-payment success
// handler or your payment provider's webhook.
track('subscription_started', {
  plan_id: 'scale',
  amount_cents: 9900,
  billing: 'monthly',
})
```

`plan_intent` is the signal that a visitor was headed toward a specific dollar amount. `subscription_started` (or `purchase_completed` for a one-time sale, or `checkout_completed` for an order) is the signal that they landed, and at what amount. Fire the realized event even when the amount matches what was intended: revenue opportunities only exist as a *comparison*, so a realized-only session or an intent-only session doesn't produce one on its own.

If you have both a hover-level and click-level intent, `plan_hover`, `plan_selected`, and `cart_added` are also recognized as intent events, so you don't have to collapse everything into `plan_intent`.

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

## Per-issue estimates

Every open issue carries a `revenue_impact` object, visible via `GET /v1/issues` and `GET /v1/issues/:id`. It works even if you never call `track()`, because it starts from a published research benchmark rather than your own conversion data.

For the issue's dominant signal type, the engine looks up a conversion-drop range (a `rage_click` on a critical element is benchmarked at 18-28%, `form_abandonment` at 25-40%, and so on down to ambient signals in the 4-10% range), then applies that range to the percentage of sessions on the page the issue actually affected:

```json
{
  "id": "iss_xxxxxxxxxxxx",
  "title": "Dead clicks on complete purchase button",
  "page": "/checkout",
  "signal_count": 47,
  "revenue_impact": {
    "low_cents": 0,
    "high_cents": 0,
    "affected_pct": 14.2,
    "method": "benchmark",
    "label": "Affects 14% of sessions. Industry data suggests 12-20% conversion impact."
  }
}
```

`method: "benchmark"` with `low_cents`/`high_cents` at zero means no dollar figure yet: you haven't told Flusterduck what your revenue actually is. Set one of these under Settings > Revenue (or `PATCH /v1/manage/sdk-config` with a `revenue_config` object) and the same issue gets a real number:

| Field | Type | What it does |
|---|---|---|
| `monthly_revenue_cents` | integer | Total monthly revenue for the site. The most accurate input: the benchmark rate is applied directly to your real revenue times the affected share |
| `avg_order_value_cents` | integer | Per-session value, if you'd rather not share total revenue. Combined with `baseline_conversion_pct` |
| `baseline_conversion_pct` | number, 0-100 | Your normal conversion rate. Without it, every affected session is priced as though it would have converted, which overstates impact by roughly the inverse of your real conversion rate |
| `cac_cents` | integer | Cost to acquire a customer. Prices friction earlier in the funnel, before a cart exists, as a floor on what a lost visitor cost you |
| `conversion_goal` | string | Shared with the [conversion trigger](./conversion-trigger); not used by revenue estimates |

With `monthly_revenue_cents` set, the same issue reads `"method": "configured"` and `low_cents`/`high_cents` carry real numbers.

Use the range as a prioritization signal, not a forecast. It assumes the benchmarked signal type is actually suppressing conversion on your specific page, which is usually a safe assumption but not a guaranteed one.

## Revenue opportunities

`GET /v1/revenue` looks for something more concrete than a benchmark: sessions where a visitor's intent and their outcome don't match.

```bash
curl "https://api.flusterduck.com/v1/revenue" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

```json
{
  "site_id": "site_xxxxxxxxxxxx",
  "currency": "usd",
  "total_monthly_recurring_loss_cents": 8400,
  "total_one_time_loss_cents": 0,
  "sessions_with_revenue_leak": 3,
  "opportunities": [
    {
      "session_id": "ses_xxxxxxxxxxxx",
      "intended_plan_id": "scale",
      "realized_plan_id": "grow",
      "intended_amount_cents": 9900,
      "realized_amount_cents": 3900,
      "monthly_recurring_loss_cents": 6000,
      "one_time_loss_cents": 0,
      "revenue_model": "subscription",
      "heuristic": "value_collapse_downgrade",
      "first_seen_at": "2026-06-10T09:02:00Z",
      "realized_at": "2026-06-10T09:14:00Z"
    }
  ],
  "reason": null,
  "message": "Revenue impact is estimated from explicit plan intent and checkout telemetry."
}
```

Each entry in `opportunities` is one session that intended a plan or amount and realized a lower one, or bought nothing at all. Subscription amounts are normalized to a common monthly cadence before comparing, so an annual intent against a monthly purchase is compared correctly rather than as raw totals. `heuristic` tells you which pattern the session matched: `value_collapse_downgrade` (settled for less without much visible struggle) or `layout_exhaustion_settling` (three or more friction signals before settling, meaning the friction plausibly caused the downgrade rather than an ordinary price-sensitive choice).

If `opportunities` comes back empty, `reason` is `"insufficient_data"`: you either haven't fired both an intent and a realized event in the same session, or every session that has both realized the full intended amount (which is a good sign, not a data problem).

## Revenue via MCP

```
What's the revenue impact of the current open issues?
Which issue is costing us the most?
Rank the open issues by revenue at risk.
Are there sessions where someone intended a bigger plan and downgraded?
```

The `get_revenue_impact` tool returns the same opportunities data as `GET /v1/revenue`, formatted for your AI assistant to reason about.

## Annual vs monthly

Both features use `billing` to normalize to a common monthly cadence. Pass `billing: "annual"` for annual subscribers and `billing: "monthly"` for monthly ones. If you don't pass `billing`, the engine assumes monthly.

## When numbers look off

**Per-issue estimate stuck at zero with `method: "benchmark"`**: you haven't set `monthly_revenue_cents` or `avg_order_value_cents` under Settings > Revenue. The affected-percentage and the benchmark range are still shown; only the dollar figure is missing.

**No revenue opportunities showing up**: check that you're firing both a `plan_intent`-family event and a realized event (`checkout_completed`, `purchase_completed`, or `subscription_started`) in the *same* session, even when the amounts match. A session with only one side of the pair never becomes an opportunity.

**Per-issue estimate looks too high**: check whether `subscription_started` or other realized events are firing from test or CI sessions. If they are, the estimate can be built on inflated volume. Pass `environment: "development"` to your SDK init in non-production environments to keep test data out of production numbers.

## See also

The same `track()` events power the [conversion trigger](./conversion-trigger): the confused-vs-calm analysis that shows, site-wide, how much less confused sessions convert than calm ones. Revenue impact prices individual issues and finds session-level leaks; the conversion trigger proves confusion is costing conversions in aggregate and pinpoints where. Wire the events once and all three get sharper.
