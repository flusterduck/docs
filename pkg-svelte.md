# @flusterduck/svelte

The SvelteKit wrapper. Initialize once in your root layout behind a browser guard, then import the helpers you need anywhere.

## Install

```bash
npm install @flusterduck/svelte flusterduck
```

## Usage

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { browser } from '$app/environment'
  import { initFlusterduck } from '@flusterduck/svelte'

  if (browser) {
    initFlusterduck({ key: 'fd_pub_xxxxxxxxxxxx' })
  }
</script>

<slot />
```

```ts
// Any module.
import { track } from '@flusterduck/svelte'

track('checkout_completed', { value: 49 })
```

## Links

Published on npm as `@flusterduck/svelte`. Install pulls the latest published version.
