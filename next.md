# Next.js

For most apps: add `FlusterduckScript` to your root layout. Done. It's a Server Component that renders a non-blocking script tag with no client JS added to your bundle and no extra render cycle.

Reach for `useFlusterduck` when you need to gate on consent state, flip tracking on/off based on auth, or access methods like `setConsent` in a component.

Don't use both in the same app.

## Install

```bash
pnpm add flusterduck @flusterduck/next
```

## App Router

```tsx
// app/layout.tsx
import { FlusterduckScript } from '@flusterduck/next'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <FlusterduckScript apiKey={process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!} />
      </body>
    </html>
  )
}
```

```bash
# .env.local
NEXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

`FlusterduckScript` is a Server Component. Don't add `'use client'` to your layout for this. That defeats the purpose. The script executes in the browser after delivery.

## Pages Router

```tsx
// pages/_app.tsx
import type { AppProps } from 'next/app'
import { FlusterduckScript } from '@flusterduck/next'

export default function App({ Component, pageProps }: AppProps) {
  return (
    <>
      <Component {...pageProps} />
      <FlusterduckScript apiKey={process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!} />
    </>
  )
}
```

## FlusterduckScript

Renders a `next/script` tag with `strategy="afterInteractive"`. Doesn't block rendering or hydration. Configuration is carried as `data-*` attributes on the tag, no runtime bundle cost.

### Props

| Prop | Type | Default | Description |
|---|---|---|---|
| `apiKey` | `string` | - | Required. Your `fd_pub_` key. |
| `environment` | `string` | - | `"production"`, `"staging"`, `"development"` |
| `sampleRate` | `number` | `1.0` | `0.5` tracks half of sessions. |
| `cookieless` | `boolean` | `false` | Use memory-only session IDs instead of cookies. |
| `debug` | `boolean` | `false` | Logs every signal and flush to the console. |
| `segment` | `Record<string, string>` | - | Key/value pairs attached to every event. Use for deploy tagging, A/B variants, feature flags. |

`domMode`, `ignoreElements`, `batchInterval`, and other advanced options aren't available on `FlusterduckScript`. It's a script tag, not a programmatic init call. Use `useFlusterduck` if you need them.

Passing a secret key (`fd_sec_`) logs an error in development and renders nothing in production.

## useFlusterduck

A Client Component hook. Initializes the full SDK with all options and returns tracking methods. Use this instead of `FlusterduckScript` when initialization needs to be conditional.

```tsx
'use client'
import { useFlusterduck } from '@flusterduck/next'

export function Analytics() {
  useFlusterduck({ key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY! })
  return null
}
```

Drop this component in your root layout as a `'use client'` island.

### Options

Everything from the core `Config` type plus `enabled`:

| Option | Type | Default | Description |
|---|---|---|---|
| `key` | `string` | - | Required. Your `fd_pub_` key. |
| `environment` | `string` | - | Deployment environment. |
| `sampleRate` | `number` | `1.0` | Fraction of sessions to track. |
| `domMode` | `'off' \| 'metadata' \| 'snapshot'` | `'off'` | `'metadata'` captures element attributes. `'snapshot'` adds layout and computed styles. |
| `cookieless` | `boolean` | `false` | Memory-only session IDs instead of cookies. |
| `respectDoNotTrack` | `boolean` | `false` | Honor `navigator.doNotTrack`. |
| `ignoreElements` | `string[]` | `[]` | CSS selectors to suppress signals on. Useful for admin overlays or debug panels. |
| `ignorePages` | `string[]` | `[]` | Page paths to skip. Match is prefix-based. |
| `segment` | `Record<string, string>` | - | Static tags on every event. |
| `debug` | `boolean` | `false` | Verbose console logging. |
| `batchInterval` | `number` | - | Milliseconds between flushes. Default flushes on idle and before unload. |
| `compression` | `'auto' \| 'off'` | `'auto'` | `'auto'` uses compression when the browser supports it. |
| `elementImpressionSelectors` | `string[]` | - | CSS selectors to emit an impression signal when they enter the viewport. |
| `enabled` | `boolean` | `true` | `false` skips initialization entirely. Flip to `true` and it initializes then. |

### Return value

```ts
const { signal, track, identify, setConsent, optOut } = useFlusterduck({ key: '...' })
```

| Method | Description |
|---|---|
| `signal(name, data?)` | Emit a friction signal manually. |
| `track(name, metadata?)` | Track a business event. |
| `identify(segment)` | Tag the session with user properties. |
| `setConsent(consented)` | Pause or resume collection. |
| `optOut()` | Stop collection permanently for this session. |

## Tracking in Client Components

Initialize once at the root. After that, import from `flusterduck` directly. No hook needed:

```tsx
'use client'
import { track } from 'flusterduck'

export function PlanCard({ planId, price }: { planId: string; price: number }) {
  return (
    <button onClick={() => track('plan_intent', { plan_id: planId, price_cents: price })}>
      Get started
    </button>
  )
}
```

```tsx
'use client'
import { signal } from 'flusterduck'

export function OnboardingStep({ step }: { step: number }) {
  return (
    <div onMouseLeave={() => signal('task_abandonment', { metadata: { step } })}>
      {/* step content */}
    </div>
  )
}
```

Core SDK functions buffer until the SDK is ready. Calling them before initialization resolves is safe.

## Consent flow

The recommended pattern: initialize the SDK, call `setConsent(false)` before showing your banner, then flip it based on the user's choice. Collection is paused until they respond.

```tsx
// app/layout.tsx
import { FlusterduckScript } from '@flusterduck/next'
import { ConsentBanner } from '@/components/consent-banner'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <FlusterduckScript apiKey={process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!} />
        <ConsentBanner />
      </body>
    </html>
  )
}
```

```tsx
// components/consent-banner.tsx
'use client'
import { useEffect, useState } from 'react'
import { setConsent } from 'flusterduck'

export function ConsentBanner() {
  const [visible, setVisible] = useState(false)

  useEffect(() => {
    const stored = localStorage.getItem('fd_consent')
    if (stored === null) {
      setConsent(false)
      setVisible(true)
    } else {
      setConsent(stored === 'true')
    }
  }, [])

  if (!visible) return null

  const accept = () => {
    localStorage.setItem('fd_consent', 'true')
    setConsent(true)
    setVisible(false)
  }

  const decline = () => {
    localStorage.setItem('fd_consent', 'false')
    setConsent(false)
    setVisible(false)
  }

  return (
    <div>
      <p>We use behavioral analytics to improve this site.</p>
      <button onClick={accept}>Accept</button>
      <button onClick={decline}>Decline</button>
    </div>
  )
}
```

If you want full React-driven consent gating with no collection until consent is granted, use `useFlusterduck` with `enabled` instead:

```tsx
'use client'
import { useFlusterduck } from '@flusterduck/next'

export function Analytics({ consented }: { consented: boolean }) {
  useFlusterduck({
    key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!,
    enabled: consented,
  })
  return null
}
```

The difference matters for compliance. `setConsent(false)` initializes but pauses. Anonymous session data is captured and held until consent is granted. `enabled: false` skips initialization entirely. Nothing is recorded until it flips to `true`. If your legal requirement is zero data before consent, use `enabled: false`. If anonymous pre-consent analytics are acceptable, `setConsent(false)` is simpler to wire up.

## Identifying users

```tsx
'use client'
import { useEffect } from 'react'
import { identify } from 'flusterduck'

export function IdentifyUser({
  userId,
  orgId,
  plan,
}: {
  userId?: string
  orgId?: string
  plan?: string
}) {
  useEffect(() => {
    if (!userId) return
    identify({ user_id: userId, org_id: orgId ?? '', plan: plan ?? 'trial' })
  }, [userId, orgId, plan])

  return null
}
```

Render this inside your auth provider where user state is available. Works with Clerk, NextAuth, and any library that exposes user state as props.

Pass an opaque internal ID, not an email, name, or anything PII.

## Deploy tagging

```tsx
<FlusterduckScript
  apiKey={process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!}
  environment={process.env.NODE_ENV}
  segment={{ app_version: process.env.NEXT_PUBLIC_APP_VERSION ?? 'unknown' }}
/>
```

In Vercel, set `NEXT_PUBLIC_APP_VERSION` to `$VERCEL_GIT_COMMIT_SHA`. Every session gets tagged with the commit that was live when they landed. Before/after confusion score comparisons then align to actual deploys.

## Gotchas

**`FlusterduckScript` is a Server Component.** If your `layout.tsx` has `'use client'` at the top for an unrelated reason, move `FlusterduckScript` to a separate server-rendered layout wrapper, or switch to `useFlusterduck`.

**`useFlusterduck` and `signal`/`track` from `flusterduck` must be in Client Components.** The SDK doesn't run server-side. Calls in Server Components or during SSR are no-ops: no error, no data.

**Don't use both `FlusterduckScript` and `useFlusterduck` in the same app.** The script loads the SDK via CDN; `useFlusterduck` imports the npm package. They're two separate init paths and you'll get duplicate initialization.

**`useFlusterduck` in multiple components won't double-initialize.** The SDK guards against it, but there's no reason to call it more than once. Initialize at the root, use direct imports everywhere else.

## TypeScript

```ts
import type { SignalData } from 'flusterduck'

const data: SignalData = {
  element: '[data-action="publish"]',
  metadata: { blocked_by: 'missing_title' },
}
signal('dead_click', data)
```
