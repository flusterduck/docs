# How Guide explanations work

Guide generates plain-language explanations for UI elements on the fly. When a user hovers over something in Guide mode, the SDK captures the element's identity, sends it to the explanation engine, and renders the response in a card near the cursor. Most of this happens in under 200ms because of aggressive caching.

## What the engine sees

Guide doesn't screenshot your page or read its full DOM. It sends a narrow context packet per element:

- **Selector and label.** The CSS selector Flusterduck already computes for signal tracking, plus the element's accessible name (from `aria-label`, `aria-labelledby`, visible text, or `title`).
- **Role and state.** The ARIA role (explicit or implicit from the tag), plus whether it's disabled, expanded, checked, or required.
- **Parent context.** The role and label of the nearest named ancestor. A "Submit" button inside a `<form aria-label="Payment details">` is more meaningful than a "Submit" button inside a generic `<div>`.
- **Friction data.** If Flusterduck has signals on this element (rage clicks, dead clicks, disabled taps), the engine knows what specifically confused other users. This is the piece that makes Guide precise instead of generic.

No form values. No user-typed text. No page content beyond what's needed to identify the control and its purpose.

## Friction-aware explanations

A generic screen reader could tell you a button says "Deploy." Guide tells you what Flusterduck's data tells it:

> "Pushes your current changes to production. 43 users clicked this expecting a preview, but it deploys immediately. There's no confirmation dialog. If you want to preview first, use the 'Staging' button to the left."

That second sentence comes from the dead-click and rage-click signals Flusterduck has already clustered on this element. The explanation engine weaves behavioral evidence into the response so users get the warning before they make the same mistake.

Elements with no friction data still get explanations. They're just simpler: what the element does and what to expect when you interact with it.

## The cache

Most hovers don't trigger an AI call. Three cache layers sit between the hover event and the model.

### Layer 1: In-memory (client-side)

A map in the SDK's JavaScript. Instant. Holds the last 500 explanations the user has seen in this session. If the user hovers the same button twice, the second hover is free. Cleared on full page navigation, preserved across SPA route changes.

### Layer 2: Local storage (client-side, persistent)

Explanations persist in the browser's IndexedDB across sessions. When a returning user hovers over an element they saw last week, the explanation loads from local storage with no network request. Entries expire after 7 days or when a deploy invalidates them.

### Layer 3: Edge cache (server-side, shared)

Explanations are cached at the edge, keyed by site + page + element fingerprint. When user A hovers over the "Submit" button and gets an explanation, user B hovering the same button on the same page hits the edge cache. No new AI call.

After a handful of users explore a page, every element on it is cached. Expected edge hit rate after warm-up: 90-95%. The AI model only fires for new elements it hasn't seen before or freshly deployed pages.

### Invalidation

Caches invalidate on two triggers:

- **Deploy.** If you use Flusterduck's deploy correlation, explanations for the affected site are cleared automatically. The first user after a deploy re-primes the cache.
- **TTL.** Explanations expire after 7 days regardless of deploys. This catches sites that don't use deploy correlation.

The `data-fd-guide` attribute bypasses all caching. Its content is always used verbatim.

## Prefetching

On page load, the SDK scans the visible viewport for interactive elements, batches their fingerprints, and requests cached explanations in a single call. This pre-warms the in-memory cache so the first hover on any visible element is instant, not just the second.

Prefetching only requests elements that already have cached explanations on the edge. It doesn't trigger AI generation. The cost is one lightweight network request per page load.

## Cost and budgeting

Guide uses a fast, cost-effective model for generation. Each explanation is roughly 300 input tokens (element context + friction data) and 100-150 output tokens (the explanation itself).

Every Flusterduck plan includes a monthly Guide allocation. The allocation scales with your plan tier because higher tiers serve more sessions, which means more hovers, which means more AI calls. The included amount is designed to cover normal usage with room to spare.

When the allocation is spent, Guide degrades gracefully: cached explanations still work (they're free), but new AI generations pause until the next billing cycle or until you add overage capacity. The floating button shows a subtle indicator when the budget is low.

### What counts against the budget

Only live AI generations count. Cache hits at any layer (in-memory, local storage, edge) are free. `data-fd-guide` attribute explanations are free. Prefetch requests are free (they only return already-cached content).

In practice, the cache handles 90%+ of hovers after the first few sessions on each page. The budget covers the cold starts and the long tail of rarely-visited elements.

## Monitoring usage

The dashboard shows Guide usage alongside your other Flusterduck metrics:

- Total hovers this billing period
- Cache hit rate (broken down by layer)
- AI generations used vs. allocation remaining
- Top elements by hover count (which elements users ask about most)
- Friction reduction on Guide-explained elements vs. unexplained ones

The last metric is the one that matters. It closes the loop: Flusterduck detected friction, Guide explained it, and the verification engine confirms whether confusion dropped.
