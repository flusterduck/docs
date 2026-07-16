# flusterduck-webpack-plugin

The webpack plugin. It handles deploy tagging and source map upload at build time so issue evidence resolves to your original source.

## Install

```bash
npm install -D flusterduck-webpack-plugin
```

## Usage

```js
// webpack.config.js
const { FlusterduckPlugin } = require('flusterduck-webpack-plugin')

module.exports = {
  plugins: [
    new FlusterduckPlugin({
      secretKey: process.env.FLUSTERDUCK_SECRET_KEY,
      release: process.env.APP_VERSION,
      environment: 'production',
    }),
  ],
}
```

The plugin records a deploy when the build completes and uploads source maps so stack traces in issue evidence map back to original file names and line numbers. Keep `FLUSTERDUCK_SECRET_KEY` in your build environment, never in client code.

## Links

Published on npm as `flusterduck-webpack-plugin`. Install pulls the latest published version.
