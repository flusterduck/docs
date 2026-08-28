# Nuxt 3

`@flusterduck/nuxt` isn't a Nuxt module you drop into the `modules` array. `createFlusterduckPlugin` returns a plugin function; you export it from a `.client.ts` plugin file yourself. After that, import `signal`, `track`, `identify`, `setConsent`, and `optOut` anywhere in your app. All of them check for `window` before doing anything, so calls from server routes or during SSR are silently no-ops instead of errors.

## Install

```bash
pnpm add flusterduck @flusterduck/nuxt
```

## Setup

```ts
// plugins/flusterduck.client.ts
import { createFlusterduckPlugin } from '@flusterduck/nuxt'

export default defineNuxtPlugin((nuxtApp) => {
  return createFlusterduckPlugin({
    key: useRuntimeConfig().public.flusterduckKey,
  })(nuxtApp)
})
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      flusterduckKey: '',
    },
  },
})
```

```bash
# .env
NUXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

Nuxt maps `NUXT_PUBLIC_FLUSTERDUCK_KEY` to `runtimeConfig.public.flusterduckKey` automatically. The naming convention handles the wiring. No manual `process.env` access needed in production.

`createFlusterduckPlugin(options)` returns a function shaped `(nuxtApp) => Promise<void>`, which is exactly what a Nuxt plugin needs to be. `defineNuxtPlugin` is what gives you a safe place to call `useRuntimeConfig()` (it needs an active Nuxt context, which isn't guaranteed at module top level), and its callback receives `nuxtApp`, which you then pass straight into the function `createFlusterduckPlugin` handed back. Skipping that last call, calling `createFlusterduckPlugin({...})` and just returning the function without invoking it, is a real mistake to watch for: the SDK will never initialize and nothing will error, because Nuxt doesn't require a plugin to do anything.

If your key is a plain string (no runtime config needed), you can skip `defineNuxtPlugin` entirely and export the plugin function directly, matching the shape Nuxt expects:

```ts
// plugins/flusterduck.client.ts
import { createFlusterduckPlugin } from '@flusterduck/nuxt'

export default createFlusterduckPlugin({ key: 'fd_pub_xxxxxxxxxxxx' })
```

The `.client.ts` suffix is what makes this browser-only. Without it, Nuxt runs the plugin during SSR and the SDK's browser guard exits before doing anything useful, so you'd get a plugin that quietly never initializes on the server render and only catches up once the client bundle runs the same file again. The suffix is not optional.

## createFlusterduckPlugin

Builds the plugin function. When Nuxt calls it, it initializes the SDK and, if `nuxtApp.hook` is available, registers a teardown on `app:unmounted` so listeners and timers don't leak across SPA re-mounts or test runs.

### Options

Anything from the core SDK `Config` type works here since `createFlusterduckPlugin` forwards the whole options object to `init()` unfiltered.

| Option | Type | Default | Description |
|---|---|---|---|
| `key` | `string` | - | Required. Your `fd_pub_` key. |
| `environment` | `string` | - | `"production"`, `"staging"`, `"development"` |
| `sampleRate` | `number` | `1.0` | `0.1` tracks 10% of sessions. |
| `domMode` | `'off' \| 'metadata' \| 'snapshot'` | `'off'` | `'metadata'` captures element attributes with each signal. `'snapshot'` adds computed layout and styles. |
| `cookieless` | `boolean` | `false` | Memory-only session IDs instead of cookies. |
| `respectDoNotTrack` | `boolean` | `true` | Honor `navigator.doNotTrack`. Most teams pass `false`: Flusterduck captures no PII, and the default silently drops 30-50% of visitors on Firefox, Brave, and other privacy-focused browsers. |
| `ignoreElements` | `string[]` | `[]` | CSS selectors to suppress signals on. |
| `ignorePages` | `string[]` | `[]` | Page paths to skip. |
| `segment` | `Record<string, string>` | - | Static tags on every event. |
| `debug` | `boolean` | `false` | Verbose console logging. |
| `batchInterval` | `number` | - | Milliseconds between flushes. Default flushes on idle and before unload. |
| `elementImpressionSelectors` | `string[]` | - | CSS selectors to emit an impression signal when they enter the viewport. |

Passing a secret key (`fd_sec_`) logs an error and returns an inert plugin function. Calling it still won't throw, it just never initializes.

## defineFlusterduckConfig

A typed helper for defining config once and referencing it in multiple places (plugin, tests, Storybook). It's narrower than the full `Config` type: it only forwards `key`, `environment`, `sampleRate`, `debug`, `cookieless`, `respectDoNotTrack`, and `segment`. If you need `domMode`, `ignoreElements`, `ignorePages`, `batchInterval`, or `elementImpressionSelectors`, pass them straight to `createFlusterduckPlugin` instead, this helper silently drops anything outside its seven fields.

```ts
// flusterduck.config.ts
import { defineFlusterduckConfig } from '@flusterduck/nuxt'

export default defineFlusterduckConfig({
  key: process.env.NUXT_PUBLIC_FLUSTERDUCK_KEY!,
  environment: process.env.NODE_ENV,
  sampleRate: 1.0,
})
```

```ts
// plugins/flusterduck.client.ts
import { createFlusterduckPlugin } from '@flusterduck/nuxt'
import config from '../flusterduck.config'

export default defineNuxtPlugin((nuxtApp) => createFlusterduckPlugin(config)(nuxtApp))
```

## Tracking in components

Nuxt auto-imports Vue composition API, so you don't need explicit `import` statements for `ref`, `onMounted`, `watch`, etc. in `.vue` files.

```vue
<!-- pages/billing.vue -->
<script setup lang="ts">
import { track, signal } from '@flusterduck/nuxt'

function onPlanSelect(planId: string, priceCents: number) {
  track('plan_intent', { plan_id: planId, price_cents: priceCents })
}

function onPaymentError(code: string) {
  signal('interaction_error', { metadata: { error_code: code } })
}
</script>

<template>
  <div>
    <button @click="onPlanSelect('scale', 9900)">Scale plan</button>
  </div>
</template>
```

Importing from `flusterduck` directly works too. Both go through the same SDK instance.

## Exports

| Export | Signature | Description |
|---|---|---|
| `createFlusterduckPlugin` | `(options) => (nuxtApp) => Promise<void>` | Plugin factory. Export the returned function (or invoke it inside `defineNuxtPlugin`) from a `.client.ts` plugin file. |
| `defineFlusterduckConfig` | `(options) => options` | Typed config helper. Forwards only `key`, `environment`, `sampleRate`, `debug`, `cookieless`, `respectDoNotTrack`, `segment`. |
| `signal` | `(name: string, data?: { element?: string; metadata?: Record<string, unknown>; weight?: number }) => void` | Emit a friction signal manually. |
| `track` | `(name: string, metadata?: Record<string, unknown>) => void` | Track a business event. |
| `identify` | `(segment: Record<string, string>) => void` | Tag the session with user properties. |
| `setConsent` | `(consented: boolean) => void` | Pause or resume collection. |
| `optOut` | `() => void` | Stop collection permanently for this session. |

## Consent flow

```vue
<!-- components/ConsentBanner.vue -->
<script setup lang="ts">
import { setConsent } from '@flusterduck/nuxt'

const visible = ref(false)

onMounted(() => {
  const stored = localStorage.getItem('fd_consent')
  if (stored === null) {
    setConsent(false)
    visible.value = true
  } else {
    setConsent(stored === 'true')
  }
})

function accept() {
  localStorage.setItem('fd_consent', 'true')
  setConsent(true)
  visible.value = false
}

function decline() {
  localStorage.setItem('fd_consent', 'false')
  setConsent(false)
  visible.value = false
}
</script>

<template>
  <div v-if="visible">
    <p>We use behavioral analytics to improve this product.</p>
    <button @click="accept">Accept</button>
    <button @click="decline">Decline</button>
  </div>
</template>
```

`setConsent(false)` isn't a pause. It flushes anything already buffered, sends it, then tears the SDK down and clears the session cookie. If the plugin already initialized before this component mounted (likely, since plugins run before page components), whatever the SDK captured in that gap gets sent before teardown. If you need zero data collected before explicit consent, don't run `createFlusterduckPlugin` at all until you have it, gate the plugin file's own logic behind your stored consent value instead of calling `setConsent(false)` after the fact.

`setConsent(true)` calls `init()` again with the config from the last `createFlusterduckPlugin` call, starting a fresh session. It doesn't resume whatever was flushed and cleared by `setConsent(false)`.

## Identifying users

```vue
<!-- components/IdentifyUser.vue -->
<script setup lang="ts">
import { identify } from '@flusterduck/nuxt'

const props = defineProps<{
  userId?: string
  orgId?: string
  plan?: string
}>()

watchEffect(() => {
  if (!props.userId) return
  identify({
    user_id: props.userId,
    org_id: props.orgId ?? '',
    plan: props.plan ?? 'trial',
  })
})
</script>
```

Render this inside whatever layout has access to your auth state. Pass an opaque internal ID, not an email or display name.

## Deploy tagging

```ts
// plugins/flusterduck.client.ts
import { createFlusterduckPlugin } from '@flusterduck/nuxt'

export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()
  return createFlusterduckPlugin({
    key: config.public.flusterduckKey,
    segment: {
      app_version: config.public.appVersion ?? 'unknown',
    },
  })(nuxtApp)
})
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      flusterduckKey: '',
      appVersion: '',
    },
  },
})
```

In Vercel, set `NUXT_PUBLIC_APP_VERSION` to `$VERCEL_GIT_COMMIT_SHA` in your project environment variables.

`segment.app_version` tags sessions for correlation, it isn't the deploy record itself. For `confusion_before`/`confusion_after` and issue verification, see [Deploy Correlation](./deploy-correlation): connect your GitHub repo and Flusterduck detects deploys on its own, no plugin or manual API call needed for most teams.

## Gotchas

**`.client.ts` is required.** Name your plugin file `flusterduck.client.ts`, not `flusterduck.ts`. Without the suffix, Nuxt runs the plugin on the server, where the SDK's `window` guard makes `init()` a no-op, so nothing initializes there and the client run has to carry the whole job anyway. Skipping the suffix doesn't throw, it just makes the SSR pass pointless and can surface as inconsistent behavior between first paint and hydration.

**`createFlusterduckPlugin` returns a function. Call it.** `export default defineNuxtPlugin(() => createFlusterduckPlugin({...}))` looks reasonable and does nothing: the callback returns the plugin function instead of running it, so `init()` never fires. Nuxt doesn't error on this, so the SDK just stays uninitialized with no signal that anything's wrong. Either export `createFlusterduckPlugin({...})` directly, or invoke the returned function with `nuxtApp` inside `defineNuxtPlugin((nuxtApp) => createFlusterduckPlugin({...})(nuxtApp))`.

**`useRuntimeConfig()` needs the `defineNuxtPlugin` callback.** Don't call it at module top level in your plugin file unless you're also calling `createFlusterduckPlugin` directly as the default export (no wrapper). Once you introduce a `defineNuxtPlugin(() => {...})` callback, do the `useRuntimeConfig()` call inside it.

**`NUXT_PUBLIC_*` env vars need the right key casing.** Nuxt maps `NUXT_PUBLIC_FLUSTERDUCK_KEY` to `runtimeConfig.public.flusterduckKey` using camelCase conversion. If you see `undefined`, check that the env var prefix and the `runtimeConfig.public` key name match the convention.

**Tracking functions in server routes do nothing.** The `typeof window` guard in every function exits silently on the server. This is intentional: you won't get errors, but you also won't get data.

## TypeScript

```ts
import { signal } from 'flusterduck'

signal('dead_click', {
  element: '[data-action="confirm-delete"]',
  metadata: { resource_type: 'workspace', confirmed: false },
})
```
