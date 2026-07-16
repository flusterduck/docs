# flusterduck

The browser SDK. It detects friction signals in the browser and sends them to Flusterduck. This is the core package every integration depends on.

## Install

```bash
npm install flusterduck
```

Prefer no build step? Skip the package and drop in the script tag instead:

```html
<script src="https://flusterduck.com/d.js" data-key="fd_pub_xxxxxxxxxxxx" data-dnt="false" async></script>
```

Use `async` (not `defer`) so a slow or unreachable CDN can never delay the host page's `DOMContentLoaded`. The bootstrap reads its config from `document.currentScript`, with a fallback to `document.querySelector('script[data-key^="fd_pub_"]')`, so a dynamically injected `<script async>` tag (tag managers, `document.createElement('script')`) works too.

## Usage

```ts
import { init, signal, track, identify, onSignal } from 'flusterduck'

init({
  key: 'fd_pub_xxxxxxxxxxxx',
})

// Identify the current session with safe, non-PII properties.
identify({ plan: 'scale', cohort: 'beta' })

// Track a conversion event for revenue attribution.
track('checkout_completed', { value: 49 })

// React the instant a friction signal fires.
onSignal((s) => {
  console.log('friction detected:', s.type)
})

// Emit a custom signal when you detect friction yourself.
signal('search_no_results', { query: 'pricing' })
```

Use `setConsent(true)` to gate collection behind consent, or `optOut()` to stop collection for the current session.

## Links

Published on npm as `flusterduck`. Install pulls the latest published version.
