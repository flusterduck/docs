# Nuxt 3

Create a client-side plugin, return `createFlusterduckPlugin` from it, done. After that, import `signal`, `track`, `identify`, `setConsent`, and `optOut` anywhere in your app. All functions are SSR-safe. Calls that happen server-side are silently swallowed.

## Install

```bash
pnpm add flusterduck @flusterduck/nuxt
```

## Setup

```ts
// plugins/flusterduck.client.ts
import { createFlusterduckPlugin } from '@flusterduck/nuxt'

export default defineNuxtPlugin(() => {
  return createFlusterduckPlugin({
    key: useRuntimeConfig().public.flusterduckKey,
  })
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

The `.client.ts` suffix is what makes this browser-only. Without it, Nuxt runs the plugin during SSR and the SDK throws. The suffix is not optional.

## createFlusterduckPlugin

Builds the plugin object returned to `defineNuxtPlugin`. Initializes the SDK via `configureApp` and injects the tracking script into the document head.

### Options

| Option | Type | Default | Description |
|---|---|---|---|
| `key` | `string` | - | Required. Your `fd_pub_` key. |
| `environment` | `string` | - | `"production"`, `"staging"`, `"development"` |
| `sampleRate` | `number` | `1.0` | `0.1` tracks 10% of sessions. |
| `cookieless` | `boolean` | `false` | Memory-only session IDs instead of cookies. |
| `respectDoNotTrack` | `boolean` | `false` | Honor `navigator.doNotTrack`. |
| `segment` | `Record<string, string>` | - | Static tags on every event. |
| `debug` | `boolean` | `false` | Verbose console logging. |

Passing a secret key (`fd_sec_`) logs an error and returns an inert plugin. Nothing initializes.

## defineFlusterduckConfig

A typed helper that validates your config shape. Useful when you want to define config once and reference it in multiple places (plugin, tests, Storybook):

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

export default defineNuxtPlugin(() => createFlusterduckPlugin(config))
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
| `createFlusterduckPlugin` | `(options) => object` | Plugin factory. Return from `defineNuxtPlugin`. |
| `defineFlusterduckConfig` | `(options) => options` | Typed config helper. |
| `signal` | `(name: string, data?: SignalData) => void` | Emit a friction signal manually. |
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

export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()
  return createFlusterduckPlugin({
    key: config.public.flusterduckKey,
    segment: {
      app_version: config.public.appVersion ?? 'unknown',
    },
  })
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

## Gotchas

**`.client.ts` is required.** Name your plugin file `flusterduck.client.ts`, not `flusterduck.ts`. Without the suffix, Nuxt runs the plugin on the server and the SDK init throws. The error surfaces as a hydration mismatch, which can be confusing to trace back.

**`useRuntimeConfig()` only works inside Nuxt context.** Don't call it at module top-level in your plugin file. The `defineNuxtPlugin(() => { ... })` callback is the right place, since `useRuntimeConfig` is available there. The pattern in the setup section above is correct.

**`NUXT_PUBLIC_*` env vars need the right key casing.** Nuxt maps `NUXT_PUBLIC_FLUSTERDUCK_KEY` to `runtimeConfig.public.flusterduckKey` using camelCase conversion. If you see `undefined`, check that the env var prefix and the `runtimeConfig.public` key name match the convention.

**Tracking functions in Server Components or `server/` routes do nothing.** The `typeof window` guard in every function exits silently on the server. This is intentional: you won't get errors, but you also won't get data.

## TypeScript

```ts
import type { SignalData } from 'flusterduck'

const data: SignalData = {
  element: '[data-action="confirm-delete"]',
  metadata: { resource_type: 'workspace', confirmed: false },
}
signal('dead_click', data)
```
