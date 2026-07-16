# Vue 3

Register `FlusterduckPlugin` once in your app entry. Signal detection starts immediately for the entire app. After that, import directly from `flusterduck` in any component. No composable needed for most cases.

## Install

```bash
pnpm add flusterduck @flusterduck/vue
```

## Setup

```ts
// src/main.ts
import { createApp } from 'vue'
import { FlusterduckPlugin } from '@flusterduck/vue'
import App from './App.vue'

const app = createApp(App)

app.use(FlusterduckPlugin, {
  key: import.meta.env.VITE_FLUSTERDUCK_KEY,
})

app.mount('#app')
```

```bash
# .env
VITE_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

## FlusterduckPlugin options

| Option | Type | Default | Description |
|---|---|---|---|
| `key` | `string` | - | Required. Your `fd_pub_` key. |
| `environment` | `string` | - | `"production"`, `"staging"`, `"development"` |
| `sampleRate` | `number` | `1.0` | `0.25` tracks one in four sessions. |
| `domMode` | `'off' \| 'metadata' \| 'snapshot'` | `'off'` | `'metadata'` captures element attributes. `'snapshot'` adds layout and computed styles. |
| `cookieless` | `boolean` | `false` | Memory-only session IDs instead of cookies. |
| `respectDoNotTrack` | `boolean` | `false` | Honor `navigator.doNotTrack`. |
| `ignoreElements` | `string[]` | `[]` | CSS selectors to suppress signals on. |
| `ignorePages` | `string[]` | `[]` | Page paths to skip. |
| `segment` | `Record<string, string>` | - | Static tags on every event. |
| `debug` | `boolean` | `false` | Verbose console logging. |

## useFlusterduck composable

Returns tracking methods. Takes no arguments. The plugin handles initialization.

```vue
<script setup lang="ts">
import { useFlusterduck } from '@flusterduck/vue'

const { signal, track, identify, setConsent, optOut } = useFlusterduck()
</script>
```

If you're coming from React: `useFlusterduck()` in Vue takes no options. Don't pass a key here. It won't do anything. Configuration lives entirely in `app.use(FlusterduckPlugin, { ... })`.

### Return value

| Method | Description |
|---|---|
| `signal(name, data?)` | Emit a friction signal manually. |
| `track(name, metadata?)` | Track a business event. |
| `identify(segment)` | Tag the session with user properties. |
| `setConsent(consented)` | Pause or resume collection. |
| `optOut()` | Stop collection permanently for this session. |

## Tracking in components

For most components, skip `useFlusterduck` entirely and import from `flusterduck` directly:

```vue
<script setup lang="ts">
import { signal, track } from 'flusterduck'
</script>

<template>
  <button @click="track('export_triggered', { format: 'csv', rows: rowCount })">
    Export CSV
  </button>
</template>
```

```vue
<script setup lang="ts">
import { signal } from 'flusterduck'

function onStepLeave(step: number) {
  signal('task_abandonment', { metadata: { step, form: 'onboarding' } })
}
</script>

<template>
  <div @mouseleave="onStepLeave(currentStep)">
    <!-- wizard step content -->
  </div>
</template>
```

Core SDK functions buffer events until the SDK is ready. Safe to call before initialization resolves.

## Consent flow

```vue
<!-- components/ConsentBanner.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { setConsent } from 'flusterduck'

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
import { watch } from 'vue'
import { identify } from 'flusterduck'

const props = defineProps<{
  userId?: string
  orgId?: string
  plan?: string
}>()

watch(
  () => props.userId,
  (userId) => {
    if (!userId) return
    identify({
      user_id: userId,
      org_id: props.orgId ?? '',
      plan: props.plan ?? 'trial',
    })
  },
  { immediate: true },
)
</script>
```

Render this inside whatever component owns your auth state. Pass an opaque internal ID, not an email or display name.

## Vue Router

Route-to-route navigation is tracked automatically via the history API. No router hooks needed.

If you want to attach route metadata to sessions:

```ts
router.afterEach((to) => {
  if (to.meta.product) {
    identify({ product_area: String(to.meta.product) })
  }
})
```

## Deploy tagging

```ts
app.use(FlusterduckPlugin, {
  key: import.meta.env.VITE_FLUSTERDUCK_KEY,
  segment: {
    app_version: import.meta.env.VITE_APP_VERSION ?? 'unknown',
  },
})
```

## Gotchas

**`useFlusterduck()` takes no arguments.** Options go in `app.use`, not the composable. Passing config to `useFlusterduck` has no effect. It won't initialize a second instance and it won't override anything.

**The plugin doesn't support conditional initialization.** If you need to gate on consent state before initializing, call `init` from `flusterduck` directly with a Pinia action or composable that owns your consent state.

**Server-side rendering.** If you're using Vue with SSR (Nuxt, Vite SSR), the plugin's browser guard prevents SDK calls on the server. `app.use(FlusterduckPlugin, ...)` itself can run on the server, but it exits early without initializing.

## TypeScript

```ts
import type { SignalData } from 'flusterduck'

const data: SignalData = {
  element: '[data-testid="submit-payment"]',
  metadata: { error_code: 'card_declined', attempt: 2 },
}
signal('interaction_error', data)
```
