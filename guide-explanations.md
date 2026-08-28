# How Guide explanations work

Guide generates plain-language explanations for UI elements on the fly. When a user hovers over something in Guide mode, the SDK captures the element's identity, sends it to the explanation engine, and renders the response in a card near the cursor. Most hovers hit a cache and render instantly; only a cold element waits on the model.

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

Explanations persist in the browser's IndexedDB across sessions. When a returning user hovers over an element they saw last week, the explanation loads locally with no network request. Entries expire after 7 days.

### Layer 3: Edge cache (server-side, shared)

Explanations are cached server-side for 7 days, keyed by site, page, and element (the key is derived on the server from the sanitized selector, so a client can't poison another element's entry). When user A hovers over the "Submit" button and gets an explanation, user B hovering the same button on the same page hits this shared cache. No new AI call.

After a handful of users explore a page, most of its elements are cached and the model only fires for elements it hasn't seen before.

### Invalidation

Every layer works the same way: entries expire 7 days after they were written. There's no manual purge and no deploy-triggered invalidation; if you ship a change to an element, its old explanation can survive for up to a week until the TTL rolls it over.

The `data-fd-guide` attribute bypasses all caching. Its content is always used verbatim.

## Prefetching

A couple of seconds after page load, during browser idle time, the SDK collects up to 20 visible interactive elements (skipping anything under `data-fd-guide-ignore` and anything carrying its own `data-fd-guide` text) and requests an explanation for each one it doesn't already have locally. That pre-warms the cache so the first hover on a visible element is instant.

Prefetch requests go through the same endpoint as a live hover. On a warm page they're all server cache hits. On a page the server hasn't seen, they generate, which uses your Guide allowance; that's the tradeoff for instant first hovers. In feedback-only mode the SDK skips prefetching entirely, since no explanations are ever shown.

## Guide allowance

Guide uses a fast model for generation, and each explanation is a couple of sentences, so a single generation is cheap. The allowance exists so a crawler or a hostile client can't run up your bill.

The three-day trial includes a fixed Guide allowance for the lifetime of the trial. It does not reset when the month changes. Paid plans include a monthly Guide allowance that scales with the plan tier.

When the allowance is used, cached explanations still work, but new AI generations pause. During a trial, Billing offers **Start my plan now**. On a paid plan, generation resumes at the start of the next month or after a plan upgrade. There is no add-on capacity to buy.

### What uses the allowance

Only live AI generations use the allowance. Cache hits at any layer (in-memory, IndexedDB, server) do not, and `data-fd-guide` attribute explanations never touch the network at all. Prefetch uses the allowance only when it hits an element the server hasn't generated yet.

In practice, the cache absorbs most hovers after the first few sessions on each page. The allowance covers cold starts and the long tail of rarely visited elements.

## Monitoring usage

The Guide page in your dashboard shows what your users are asking about:

- Explanations shown, elements explained, and confusion reports received
- The elements users ask about most. High counts on one element are confusion evidence in their own right
- Confusion reports with triage controls (new, reviewed, dismissed)
- Pages where Guide was active and the underlying issue has since been fixed, so you can see the loop close

Allowance status and generation counts live on Settings → Usage.
