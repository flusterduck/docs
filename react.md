# React

Wrap your app root with `FlusterduckProvider`. Signal detection starts immediately across the entire tree. That's the whole setup for most apps.

Reach for `useFlusterduck` when initialization needs to be conditional: consent flows, feature flags, auth-gated tracking. Don't use both patterns in the same app.

## Install

```bash
pnpm add flusterduck @flusterduck/react
```

## Setup

```tsx
// src/main.tsx
import { createRoot } from 'react-dom/client'
import { FlusterduckProvider } from '@flusterduck/react'
import { App } from './App'

createRoot(document.getElementById('root')!).render(
  <FlusterduckProvider apiKey={import.meta.env.VITE_FLUSTERDUCK_KEY}>
    <App />
  </FlusterduckProvider>
)
```

```bash
# .env
VITE_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

Note `apiKey`, not `key`. React reserves the `key` prop for reconciliation. It gets stripped before it reaches the component. `apiKey` maps internally to `key` in the SDK config.

## FlusterduckProvider

### Props

| Prop | Type | Default | Description |
|---|---|---|---|
| `apiKey` | `string` | - | Required. Your `fd_pub_` key. |
| `environment` | `string` | - | `"production"`, `"staging"`, `"development"` |
| `sampleRate` | `number` | `1.0` | `0.1` tracks 10% of sessions. Useful for high-traffic apps where full coverage is expensive. |
| `domMode` | `'off' \| 'metadata' \| 'snapshot'` | `'off'` | `'metadata'` captures element attributes with each signal. `'snapshot'` adds computed layout and styles. |
| `cookieless` | `boolean` | `false` | Memory-only session IDs instead of cookies. |
| `respectDoNotTrack` | `boolean` | `true` | Honor `navigator.doNotTrack`. Most teams pass `false`: Flusterduck captures no PII, and the default silently drops 30-50% of visitors on Firefox, Brave, and other privacy-focused browsers. |
| `ignoreElements` | `string[]` | `[]` | CSS selectors to suppress signals on. Good for admin toolbars or debug overlays. |
| `ignorePages` | `string[]` | `[]` | Page paths to skip entirely. |
| `segment` | `Record<string, string>` | - | Static tags on every event: app version, experiment variant, user cohort. |
| `debug` | `boolean` | `false` | Logs every signal and flush to the console. |
| `batchInterval` | `number` | - | Milliseconds between flushes. Default flushes on idle and before unload. |
| `elementImpressionSelectors` | `string[]` | - | CSS selectors to emit an impression signal when they enter the viewport. |
| `enabled` | `boolean` | `true` | `false` skips initialization. Flip to `true` and it initializes then. |

`FlusterduckProvider` accepts anything from the core SDK `Config` type, not just the props listed above. It spreads everything but `apiKey` straight into `init()`. See [SDK Reference](./sdk) for the full option set, including `guide`, `onSignal`, `transport`, and per-signal tuning.

## useFlusterduck hook

Initializes the SDK and returns tracking methods. Use this instead of `FlusterduckProvider` when the init decision isn't static.

```tsx
// src/App.tsx
import { useFlusterduck } from '@flusterduck/react'

export function App() {
  const { signal, track, identify, setConsent, optOut } = useFlusterduck({
    key: import.meta.env.VITE_FLUSTERDUCK_KEY,
  })

  return <Router />
}
```

Options are identical to `FlusterduckProvider` props, but this one uses `key` instead of `apiKey` (no JSX involved, no prop stripping).

### Return value

| Method | Description |
|---|---|
| `signal(name, data?)` | Emit a friction signal manually. |
| `track(name, metadata?)` | Track a business event. |
| `identify(segment)` | Tag the session with user properties. |
| `setConsent(consented)` | Pause or resume collection. |
| `optOut()` | Stop collection permanently for this session. |

## Tracking in components

Initialize once at the root. After that, import from `flusterduck` directly in any component. No hook call needed:

```tsx
import { signal } from 'flusterduck'

export function MultiStepForm({ step }: { step: number }) {
  return (
    <div onMouseLeave={() => signal('task_abandonment', { metadata: { step } })}>
      {/* form content */}
    </div>
  )
}
```

```tsx
import { track } from 'flusterduck'

export function PublishButton({ docId }: { docId: string }) {
  return (
    <button onClick={() => track('document_published', { doc_id: docId })}>
      Publish
    </button>
  )
}
```

Core SDK functions buffer events until the SDK initializes. Calling them before the provider mounts is safe.

## Conditional initialization

`enabled` controls whether the SDK initializes at all. Set it to `false` and nothing runs. Flip it to `true` and the SDK initializes at that point.

```tsx
export function Root() {
  const [consented, setConsented] = useState(false)

  return (
    <FlusterduckProvider
      apiKey={import.meta.env.VITE_FLUSTERDUCK_KEY}
      enabled={consented}
    >
      <App onConsentChange={setConsented} />
    </FlusterduckProvider>
  )
}
```

`enabled: false` skips initialization entirely: the SDK module is never even imported, so nothing runs, no cookie is set, no detector attaches, until you flip it to `true`. `setConsent(false)` is a different mechanism and only does something useful once the SDK has already started: it flushes anything already buffered in memory, sends it, tears the SDK down, and clears the session cookie. It's not a pause that holds data waiting for consent, and calling it before the SDK ever initialized (nothing to flush, no session to clear) is a harmless no-op. If your legal requirement is zero data before consent, use `enabled: false`, not `setConsent(false)` after the fact.

## Consent flow

```tsx
import { setConsent } from 'flusterduck'

export function ConsentBanner({ onDecide }: { onDecide: (accepted: boolean) => void }) {
  return (
    <div>
      <p>We use behavioral analytics to improve this product.</p>
      <button onClick={() => { setConsent(true); onDecide(true) }}>Accept</button>
      <button onClick={() => { setConsent(false); onDecide(false) }}>Decline</button>
    </div>
  )
}
```

If this banner sits inside an already-mounted `FlusterduckProvider`, the SDK is likely running by the time someone clicks Decline, so `setConsent(false)` sends whatever was captured between mount and that click before tearing down. `setConsent(true)` doesn't resume a paused session: it calls `init()` again with the last config the provider passed in, starting a new one. Consent state itself doesn't persist across reloads, you'll re-apply it on mount from whatever storage you use.

## Identifying users

```tsx
import { useEffect } from 'react'
import { identify } from 'flusterduck'

export function IdentityBridge({ userId, plan }: { userId?: string; plan?: string }) {
  useEffect(() => {
    if (!userId) return
    identify({ user_id: userId, plan: plan ?? 'trial' })
  }, [userId, plan])

  return null
}
```

Render this inside your auth provider where `userId` is reliably available. Pass an opaque internal ID, not an email, name, or anything PII.

## Deploy tagging

```tsx
<FlusterduckProvider
  apiKey={import.meta.env.VITE_FLUSTERDUCK_KEY}
  segment={{ app_version: import.meta.env.VITE_APP_VERSION ?? 'unknown' }}
>
  <App />
</FlusterduckProvider>
```

Every session gets tagged with the version that was running when they landed. Confusion score comparisons in the dashboard then map cleanly to your deploy history.

## React StrictMode

StrictMode double-invokes effects in development to surface side effects. The `useFlusterduck` hook handles this correctly. A `useRef` flag prevents double initialization, and the cleanup function cancels any in-flight dynamic import. You won't get duplicate `init()` calls.

## Gotchas

**Use `apiKey`, not `key`.** React strips the `key` prop from elements before they reach the component. `<FlusterduckProvider key="fd_pub_...">` silently passes nothing. This is by design. The fix is already baked in, but if you see the SDK not initializing, check that you're passing `apiKey`.

**Don't call `useFlusterduck` in child components.** There's no reason to. Initialize once at the root, import directly from `flusterduck` everywhere else. Calling `useFlusterduck` in multiple components won't double-initialize, but it creates unnecessary hooks.

**`FlusterduckProvider` and `useFlusterduck` both call `init()` internally.** Don't wrap your app with `FlusterduckProvider` AND call `useFlusterduck` somewhere in the tree. Pick one approach.

## TypeScript

The package ships types, but there's no exported `SignalData` type to import, `signal()`'s second argument is an inline shape. Just pass the object literal and let TypeScript check it structurally:

```ts
import { signal } from 'flusterduck'

signal('form_error', {
  element: '[data-role="submit"]',
  metadata: { validation_errors: ['email_invalid', 'name_required'] },
})
```
