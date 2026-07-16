# @flusterduck/nuxt

The Nuxt module. Create a client-side plugin returning `createFlusterduckPlugin`. All functions are SSR-safe and only run in the browser.

## Install

```bash
npm install @flusterduck/nuxt flusterduck
```

## Usage

```ts
// plugins/flusterduck.client.ts
import { createFlusterduckPlugin } from '@flusterduck/nuxt'

export default defineNuxtPlugin(() => {
  return createFlusterduckPlugin({
    key: useRuntimeConfig().public.flusterduckKey,
  })
})
```

```vue
<!-- Any component. -->
<script setup lang="ts">
import { track } from '@flusterduck/nuxt'
</script>

<template>
  <button @click="track('checkout_completed', { value: 49 })">Pay</button>
</template>
```

## Links

Published on npm as `@flusterduck/nuxt`. Install pulls the latest published version.
