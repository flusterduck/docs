# Setting up Guide

Guide ships as part of the core SDK. No separate package, no browser extension, no extra script tag. Enable it in your `init()` call and it's live.

## Enable Guide

```ts
import { init } from 'flusterduck'

init({
  key: 'fd_pub_...',
  guide: {
    enabled: true,
  },
})
```

That's it. The SDK adds a floating help button to the page and registers the keyboard shortcut. Users activate Guide mode, hover over elements, and get explanations.

## Modes

Guide has three modes, set with `mode`:

```ts
guide: {
  enabled: true,
  mode: 'both', // 'explain' (default) | 'feedback' | 'both'
}
```

- **`explain`** (default): hover any element in Guide mode, get a plain-English explanation.
- **`feedback`**: the help button opens a "Tell us what's confusing" box instead. Users type what lost them and submit; nothing else. No AI calls are made in this mode, so it also works on plans or pages where you don't want generated explanations.
- **`both`**: hover-to-explain as normal, plus a "Still confused? Tell us" link on every explanation card that opens the feedback box.

Feedback submissions appear in your dashboard under **Guide → Confusion reports**, verbatim, with the page (and the element, when the report came from an explanation card). Comments are capped at 500 characters and sanitized server-side; submissions are rate limited per visitor.

Customize the panel title:

```ts
guide: {
  enabled: true,
  mode: 'feedback',
  feedbackPrompt: 'What were you looking for?',
}
```

Open the feedback box from your own UI (a "Report a problem" menu item, for example):

```ts
import { guide } from 'flusterduck'

guide.feedback()
```

## Activation mechanisms

Guide is always user-triggered. It never pops up on its own. You decide how users activate it: a built-in floating button, a keyboard shortcut, your own custom trigger, or any combination.

### Floating help button

A small fixed-position button in the bottom-right corner of the viewport. One tap toggles Guide mode on and off.

```ts
guide: {
  enabled: true,
  fab: {
    position: 'bottom-right',  // 'bottom-right' | 'bottom-left'
    offset: { x: 24, y: 24 },  // pixels from corner
    label: 'Help',              // screen reader label and tooltip
  },
}
```

If you already have a chat widget or cookie banner in the bottom-right, move the button to `bottom-left` or adjust the offset.

The button renders inside a closed shadow root. It won't inherit your fonts, colors, or resets, and its CSS won't leak into your page.

### No floating button

Set `fab: false` to hide the built-in button entirely. Users activate Guide through the keyboard shortcut or your own trigger.

```ts
guide: {
  enabled: true,
  fab: false,
}
```

### Your own trigger

Wire Guide to any element in your app. The `guide` export gives you programmatic control:

```ts
import { guide } from 'flusterduck'

// In your help menu
document.querySelector('#my-help-btn').addEventListener('click', () => {
  guide.toggle()
})

// Or conditionally
if (userIsNew) guide.activate()
```

`guide.activate()`, `guide.deactivate()`, `guide.toggle()`, and `guide.isActive()` work regardless of whether the FAB is visible. This is the recommended path if you want Guide to feel native to your app rather than an overlay.

### Keyboard shortcut

Press `?` (Shift + /) to toggle Guide mode. This matches the convention used by GitHub, Gmail, and Slack for help overlays.

The shortcut only fires when no text input is focused. If the cursor is in an `<input>`, `<textarea>`, or `contenteditable` element, the keystroke passes through normally.

```ts
guide: {
  enabled: true,
  shortcut: '?',         // any single character, or false to disable
}
```

Set `shortcut: false` if `?` conflicts with your app's own shortcuts. You can also trigger Guide mode programmatically:

```ts
import { guide } from 'flusterduck'

guide.activate()    // enter Guide mode
guide.deactivate()  // exit Guide mode
guide.toggle()      // flip
```

Wire these to your own button, keyboard shortcut, or menu item if you want full control over activation.

### Mobile

On touch devices, the floating button still appears. When Guide mode is active, a single tap on any element shows the explanation card. Tap elsewhere or tap the button again to dismiss. No hover required.

Long-press also works as a trigger: the user holds on an element for 400ms and the explanation appears. This avoids interference with normal tap interactions.

## Positioning the explanation card

The explanation card appears near the cursor (or near the tapped element on mobile). It automatically flips to stay within the viewport. You don't need to configure positioning.

If you want to constrain where the card can appear:

```ts
guide: {
  enabled: true,
  card: {
    maxWidth: 320,       // pixels, default 320
    offset: 12,          // gap between cursor and card edge
  },
}
```

The card renders inside the same closed shadow root as the button. Your page styles don't affect it.

## Scoping to specific pages

By default, Guide is available on every page the SDK loads on. To restrict it:

```ts
guide: {
  enabled: true,
  pages: ['/checkout', '/settings', '/onboarding/*'],
}
```

Glob patterns work. `*` matches any single path segment. `**` matches any depth.

On pages not in the list, the button and shortcut are both hidden. The SDK still detects friction signals normally; only the Guide UI is scoped.

## Server-side settings

The snippet controls how the widget looks; the dashboard controls whether it answers. Under **Settings → Guide** you can:

- **Disable Guide entirely**: a real kill switch, enforced by the API. Requests are rejected server-side no matter what the client snippet asks for.
- **Restrict the mode**, e.g. feedback-only: explanation requests get a `403 guide_explain_disabled` even if a stale snippet still asks for them.
- **Allowlist pages**: same glob syntax as the client `pages` option, enforced at the API so the client scoping can't be bypassed.

Everything Guide collects lands in the dashboard under **Guide**: total explanations shown, the elements users ask about most (your strongest confusion evidence), confusion reports with triage controls, and a closed-loop view of pages where Guide was active and the underlying issue has since been fixed.

## Guide as an API

Guide's backend is a plain HTTPS API authenticated with your publishable key, the same contract whether you call it from the SDK, your own client, or a native app. The base endpoint is:

```
POST https://api.flusterduck.com/v1/guide
x-fd-key: fd_pub_...
Content-Type: application/json
```

### Explanation requests

```json
{
  "fingerprint": "a1b2c3d4e5f60718",
  "page": "/checkout",
  "context": {
    "selector": "button.submit",
    "label": "Place order",
    "role": "button",
    "tagName": "button",
    "isDisabled": false,
    "isExpanded": false,
    "isRequired": false,
    "parentLabel": "Payment form",
    "parentRole": "form"
  }
}
```

`fingerprint` is any stable hash of the element + page (the SDK uses a context hash); it keys the server-side cache. `context.selector` is required; everything else is optional but improves the explanation. Response envelope:

```json
{ "data": { "explanation": "Charges your card and places the order. You'll get a confirmation email within a few seconds." }, "error": null }
```

Explanations are cached server-side per fingerprint, sanitized (no HTML, no URLs, no email addresses), and rate limited at 200 requests/minute per site and 15/minute per visitor IP.

### Feedback submissions

```json
{
  "kind": "feedback",
  "page": "/checkout",
  "comment": "I typed my discount code but nothing happened",
  "selector": "button#apply-coupon",
  "label": "Apply coupon"
}
```

`page` and `comment` (3–500 characters) are required; `selector` and `label` are optional. Response: `{ "data": { "received": true }, "error": null }`. Rate limited at 60/minute per site and 5 per 5 minutes per visitor IP.

### Errors

| Status | Code | Meaning |
| --- | --- | --- |
| 403 | `guide_disabled` | Guide is switched off in Settings → Guide |
| 403 | `guide_explain_disabled` | The site is in feedback-only mode |
| 403 | `guide_feedback_disabled` | The site is in explain-only mode |
| 403 | `guide_page_not_allowed` | The page isn't in the server-side allowlist |
| 400 | `invalid_*` / `comment_too_short` | Validation failure; the message says which field |
| 429 | `rate_limited` | Back off and retry later |

## Bring your own API (BYOK)

By default, Guide calls the Flusterduck edge function for AI explanations, which uses your plan's included allocation. If you want to use your own Anthropic key, your own model, or a completely custom backend, point Guide at your endpoint:

```ts
guide: {
  enabled: true,
  apiEndpoint: 'https://your-server.com/api/guide-explain',
}
```

Your endpoint receives the exact request bodies documented above: explanation requests and, when feedback mode is on, feedback submissions (distinguished by `"kind": "feedback"`). For explanations, return JSON with an `explanation` field (a bare `{ "explanation": "..." }` or the `{ "data": { ... } }` envelope both work):

```json
{
  "explanation": "Charges your card and places the order. You'll get a confirmation email within a few seconds."
}
```

BYOK requests don't count against your Flusterduck Guide allocation. You're paying your own provider directly.

## Custom explanations

For critical elements where you know exactly what the explanation should say, skip the AI entirely:

```html
<button data-fd-guide="Creates a new project. Doesn't affect existing projects. You can rename or delete it later from Settings.">
  New Project
</button>
```

When a user hovers over an element with `data-fd-guide`, the SDK shows that text directly. No API call, no latency, no AI cost. The attribute content always wins over generated explanations.

Use this for high-traffic elements where you want guaranteed wording, or for controls that handle money, deletion, or permissions where precision matters.

## Disabling Guide for specific elements

```html
<div data-fd-guide-ignore>
  <!-- nothing inside this subtree will show Guide explanations -->
</div>
```

Useful for decorative regions, ads, or third-party embeds where explanations would be confusing or irrelevant.

## Content Security Policy

Guide runs entirely within the existing SDK script. No new CSP entries are needed for the UI or style injection (it uses `adoptedStyleSheets` inside a shadow root, which bypasses `style-src` restrictions).

The SDK already requires your CSP's `connect-src` to include the Flusterduck API domain for event ingestion. Guide uses the same endpoint for fetching explanations. If ingestion works, Guide works.

## Privacy

Guide follows the same privacy rules as the rest of the SDK. When extracting element context to generate an explanation, it reads:

- Tag name, role, and accessible label
- Element state (disabled, checked, expanded)
- Parent context (the nearest labeled ancestor)
- The CSS selector Flusterduck already computes for signal tracking

It never reads form values, text content the user typed, or any attribute you haven't explicitly set. The `data-fd-guide` attribute is opt-in content you control entirely.

Explanation requests are scoped to your site's publishable key. Flusterduck never sends element context from one customer's site to generate explanations for another.
