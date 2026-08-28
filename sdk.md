# SDK Reference

One required config option: `key`. The SDK handles everything else automatically.

## Installation

```bash
pnpm add flusterduck
```

Framework wrappers install alongside the core SDK:

```bash
pnpm add flusterduck @flusterduck/next    # Next.js
pnpm add flusterduck @flusterduck/react   # React
pnpm add flusterduck @flusterduck/vue     # Vue
pnpm add flusterduck @flusterduck/svelte  # SvelteKit
pnpm add flusterduck @flusterduck/nuxt    # Nuxt
```

### Or the script tag

No package install needed:

```html
<script src="https://flusterduck.com/d.js" data-key="fd_pub_xxxxxxxxxxxx" data-dnt="false" async></script>
```

Always use `async`, never `defer`. It's the only way a slow or unreachable CDN can never delay the host page's `DOMContentLoaded`. The bootstrap reads its config from `document.currentScript`, falling back to `document.querySelector('script[data-key^="fd_pub_"]')` when the tag is injected dynamically (tag managers, `document.createElement('script')`), so `<script async>` injection works either way.

## What you get automatically

Once `init()` runs, the SDK starts detecting all 132 behavioral signal types without any additional code. You don't call `signal()` for rage clicks, dead clicks, form abandonment, scroll confusion, or mobile tap misses. Those just work.

Signal classification runs locally in the browser. The SDK sends classified signals and safe metadata to the ingest API; scoring, alerting, and issue correlation happen after ingest.

You only call `signal()` manually for signals tied to application logic the SDK can't observe (a specific element that isn't interactive by default, a custom form flow, etc.), or to add metadata to auto-detected signals.

## init()

Call once at app startup. Calling it multiple times after initialization is a no-op.

```ts
import { init } from 'flusterduck'

init({ key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY! })
```

### Config options

| Option | Type | Default | Notes |
|---|---|---|---|
| `key` | `string` | **required** | Your `fd_pub_` publishable key. Passing a `fd_sec_` key here logs an error and refuses to initialize. |
| `environment` | `string` | `"production"` | Attached to every signal. Use `"staging"` or `"development"` to keep non-prod data out of your production scores. |
| `sampleRate` | `number` | `1` | Fraction of sessions to track. Most teams don't need this until they're at 50k+ sessions/month. At that point, `0.5` cuts your ingestion in half with negligible impact on signal quality. |
| `debug` | `boolean` | `false` | Logs every signal to the console as it's emitted. Don't ship with this on. It's noisy. |
| `segment` | `Record<string, string>` | `undefined` | Static tags attached to every signal in the session, such as release version or experiment cohort. |

Most teams ship with only `key`, `environment`, and a small `segment` object. That's enough.

### Other config options

Everything above covers what most teams touch. The rest exist for specific situations:

| Option | Type | Default | Notes |
|---|---|---|---|
| `respectDoNotTrack` | `boolean` | `true` | Honor `navigator.doNotTrack` and Global Privacy Control. See [Consent and GDPR](/consent). |
| `cookieless` | `boolean` | `false` | Use a memory-only session ID instead of a cookie. A new session starts on every page load. |
| `sessionTimeout` | `number` | `1800000` | Inactivity window in milliseconds before the next interaction starts a fresh session (30 minutes, matching GA4/Hotjar/Amplitude). |
| `transport` | `'auto' \| 'none'` | `'auto'` | `'none'` runs detectors and fires `onSignal` but never queues or sends anything. For local demos and tests. |
| `signals` | `Partial<Record<string, { enabled?: boolean }>>` | `undefined` | Turn individual detectors off by camelCase name, e.g. `{ rageClick: { enabled: false } }`. Every detector is on by default. |
| `trackForms` | `boolean` | `true` | Turn off all form-related detectors (abandonment, hesitation, validation loops) at once. |
| `trackMouse` | `boolean` | `true` | Turn off cursor-based detectors (thrash cursor). |
| `breadcrumbs` | `boolean` | `true` | The path a session took (clicks, form entry/exit, scroll milestones), same privacy envelope as signals. |
| `ignoreElements` | `string[]` | `undefined` | CSS selectors detectors should never fire on. |
| `ignorePages` | `string[]` | `undefined` | Path patterns (supports a leading or trailing `*`) where nothing is tracked. |
| `endpoint` | `string` | Flusterduck's ingest URL | Override for self-hosted proxies. Must be `https:` (or `http://localhost` for local dev). |
| `onSignal` / `emitSignalEvents` | see [Reacting to signals](/react-to-signals) | | React to friction the instant it's detected, before it's sent. |

### Example

```ts
init({
  key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!,
  environment: process.env.NODE_ENV === 'production' ? 'production' : 'staging',
  segment: {
    app_version: process.env.NEXT_PUBLIC_APP_VERSION ?? 'unknown',
  },
})
```

## signal()

Emit a behavioral signal manually. Use this when you need to tie a signal to application state the SDK can't observe automatically, or to attach metadata.

```ts
import { signal } from 'flusterduck'

signal(type, { element?, metadata?, weight? })
```

The second argument is one object with three optional fields: `element` (a CSS selector, up to 256 characters), `metadata` (up to 2048 bytes once serialized), and `weight` (0-100, defaults to 15). Whatever you pass under `metadata` is what lands in the signal's payload; anything at the top level of the second argument outside those three keys is ignored.

```ts
// Attach metadata to a signal the SDK might also auto-detect
signal('dead_click', {
  element: '[data-plan="enterprise"]',
  metadata: { label: 'Enterprise CTA', page_section: 'pricing-table' },
})

// Signal for a custom multi-step flow, with a custom severity weight
signal('form_abandonment', {
  weight: 30,
  metadata: { form: 'onboarding', step: 3, last_field: 'company_size' },
})
```

Never put PII, form values, or anything user-typed into metadata. Safe fields: element selectors, labels, page section names, plan IDs, product IDs.

`type` should be one of the 132 built-in signal types listed in [Signals](/signals). A name outside that list is accepted and transmitted, but it won't be weighted, scored, or clustered into an issue, there's no free-form custom signal type today. For a pattern that isn't one of the 132, a Custom Monitor is the supported path.

## track()

Record business events. The scoring engine uses these to estimate revenue impact from friction.

```ts
import { track } from 'flusterduck'

track(event, properties?)
```

```ts
// User expressed intent to purchase
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

// Cart or checkout abandoned
track('checkout_abandoned', {
  plan_id: 'grow',
  amount_cents: 3900,
})
```

Safe properties: `plan_id`, `amount_cents`, `billing`, `currency`, `quantity`, `product_id`. Never pass names, emails, addresses, or any text the user typed.

## identify()

Tag the current session with user or account properties. Values must be safe strings, numbers, or booleans. Use opaque IDs only: a database primary key or UUID is fine, but not an email or name.

```ts
import { identify } from 'flusterduck'

// After login
identify({
  user_id: 'usr_8f3a2c91',
  plan: 'scale',
  account_age_days: '42',
})
```

Do not pass `user@example.com` or `"Alice Johnson"` as any property. Flusterduck doesn't know which of your values are personally identifying, so that's on you. Full reference, including the string shorthand and the caps on keys, values, and PII-shaped fields: [Identifying sessions](/identify).

## setConsent()

Update consent state after initialization. Called when a user accepts or rejects your cookie/tracking prompt.

```ts
import { setConsent } from 'flusterduck'

setConsent(true)   // start tracking
setConsent(false)  // pause tracking, flush buffer
```

For strict pre-consent gating, wait to call `init()` until the user accepts, or use a framework wrapper with `enabled: false`. Calling `setConsent(false)` on an active session flushes the buffer and stops collection immediately.

## optOut()

Opt the current user out for the session. Semantically clearer than `setConsent(false)` when the user has explicitly requested no tracking, versus consent not yet collected.

```ts
import { optOut } from 'flusterduck'

optOut()
```

## destroy()

Shut down the SDK, remove all listeners, clear the session buffer. Use this if you need to completely tear down Flusterduck from a running app instance, for example in a test environment or an embedded widget that gets unmounted.

```ts
import { destroy } from 'flusterduck'

destroy()
```

Calling `init()` after `destroy()` restarts detection, but it's the same session: `destroy()` doesn't clear the session cookie, only the SDK's in-memory state. If you need a fresh session (a logout, a shared-device handoff), call `setConsent(false)` or `optOut()` instead, or clear the `_fd_s` cookie yourself before calling `init()` again.

## TypeScript

The package ships full type definitions. There's an exported `SignalType` union for compile-time checking, but it's a legacy subset, not the full 132: it covers the core detectors from earlier releases and won't autocomplete names added since (payment, auth, and pricing signals among them). For the current, complete list, check [Signals](/signals) rather than relying on the type.

```ts
import { signal } from 'flusterduck'
import type { SignalType } from 'flusterduck'

const type: SignalType = 'rage_click'
signal(type, { element: '#submit-btn' })
```

## Common mistakes

**Passing a secret key.** The SDK checks the key prefix on `init()`. A `fd_sec_` key logs a console error and refuses to initialize. Any other prefix, including `fd_mcp_`, also refuses to initialize but fails silently, no console output at all, so if `init()` seems to do nothing, check that `key` actually starts with `fd_pub_` before looking anywhere else.

**Calling `init()` in a server component.** The SDK is browser-only. Calling `init()` in a Next.js Server Component, an edge function, or any non-browser context will throw. Gate it with `typeof window !== 'undefined'` if needed, or use the `@flusterduck/next` wrapper which handles this automatically.

**Calling `identify()` before initialization.** `identify()` is a no-op until `init()` has completed. Call it as early as possible after login and after the SDK is initialized.
