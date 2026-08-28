# Build Plugins

`flusterduck-vite-plugin` and `flusterduck-webpack-plugin` do one thing: inject the Flusterduck script tag into your built HTML automatically, so you don't hand-edit `index.html`. That's the whole feature. They don't record deploys, upload source maps, or talk to your Flusterduck account at build time. They ship an `fd_pub_` key into your HTML, same as pasting the tag yourself.

If you're on Next.js, React, Vue, Svelte, or Nuxt with a framework wrapper, skip this page. `@flusterduck/react`, `@flusterduck/next`, `@flusterduck/vue`, `@flusterduck/svelte`, and `@flusterduck/nuxt` already handle this and do more (hooks, SSR safety, `setConsent`). These build plugins are for a plain Vite or webpack app that isn't using one of those, and would otherwise paste the script tag into a static HTML template by hand.

## Vite

```bash
pnpm add -D flusterduck-vite-plugin
```

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { flusterduck } from 'flusterduck-vite-plugin'

export default defineConfig({
  plugins: [
    flusterduck({
      apiKey: process.env.VITE_FLUSTERDUCK_KEY,
    }),
  ],
})
```

The plugin hooks Vite's `transformIndexHtml` and appends the script tag to `<head>` in every HTML entry point at build time. Run `vite build` and check the output `dist/index.html`: you'll see a `<script src="https://cdn.jsdelivr.net/npm/flusterduck@0/...">` tag with your key baked in as `data-key`.

## webpack

```bash
pnpm add -D flusterduck-webpack-plugin
```

```js
// webpack.config.js
const FlusterduckPlugin = require('flusterduck-webpack-plugin')

module.exports = {
  plugins: [
    new FlusterduckPlugin({
      apiKey: process.env.FLUSTERDUCK_KEY,
    }),
  ],
}
```

`flusterduck-webpack-plugin` exports the plugin class directly (`export = FlusterduckWebpackPlugin`), not a named export, so `require('flusterduck-webpack-plugin')` (or `import FlusterduckPlugin from 'flusterduck-webpack-plugin'` with `esModuleInterop`) gives you the class itself.

If `html-webpack-plugin` is in your config, the plugin taps its `alterAssetTagGroups` hook and adds the script to the generated head tags. Without it, the plugin scans the assets webpack emits and injects the tag directly into any `.html` output, so it works either way.

## Options

Both plugins take the same shape.

| Option | Type | Default | Description |
|---|---|---|---|
| `apiKey` | `string` | required | Your `fd_pub_` publishable key. Not a secret key: this ends up in client-side HTML, same as the manual script tag. |
| `publishableKey` | `string` | - | Alias for `apiKey`. Use whichever name reads better in your config. |
| `environment` | `string` | - | `"production"`, `"staging"`, `"development"`. Maps to `data-env` on the injected tag. |
| `debug` | `boolean` | `false` | Verbose console logging from the SDK. Maps to `data-debug`. |
| `cookieless` | `boolean` | `false` | Memory-only session IDs instead of cookies. Maps to `data-cookieless`. |
| `sampleRate` | `number` | - | Fraction of sessions to track (0 to 1). Maps to `data-sample`. |
| `dnt` | `boolean` | `false` | Respect the browser's Do Not Track signal. Defaults to `false` (DNT is ignored) because Flusterduck captures no PII and respecting DNT by default silently drops 30-50% of visitors on privacy-focused browsers. Pass `dnt: true` to opt back into honoring it. Maps to `data-dnt`. |
| `scriptSrc` | `string` | jsdelivr-hosted bundle | Override the script source URL. Use this if you're self-hosting the SDK bundle behind your own CDN or under a strict CSP. |
| `enabled` | `boolean` | `true` | Set to `false` to skip injection entirely, for example in a local dev build where you don't want tracking. |

Passing a secret key (`fd_sec_`) in `apiKey` or `publishableKey` logs a console error at build time and the plugin injects nothing. There's no way to leak a secret key into your bundle through this plugin.

## What actually gets injected

Given the Vite config above, the output is a script tag equivalent to:

```html
<script
  src="https://cdn.jsdelivr.net/npm/flusterduck@0/dist/d.global.js"
  data-key="fd_pub_xxxxxxxxxxxx"
  data-dnt="false"
  async
></script>
```

The plugin never injects the same `scriptSrc` twice into one document, so re-running the build or nesting multiple HTML entry points won't double-track visitors.

## Deploy tagging and source maps

These plugins don't do either. If you're looking for `confusion_before`/`confusion_after` on your releases, that's [Deploy Correlation](./deploy-correlation): connect your GitHub repo and Flusterduck detects deploys automatically, or call the `/v1/deploys` API from CI with a `fd_sec_` key. Neither requires a build plugin.

There's no source map upload feature in the SDK or these plugins today. If your build already uploads source maps to a symbolication service (Sentry, Bugsnag, etc.), that's independent of Flusterduck and unaffected by adding this plugin.
