# Consent and GDPR

There are two ways to handle consent with Flusterduck. They behave differently and the choice affects your compliance posture.

**`setConsent(false)`** stops the active SDK session, clears the buffer, and removes the session cookie. When the user accepts, `setConsent(true)` reinitializes with the previous config if the SDK had already been configured.

**`enabled: false`** (React/Next.js) skips initialization entirely. No SDK, no listeners, nothing. When the user accepts, you flip `enabled` to `true` and the SDK initializes fresh with no prior session data.

If your legal requirement is zero data collection before consent, use `enabled: false` or wait to call `init()` until consent is granted.

When in doubt, `enabled: false` is the safer choice.

## setConsent pattern

```tsx
// components/ConsentBanner.tsx
'use client'
import { useEffect, useState } from 'react'
import { setConsent } from 'flusterduck'

export function ConsentBanner() {
  const [visible, setVisible] = useState(false)

  useEffect(() => {
    const stored = localStorage.getItem('fd_consent')
    if (stored === null) {
      setConsent(false)   // pause before user decides
      setVisible(true)
    } else {
      setConsent(stored === 'true')  // restore previous decision
    }
  }, [])

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

  if (!visible) return null

  return (
    <div>
      <p>We use behavioral analytics to improve this product.</p>
      <button onClick={accept}>Accept</button>
      <button onClick={decline}>Decline</button>
    </div>
  )
}
```

Call `setConsent(false)` before showing the banner if the SDK may already be initialized. Collection stops at that point. Don't wait for the user to see the banner before stopping collection.

Consent state doesn't survive page loads. Re-apply it from localStorage (or your consent management platform) on every mount.

## enabled: false pattern

```tsx
// app/layout.tsx or src/main.tsx
export function Root() {
  const [consented, setConsented] = useState(() =>
    localStorage.getItem('fd_consent') === 'true'
  )

  return (
    <FlusterduckProvider
      apiKey={process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!}
      enabled={consented}
    >
      <App onConsentChange={setConsented} />
    </FlusterduckProvider>
  )
}
```

When `enabled` flips to `true`, the SDK initializes for the first time. There's no prior session, no prior buffer. The session starts at consent.

For `useFlusterduck`:

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

## Cookieless mode

By default, Flusterduck uses a session cookie to maintain session continuity across page loads. In cookie-restricted environments, set `cookieless: true`:

```ts
init({
  key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!,
  cookieless: true,
})
```

Cookieless mode uses a memory-only session ID. It avoids setting the Flusterduck session cookie, but a new session starts after a page load or tab restart. Signal collection works the same way.

For the strictest GDPR posture: use `enabled: false` before consent and `cookieless: true` after. No data collection of any kind before the user accepts, and no persistent identifiers after.

## respectDoNotTrack

```ts
init({
  key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!,
  respectDoNotTrack: true,
})
```

When set, the SDK checks `navigator.doNotTrack` at initialization. If it's `"1"`, the SDK doesn't initialize. Equivalent to never calling `init()`.

Off by default. Most legal frameworks don't require honoring DNT. It's a browser preference signal, not a legal standard in most jurisdictions. Turn it on if your privacy policy commits to it.

## optOut()

`optOut()` is for users who explicitly request no tracking, separate from the consent flow:

```ts
import { optOut } from 'flusterduck'

// In a "stop tracking" button in your account settings
optOut()
```

`optOut()` stops collection immediately and clears the session. Keep your own preference state if the user has asked not to be tracked, and don't reinitialize the SDK while that preference is active.

Use `optOut()` for explicit user requests ("don't track me"). Use `setConsent(false)` for consent banner declines, which can be reconsidered.

## GDPR compliance checklist

- Initialize with `enabled: false` or call `setConsent(false)` before consent is collected
- Restore the user's previous consent decision from localStorage on every page load
- Wire a "manage preferences" or "withdraw consent" option to `setConsent(false)` or `optOut()`
- Never pass PII via `identify()`, `track()`, or `signal()` metadata
- Use `cookieless: true` if you aren't separately collecting cookie consent
- Pass `environment: "development"` in non-production environments so test sessions don't pollute production data

See [Privacy](./privacy) for the full privacy architecture and data handling details.
