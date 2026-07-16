# @flusterduck/vue

The Vue wrapper. Register the plugin once in your app entry and signal detection starts immediately. Access the SDK in any component with the `useFlusterduck` composable.

## Install

```bash
npm install @flusterduck/vue flusterduck
```

## Usage

```ts
// main.ts
import { createApp } from 'vue'
import { FlusterduckPlugin } from '@flusterduck/vue'
import App from './App.vue'

createApp(App)
  .use(FlusterduckPlugin, { key: 'fd_pub_xxxxxxxxxxxx' })
  .mount('#app')
```

```vue
<!-- Any component. -->
<script setup lang="ts">
import { useFlusterduck } from '@flusterduck/vue'

const { track } = useFlusterduck()
</script>

<template>
  <button @click="track('checkout_completed', { value: 49 })">Pay</button>
</template>
```

## Links

Published on npm as `@flusterduck/vue`. Install pulls the latest published version.
