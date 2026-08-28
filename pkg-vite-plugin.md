# flusterduck-vite-plugin

The Vite plugin. It injects the Flusterduck script tag into your built HTML automatically, so you don't hand-edit the template.

## Install

```bash
npm install -D flusterduck-vite-plugin
```

## Usage

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { flusterduck } from 'flusterduck-vite-plugin'

export default defineConfig({
  plugins: [
    flusterduck({
      apiKey: 'fd_pub_xxxxxxxxxxxx',
      environment: 'production',
    }),
  ],
})
```

The plugin adds the script tag to `<head>` during `transformIndexHtml`, pointing at the jsdelivr-hosted SDK bundle. Pass a publishable key only (`fd_pub_`): a secret key is rejected and logged as an error instead of injected. See the [build plugins guide](./build-plugins) for the full option list.

## Links

Published on npm as `flusterduck-vite-plugin`. Install pulls the latest published version.
