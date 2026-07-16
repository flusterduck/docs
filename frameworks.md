# Framework Guides

Framework-specific wrappers handle initialization and lifecycle automatically. They all install alongside the core SDK.

## Next.js

### App Router

```bash
pnpm add flusterduck @flusterduck/next
```

Add `FlusterduckScript` to your root layout. It renders a script tag that initializes the SDK after the page loads.

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

`FlusterduckScript` accepts all the same options as `init()`:

```tsx
<FlusterduckScript
  apiKey={process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!}
  environment="production"
  sampleRate={0.5}
  segment={{ app_version: process.env.NEXT_PUBLIC_APP_VERSION ?? 'unknown' }}
/>
```

### Pages Router

Same approach, but add it to `_app.tsx`:

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

### useFlusterduck hook

The `@flusterduck/next` package exports a `useFlusterduck` hook for accessing SDK methods in client components:

```tsx
'use client'
import { useFlusterduck } from '@flusterduck/next'

export function UpgradeButton() {
  const { signal, track, setConsent, optOut } = useFlusterduck()

  const handleClick = () => {
    track('plan_intent', { plan_id: 'scale', amount_cents: 9900, billing: 'monthly' })
  }

  return <button onClick={handleClick}>Upgrade to Scale</button>
}
```

---

## React

```bash
pnpm add flusterduck @flusterduck/react
```

Wrap your app with `FlusterduckProvider`:

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

Use the `useFlusterduck` hook anywhere inside the provider:

```tsx
import { useFlusterduck } from '@flusterduck/react'

function CheckoutForm() {
  const { signal, track } = useFlusterduck()

  const handleAbandon = () => {
    signal('form_abandonment', { form: 'checkout' })
  }

  const handleComplete = () => {
    track('subscription_started', {
      plan_id: 'grow',
      amount_cents: 3900,
      billing: 'monthly',
    })
  }

  return (
    <form onBlur={handleAbandon} onSubmit={handleComplete}>
      {/* ... */}
    </form>
  )
}
```

The provider accepts the same config options as `init()`:

```tsx
<FlusterduckProvider
  apiKey={import.meta.env.VITE_FLUSTERDUCK_KEY}
  sampleRate={0.5}
  enabled={userHasConsented}
  segment={{ app_version: __APP_VERSION__ }}
>
```

---

## Vue 3

```bash
pnpm add flusterduck @flusterduck/vue
```

Register the plugin in your app entry:

```ts
// src/main.ts
import { createApp } from 'vue'
import { FlusterduckPlugin } from '@flusterduck/vue'
import App from './App.vue'

const app = createApp(App)

app.use(FlusterduckPlugin, {
  apiKey: import.meta.env.VITE_FLUSTERDUCK_KEY,
  segment: { app_version: import.meta.env.VITE_APP_VERSION ?? 'unknown' },
})

app.mount('#app')
```

```bash
# .env
VITE_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

Use the `useFlusterduck` composable in your components:

```vue
<script setup lang="ts">
import { useFlusterduck } from '@flusterduck/vue'

const { signal, track, setConsent } = useFlusterduck()

function onUpgradeClick() {
  track('plan_intent', { plan_id: 'scale', amount_cents: 9900 })
}
</script>

<template>
  <button @click="onUpgradeClick">Upgrade</button>
</template>
```

---

## SvelteKit

```bash
pnpm add flusterduck @flusterduck/svelte
```

Initialize in your root layout:

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { initFlusterduck } from '@flusterduck/svelte'
  import { browser } from '$app/environment'

  if (browser) {
    initFlusterduck({
      apiKey: import.meta.env.PUBLIC_FLUSTERDUCK_KEY,
    })
  }
</script>

<slot />
```

```bash
# .env
PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

Use the tracking functions in any component:

```svelte
<script>
  import { track } from '@flusterduck/svelte'

  function handleUpgrade() {
    track('plan_intent', { plan_id: 'scale', amount_cents: 9900 })
  }
</script>

<button on:click={handleUpgrade}>Upgrade</button>
```

---

## Nuxt 3

```bash
pnpm add flusterduck @flusterduck/nuxt
```

Add the module to your Nuxt config:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@flusterduck/nuxt'],

  flusterduck: {
    apiKey: process.env.NUXT_PUBLIC_FLUSTERDUCK_KEY,
    segment: { app_version: process.env.NUXT_PUBLIC_APP_VERSION ?? 'unknown' },
  },
})
```

```bash
# .env
NUXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

Use the composable in your pages and components:

```vue
<script setup lang="ts">
const { signal, track } = useFlusterduck()

function onCheckout() {
  track('plan_intent', { plan_id: 'grow', amount_cents: 3900, billing: 'monthly' })
}
</script>
```

The module auto-initializes on the client side. No additional setup needed.

---

## Vanilla JS / any framework

```bash
pnpm add flusterduck
```

Call `init()` once, as early as possible in the browser lifecycle:

```ts
import { init, signal, track } from 'flusterduck'

init({
  key: 'fd_pub_xxxxxxxxxxxx',
  segment: { app_version: '2026.06.10' },
})
```

Or via script tag (CDN):

```html
<script src="https://cdn.flusterduck.com/sdk/latest/flusterduck.min.js"></script>
<script>
  Flusterduck.init({ key: 'fd_pub_xxxxxxxxxxxx' })
</script>
```

---

## Environment variables by framework

| Framework | Variable name pattern |
|---|---|
| Next.js | `NEXT_PUBLIC_FLUSTERDUCK_KEY` |
| Vite / React / Vue | `VITE_FLUSTERDUCK_KEY` |
| SvelteKit | `PUBLIC_FLUSTERDUCK_KEY` |
| Nuxt | `NUXT_PUBLIC_FLUSTERDUCK_KEY` |
| Node / server | `FLUSTERDUCK_SECRET_KEY` (use `fd_sec_`) |

The `fd_pub_` key is the only key safe for client-side environment variables. Everything else stays on the server.
