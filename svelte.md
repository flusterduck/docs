# SvelteKit

No store, no plugin object. `@flusterduck/svelte` exports individual functions. Call `initFlusterduck` once in your root layout behind a `browser` guard, then import `signal`, `track`, `identify`, `setConsent`, and `optOut` wherever you need them.

## Install

```bash
pnpm add flusterduck @flusterduck/svelte
```

## Setup

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { initFlusterduck } from '@flusterduck/svelte'
  import { browser } from '$app/environment'

  if (browser) {
    initFlusterduck({
      key: import.meta.env.PUBLIC_FLUSTERDUCK_KEY,
    })
  }
</script>

<slot />
```

```bash
# .env
PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

The `browser` guard isn't optional. SvelteKit runs your layout on the server during SSR, and the SDK is browser-only. Remove the guard and you get a runtime error on first load.

## initFlusterduck

```ts
initFlusterduck({
  key: import.meta.env.PUBLIC_FLUSTERDUCK_KEY,
  environment: 'production',
  sampleRate: 1.0,
})
```

Calling it a second time is a no-op. A module-level `initialized` flag blocks re-entry, so hot reloads and layout remounts don't cause duplicate initialization.

### Options

| Option | Type | Default | Description |
|---|---|---|---|
| `key` | `string` | - | Required. Your `fd_pub_` key. |
| `environment` | `string` | - | `"production"`, `"staging"`, `"development"` |
| `sampleRate` | `number` | `1.0` | `0.5` tracks half of sessions. |
| `domMode` | `'off' \| 'metadata' \| 'snapshot'` | `'off'` | `'metadata'` captures element attributes. `'snapshot'` adds layout and computed styles. |
| `cookieless` | `boolean` | `false` | Memory-only session IDs instead of cookies. |
| `respectDoNotTrack` | `boolean` | `false` | Honor `navigator.doNotTrack`. |
| `ignoreElements` | `string[]` | `[]` | CSS selectors to suppress signals on. |
| `ignorePages` | `string[]` | `[]` | Page paths to skip. |
| `segment` | `Record<string, string>` | - | Static tags on every event. |
| `debug` | `boolean` | `false` | Verbose console logging. |

`destroyFlusterduck()` tears down the SDK and resets the `initialized` flag. Use it in tests or if you're dynamically switching sites.

## Exports

| Export | Signature | Description |
|---|---|---|
| `initFlusterduck` | `(config: Config) => void` | Initialize the SDK. Call once. |
| `destroyFlusterduck` | `() => void` | Tear down and reset. |
| `signal` | `(name: string, data?: SignalData) => void` | Emit a friction signal manually. |
| `track` | `(name: string, metadata?: Record<string, unknown>) => void` | Track a business event. |
| `identify` | `(segment: Record<string, string>) => void` | Tag the session with user properties. |
| `setConsent` | `(consented: boolean) => void` | Pause or resume collection. |
| `optOut` | `() => void` | Stop collection permanently for this session. |

## Tracking in components

```svelte
<script lang="ts">
  import { track } from '@flusterduck/svelte'

  export let planId: string
  export let priceCents: number
</script>

<button on:click={() => track('plan_intent', { plan_id: planId, price_cents: priceCents })}>
  Get started
</button>
```

```svelte
<script lang="ts">
  import { signal } from '@flusterduck/svelte'

  let currentStep = 0
</script>

<div on:mouseleave={() => signal('task_abandonment', { metadata: { step: currentStep } })}>
  <!-- wizard step content -->
</div>
```

Importing from `flusterduck` directly works just as well. Both go through the same initialized SDK instance.

## Consent flow

```svelte
<!-- src/lib/components/ConsentBanner.svelte -->
<script lang="ts">
  import { onMount } from 'svelte'
  import { setConsent } from '@flusterduck/svelte'

  let visible = false

  onMount(() => {
    const stored = localStorage.getItem('fd_consent')
    if (stored === null) {
      setConsent(false)
      visible = true
    } else {
      setConsent(stored === 'true')
    }
  })

  function accept() {
    localStorage.setItem('fd_consent', 'true')
    setConsent(true)
    visible = false
  }

  function decline() {
    localStorage.setItem('fd_consent', 'false')
    setConsent(false)
    visible = false
  }
</script>

{#if visible}
  <div>
    <p>We use behavioral analytics to improve this product.</p>
    <button on:click={accept}>Accept</button>
    <button on:click={decline}>Decline</button>
  </div>
{/if}
```

The `setConsent(false)` call before showing the banner pauses collection immediately. The user's choice then either resumes it or confirms the pause.

## Identifying users

```svelte
<!-- src/lib/components/IdentifyUser.svelte -->
<script lang="ts">
  import { onMount } from 'svelte'
  import { identify } from '@flusterduck/svelte'

  export let userId: string | undefined = undefined
  export let orgId: string | undefined = undefined
  export let plan: string | undefined = undefined

  onMount(() => {
    if (!userId) return
    identify({ user_id: userId, org_id: orgId ?? '', plan: plan ?? 'trial' })
  })
</script>
```

Mount this inside whatever layout has access to your auth state. Pass an opaque internal ID, not an email or display name.

## Deploy tagging

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { initFlusterduck } from '@flusterduck/svelte'
  import { browser } from '$app/environment'

  if (browser) {
    initFlusterduck({
      key: import.meta.env.PUBLIC_FLUSTERDUCK_KEY,
      segment: {
        app_version: import.meta.env.PUBLIC_APP_VERSION ?? 'unknown',
      },
    })
  }
</script>
```

## Gotchas

**The `browser` guard is required, not optional.** If you skip it, SvelteKit's SSR pass throws on first request. The error won't always be obvious. It can manifest as a hydration mismatch rather than an explicit SDK error.

**There's no Svelte store.** If you expected `$flusterduck.track(...)`, that's not how this works. The package exports plain functions. Import them directly where you need them.

**`initFlusterduck` in a `+layout.svelte` runs on both client and server** unless guarded. The `browser` import from `$app/environment` is the canonical way to guard it. Don't use `typeof window !== 'undefined'`. It works but it's not idiomatic SvelteKit.

**Hot module replacement.** During development, Vite's HMR can re-execute your layout script. The `initialized` flag means `initFlusterduck` won't re-run, which is the right behavior. You don't want a new session mid-development.

## TypeScript

```ts
import type { Config, SignalData } from 'flusterduck'
import { initFlusterduck, signal } from '@flusterduck/svelte'

const config: Config = {
  key: import.meta.env.PUBLIC_FLUSTERDUCK_KEY,
  environment: 'production',
}
initFlusterduck(config)

const data: SignalData = {
  element: '[data-action="publish"]',
  metadata: { blocked_by: 'missing_required_fields', field_count: 3 },
}
signal('dead_click', data)
```
