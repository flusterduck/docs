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

Full props table, StrictMode behavior, and the consent-timing gotcha: [Next.js](./next).

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
    signal('form_abandonment', { metadata: { form: 'checkout' } })
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

Full props table, StrictMode behavior, and the consent-timing gotcha: [React](./react).

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
  key: import.meta.env.VITE_FLUSTERDUCK_KEY,
  segment: { app_version: import.meta.env.VITE_APP_VERSION ?? 'unknown' },
})

app.mount('#app')
```

```bash
# .env
VITE_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

Note the option is `key`, not `apiKey`. The plugin checks `options.key` specifically; anything else passed there is silently ignored and nothing initializes.

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

Full plugin options, gotchas, and SSR notes: [Vue](./vue).

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

The option is `key`, not `apiKey`. `initFlusterduck` checks `config.key` specifically, so a mistyped option name fails silently instead of erroring.

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

Full exports, gotchas, and why there's no store: [SvelteKit](./svelte).

---

## Nuxt 3

```bash
pnpm add flusterduck @flusterduck/nuxt
```

There's no `modules` entry for this one. `@flusterduck/nuxt` exports a plugin factory, not a Nuxt module, so you wire it up as a client-only plugin file yourself:

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
    public: { flusterduckKey: '' },
  },
})
```

```bash
# .env
NUXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

The `.client.ts` suffix keeps the plugin off the server render, where the SDK can't run anyway. Import `signal` and `track` from `@flusterduck/nuxt` in your pages and components:

```vue
<script setup lang="ts">
import { track, signal } from '@flusterduck/nuxt'

function onCheckout() {
  track('plan_intent', { plan_id: 'grow', amount_cents: 3900, billing: 'monthly' })
}
</script>
```

Full plugin wiring, options, and the mistake to avoid (returning `createFlusterduckPlugin(...)` without calling it): [Nuxt 3](./nuxt).

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

Or skip the build step with the script tag:

```html
<script src="https://flusterduck.com/d.js" data-key="fd_pub_xxxxxxxxxxxx" data-dnt="false" async></script>
```

No `Flusterduck.init(...)` call needed. The bootstrap reads its config straight off the tag's `data-*` attributes. Keep `data-dnt="false"`: the SDK captures no PII, and the default Do Not Track behavior silently drops 30-50% of visitors on Firefox, Brave, and other privacy-focused browsers. Full script-tag reference: [Installation](./install).

The SDK has no opinion about your framework. It only has opinions about your buttons.

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

## Injecting the script tag at build time

Plain Vite or webpack apps without one of the framework wrappers above can skip hand-editing `index.html` too: [Build Plugins](./build-plugins) covers `flusterduck-vite-plugin` and `flusterduck-webpack-plugin`, which inject the script tag into your built HTML automatically.
