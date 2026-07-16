# Build Plugins

The Vite and webpack plugins handle deploy tagging and source map upload at build time. You don't need them to use Flusterduck. Add them when you want automatic deploy recording from your build pipeline or when you want stack traces in issue evidence to resolve to original source.

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
      secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
    }),
  ],
})
```

## webpack

```bash
pnpm add -D flusterduck-webpack-plugin
```

```js
// webpack.config.js
const { FlusterduckPlugin } = require('flusterduck-webpack-plugin')

module.exports = {
  plugins: [
    new FlusterduckPlugin({
      secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
    }),
  ],
}
```

Both plugins accept the same options. The examples below use Vite syntax, but the options are identical for webpack.

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `secretKey` | `string` | required | Your `fd_sec_` secret key. Never expose this client-side. |
| `release` | `string` | git SHA | Version string attached to the deploy record. Defaults to `git rev-parse HEAD`. |
| `environment` | `string` | `NODE_ENV` | `"production"`, `"staging"`, `"development"` |
| `sourceMaps` | `boolean` | `true` | Upload source maps after build. |
| `deleteSourceMaps` | `boolean` | `false` | Delete uploaded source maps from the output directory after upload. |
| `include` | `string[]` | `["dist"]` | Directories to scan for source map files. |
| `dryRun` | `boolean` | `false` | Run the plugin without uploading or recording a deploy. Useful for verifying config. |

## Deploy recording

The plugin records a deploy automatically when the build completes. You don't need the CI curl command from [Deploy Correlation](./deploy-correlation) if you're using the build plugin. Use one or the other, not both.

```ts
flusterduck({
  secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
  release: process.env.VITE_APP_VERSION || 'unknown',
  environment: 'production',
})
```

The plugin POSTs to `/v1/deploys` at the end of the build with `version` set to `release` and `environment` set accordingly. `confusion_before` is captured at that moment.

## Source maps

When `sourceMaps: true` (the default), the plugin uploads `.map` files after the build. Issue evidence in the dashboard will show original file names and line numbers instead of minified bundle references.

Source maps are uploaded to Flusterduck's servers and stored against the `release` version string. They're used only to resolve stack traces in issue evidence. They aren't accessible from the client.

### Keeping source maps off your CDN

Set `deleteSourceMaps: true` to remove map files from the output directory after upload. They'll be used for stack trace resolution but won't be served to browsers or end up in your CDN cache.

```ts
flusterduck({
  secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
  sourceMaps: true,
  deleteSourceMaps: true,
})
```

### Including only specific directories

```ts
flusterduck({
  secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
  include: ['dist/assets'],
})
```

## Environment variable setup

Store `FLUSTERDUCK_SECRET_KEY` in your build environment, not in your codebase. The secret key should never appear in client bundles.

For local development, add it to `.env.local` (which your `.gitignore` should exclude):

```bash
# .env.local
FLUSTERDUCK_SECRET_KEY=fd_sec_xxxxxxxxxxxx
```

In CI, add it as an environment secret:
- GitHub Actions: Settings > Secrets > Actions > New repository secret
- Vercel: Project Settings > Environment Variables
- Netlify: Site settings > Build > Environment variables

## Dry run

Verify plugin configuration without uploading anything:

```ts
flusterduck({
  secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
  dryRun: process.env.NODE_ENV !== 'production',
})
```

With `dryRun: true`, the plugin logs what it would upload and what deploy record it would create, then exits without sending any requests. Useful for testing your config in staging before enabling in production.

## Next.js

Next.js uses webpack under the hood. Use the webpack plugin in `next.config.js`:

```js
// next.config.js
const { FlusterduckPlugin } = require('flusterduck-webpack-plugin')

/** @type {import('next').NextConfig} */
const nextConfig = {
  webpack: (config, { isServer, buildId }) => {
    if (!isServer) {
      config.plugins.push(
        new FlusterduckPlugin({
          secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
          release: buildId,
          environment: process.env.NODE_ENV,
          deleteSourceMaps: true,
        })
      )
    }
    return config
  },
}

module.exports = nextConfig
```

Scope it to `!isServer` to avoid running the plugin twice per build (Next.js runs separate webpack compilations for client and server).

Use `buildId` as the release version. Next.js generates a stable, unique build ID per deployment.
