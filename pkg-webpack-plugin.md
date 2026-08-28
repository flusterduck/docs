# flusterduck-webpack-plugin

The webpack plugin. It injects the Flusterduck script tag into your built HTML automatically, so you don't hand-edit the template.

## Install

```bash
npm install -D flusterduck-webpack-plugin
```

## Usage

```js
// webpack.config.js
const FlusterduckWebpackPlugin = require('flusterduck-webpack-plugin')

module.exports = {
  plugins: [
    new FlusterduckWebpackPlugin({
      apiKey: 'fd_pub_xxxxxxxxxxxx',
      environment: 'production',
    }),
  ],
}
```

Works with or without `html-webpack-plugin`: when it's installed, the plugin taps its asset-tag hook, otherwise it writes the script tag directly into emitted HTML files. Pass a publishable key only (`fd_pub_`): a secret key is rejected and logged as an error instead of injected. See the [build plugins guide](./build-plugins) for the full option list.

## Links

Published on npm as `flusterduck-webpack-plugin`. Install pulls the latest published version.
