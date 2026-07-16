# flusterduck-vite-plugin

The Vite plugin. It handles deploy tagging and source map upload at build time so issue evidence resolves to your original source.

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
      secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
      release: process.env.VITE_APP_VERSION,
      environment: 'production',
    }),
  ],
})
```

The plugin records a deploy when the build completes and uploads source maps so stack traces in issue evidence map back to original file names and line numbers. Keep `FLUSTERDUCK_SECRET_KEY` in your build environment, never in client code.

## Links

Published on npm as `flusterduck-vite-plugin`. Install pulls the latest published version.
