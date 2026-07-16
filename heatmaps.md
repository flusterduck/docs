# Heatmaps

Flusterduck heatmaps don't show where users click. They show where users struggle.

Traditional heatmaps paint every click the same color. A confident tap on "Add to Cart" and a frustrated rage click on a broken button both render as orange blobs. That's not useful. You already know people click your buttons. The question is which clicks represent confusion.

Flusterduck only renders friction signals: rage clicks, dead clicks, and disabled element attempts. Normal interactions don't appear. What you see is purely the spatial distribution of frustration on your page.

There are three heatmap views, each at a different zoom level. The element heatmap shows where clicks land on a single element. The page heatmap shows where friction concentrates across the viewport. The friction map ranks every element on a page by signal volume. Together they answer: where is the problem, how bad is it, and what else on this page is struggling too.

## Element heatmap

The element heatmap zooms into one CSS selector and plots exactly where frustration clicks landed within that element's bounding box. Every dot is a rage click, dead click, or disabled element attempt.

This matters because position within an element tells you things the signal type alone can't. If every rage click on a wide navigation bar clusters in the right 10%, users are probably aiming for something that isn't there, or a submenu that doesn't open. If dead clicks scatter evenly across a hero image, users think the whole image is clickable. If they cluster on text inside the image, users are trying to select or click a phrase they think is a link.

### How positions are captured

The SDK records click position as a 0-to-1 fraction relative to the element's bounding box. For rage clicks, the SDK computes the centroid of the click burst (the average x/y of all clicks in that burst) and normalizes it against the element's width and height. For dead clicks and disabled element attempts, it's the single click position.

The raw values emitted are `hx` and `hy`, integers from 0 to 1000 (where 500,500 is the center). The API normalizes these to `rx` and `ry` floats from 0 to 1 before returning them.

No pixel coordinates leave the browser. The SDK only stores the relative position within the element, so there's nothing to reconstruct about viewport size, screen resolution, or device type from this data.

### Reading the panel

The element heatmap panel shows up on the issue detail page when the issue has an `element_selector`. It renders a rectangular box representing the element, with colored dots at their relative positions:

| Dot color | Signal type |
|---|---|
| Red | Rage click |
| Orange | Dead click |
| Purple | Disabled element attempt |

Brighter clusters (dots stacked on top of each other) mean repeated frustration at the same spot. A crosshair overlay marks the center of the element for reference.

The legend at the bottom shows counts per signal type. If you see "14 rage click, 6 dead click", that's 20 total friction events on this one element, with most of them being repeated rapid clicking.

### What patterns to look for

**Cluster in one corner.** Users are aiming for something specific that either doesn't exist or doesn't respond. Common on navigation bars, tab groups, and cards with multiple clickable regions.

**Even spread across the whole element.** Users think the entire element is interactive. Common on images, banners, and cards that lack a clear CTA. The fix is usually adding a visible button or making the whole surface clickable.

**Cluster on text inside a non-interactive element.** Users are trying to click a link that isn't a link. Check whether the text is styled like a hyperlink (blue, underlined) without actually being one.

**Two separate clusters.** Two different user intentions hitting the same element. Maybe one group wants to expand it and another wants to select it. Check whether the element needs to be split into distinct interactive regions.

### API

```
GET /query/element-heatmap?site_id={id}&page=/checkout&selector=button.submit
```

Response:

```json
{
  "page": "/checkout",
  "selector": "button.submit",
  "count": 23,
  "points": [
    { "rx": 0.482, "ry": 0.511, "signal": "rage_click" },
    { "rx": 0.91, "ry": 0.33, "signal": "dead_click" }
  ]
}
```

Each point has `rx` and `ry` (0 to 1 fractions of the element's width and height) and a `signal` string. The API caps the response at 300 points per request, pulling the most recent signals first.

The `selector` parameter must be URL-encoded. CSS selectors with brackets, spaces, or special characters need proper encoding:

```bash
curl "https://rhwhnkrqjlzyzcdhvyky.supabase.co/functions/v1/query/element-heatmap?site_id=YOUR_SITE_ID&page=%2Fcheckout&selector=button.submit" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

## Page heatmap

The page heatmap zooms out to the full viewport. Instead of showing clicks on one element, it plots every friction event on the page as a viewport-relative dot. You see where on the screen users are struggling, regardless of which element they're hitting.

This is the view that answers "which region of my page is generating the most confusion?" It's particularly useful for pages with complex layouts where multiple elements compete for attention.

### How viewport positions work

The SDK captures viewport-relative coordinates alongside every click-based signal. When a rage click fires, the SDK records `vx` and `vy` as integers from 0 to 1000, where (0,0) is the top-left corner of the visible viewport and (1000,1000) is the bottom-right. The API normalizes these to 0-to-1 floats before returning.

These coordinates represent where the click happened relative to the browser viewport at that moment, not relative to the document. A click at the very bottom of a long scrolled page still maps to the viewport position the user saw. This means the heatmap reflects the user's visual experience, not the document's scroll position.

### Reading the panel

The page heatmap panel appears on every issue detail page. It renders a viewport-shaped box with a subtle grid overlay, dots colored by signal type (same red/orange/purple scheme as the element heatmap), and crosshairs marking the center.

Bright clusters mean repeated friction in the same viewport region. Scattered dots with no clustering mean friction is distributed across the page, which usually points to a general layout or navigation problem rather than a single broken element.

### What patterns to look for

**Cluster in the top-right corner.** Users are hitting the area where close buttons, account menus, or navigation items live. If you see rage clicks here, something in that region isn't responding.

**Band of friction across the middle.** Probably a sticky element (floating CTA, cookie banner, toolbar) that's intercepting clicks meant for content underneath. Very common on mobile.

**Dense cluster surrounded by empty space.** One element is generating all the friction. Cross-reference with the friction map to identify which selector.

**Scattered dots everywhere, no clusters.** The page itself is the problem. Layout is confusing, nothing is clear, users are clicking around trying to find something. Check your information hierarchy.

### API

```
GET /query/page-heatmap?site_id={id}&page=/checkout
```

Response:

```json
{
  "page": "/checkout",
  "count": 47,
  "points": [
    { "vx": 0.512, "vy": 0.234, "signal": "rage_click", "el": "button.submit" },
    { "vx": 0.891, "vy": 0.044, "signal": "dead_click", "el": "div.header-logo" }
  ]
}
```

Each point has `vx` and `vy` (viewport fractions), a `signal` type, and an optional `el` field with the CSS selector of the element that was clicked. The API returns up to 500 points.

## Friction map

The friction map isn't a spatial visualization. It's a ranked list of every element on a page that's generating friction signals, ordered by signal count. A leaderboard of broken and confusing elements.

While the element and page heatmaps answer "where," the friction map answers "what" and "how much." It shows you every element the scoring engine has flagged on a given page, with bar charts sized proportionally to signal volume.

### Reading the panel

The friction map panel appears on issue detail pages. Each row in the list shows:

- An icon for the dominant signal type on that element
- The CSS selector (e.g., `button.cta-primary`, `div.pricing-toggle`)
- A horizontal bar colored by score severity (green through red), scaled relative to the highest-signal element on the page
- The dominant signal type and number of affected users
- The raw signal count

The element that belongs to the current issue is highlighted with a "This issue" badge so you can see it in context. If the current issue's element is third on the list, the two elements above it are generating even more friction and might deserve their own issues.

The panel shows up to 12 elements. The API returns up to 100.

### What patterns to look for

**One element dominates.** Most of the friction is concentrated on a single element. Fix that one thing and the page score will drop.

**Top 4-5 elements have similar counts.** Friction is distributed. No single fix will move the score much. This usually means a structural or layout problem rather than a broken control.

**The current issue's element is far down the list.** Other elements on the page are worse. Consider whether the higher-ranked elements should be prioritized first.

**Multiple elements share the same dominant signal type.** If every top element shows dead clicks, users are clicking on things that look interactive but aren't. That's a visual design issue, not a bug.

### API

The friction map uses the elements endpoint:

```
GET /query/elements?site_id={id}&page=/checkout
```

Response:

```json
{
  "elements": [
    {
      "selector": "button.submit",
      "page": "/checkout",
      "score": 42,
      "dominant_signal": "rage_click",
      "signal_count": 31,
      "affected_users": 18
    },
    {
      "selector": "div.promo-banner",
      "page": "/checkout",
      "score": 28,
      "dominant_signal": "dead_click",
      "signal_count": 14,
      "affected_users": 11
    }
  ]
}
```

Elements are sorted by score descending. The `dominant_signal` is whichever signal type appeared most frequently on that element. `affected_users` counts distinct sessions, not total clicks.

## How these differ from traditional heatmaps

Traditional heatmap tools (Hotjar, Clarity, FullStory) record every mouse movement, every click, every scroll event. They show aggregate behavior. That's useful for understanding user flows, but it drowns signal in noise. A button with 10,000 healthy clicks and 15 rage clicks looks like a hot spot in a traditional heatmap. In Flusterduck it shows 15 dots.

| | Traditional heatmap | Flusterduck heatmap |
|---|---|---|
| Data source | All clicks and mouse movement | Only friction signals (rage, dead, disabled) |
| Privacy | Often requires session replay | No replay, no PII, no DOM recording |
| Signal-to-noise | Low. Every click is equal. | High. Only confused users appear. |
| Granularity | Page level only | Element level, viewport level, and per-element ranked list |
| Purpose | "Where do users click?" | "Where do users struggle?" |
| Volume needed | Hundreds of sessions for a readable heatmap | A handful of friction events is enough |

The trade-off is coverage. Flusterduck heatmaps won't show you general click patterns or scroll depth. That's on purpose. If you need "where do people click," use a traditional tool. If you need "where are people confused," this is it.

## Combining heatmap views

The three views work best together. Start with the friction map to see which elements are struggling. Pick the worst one. Open its element heatmap to see where on the element users are clicking. Then zoom out to the page heatmap to see if the problem is isolated or part of a broader spatial pattern.

A typical investigation flow:

1. Friction map shows `div.pricing-toggle` at the top with 40 rage clicks
2. Element heatmap shows all clicks clustered on the right side of the toggle, near the "Annual" label
3. Page heatmap shows the same cluster plus a secondary cluster near the plan cards below

That tells you users are rage-clicking the annual/monthly toggle (it's probably not responding fast enough or not giving visual feedback) and then rage-clicking the plan cards below (probably because they're still showing monthly prices after the user thought they switched to annual).

One view gives you the what. Two views give you the where. All three give you the why.
