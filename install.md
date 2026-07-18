# Installation

## Script tag

The fastest way to install: paste one tag before the closing `</body>`, no build step required.

```html
<script src="https://flusterduck.com/d.js" data-key="fd_pub_xxxxxxxxxxxx" data-dnt="false" async></script>
```

Replace `fd_pub_xxxxxxxxxxxx` with your publishable key from Settings > API Keys. Keep `data-dnt="false"`: the SDK captures no PII (element labels are redacted before they leave the browser), no form values, and the default Do Not Track behavior silently drops 30-50% of visitors (Firefox, Brave, privacy extensions) for no privacy benefit here.

The bootstrap reads its config from `document.currentScript`, and falls back to `document.querySelector('script[data-key^="fd_pub_"]')` if that's unset, so injecting the tag dynamically (`document.createElement('script')`, tag managers, etc.) works too, as long as `async` and `data-key` are set.

### Loading & performance

The tag is `async`, not `defer`: a slow or unreachable CDN can never block the host page's parse, paint, or `DOMContentLoaded`. Once it loads, the SDK is fire-and-forget. Events are sent with `navigator.sendBeacon` or a `fetch` with `keepalive: true`, and `init()` runs inside a try/catch so an SDK failure can never throw into your page.

Want zero third-party runtime dependency, or run under a strict CSP that forbids third-party script hosts? Install via npm instead (below) and self-host the bundle.

### Verifying locally

Flusterduck treats non-user traffic as noise: events sent from `localhost` (or other dev hosts) and events from automated browsers are acknowledged but never stored, so a developer smoke-test (thanks Claude) can't burn your session quota, skew scores, or create phantom issues. To see events actually land while developing, add `data-env="development"` to the tag (or pass `environment: 'development'` to `init`), which stands those filters down and tags the traffic as development. Accepted names: `development`, `dev`, `test`, `testing`, `staging`, `local`. Remove the attribute before shipping. The [test-suite CLI](/testing) walks the whole verification for you.

## Packages

Install the core SDK plus any framework wrapper that matches your stack.

### Core SDK

```bash
pnpm add flusterduck
```

Required for every project. The framework wrappers depend on it.

### Framework wrappers

Pick the one that matches your stack. Each command installs the core SDK alongside the wrapper.

#### Next.js

Works with both the App Router and the Pages Router.

```bash
pnpm add flusterduck @flusterduck/next
```

#### React

For Vite, CRA, or any other React app.

```bash
pnpm add flusterduck @flusterduck/react
```

#### Vue 3

```bash
pnpm add flusterduck @flusterduck/vue
```

#### SvelteKit

```bash
pnpm add flusterduck @flusterduck/svelte
```

#### Nuxt 3

```bash
pnpm add flusterduck @flusterduck/nuxt
```

Not using a framework wrapper? Initialize the SDK directly with `init()`. It works in any browser environment.

## Publishable key

Your publishable key starts with `fd_pub_`. Find it in the dashboard under Settings > API Keys.

Set it as an environment variable:

```bash
# Next.js
NEXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx

# Vite / Vue / SvelteKit
VITE_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx

# Nuxt
NUXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

The `fd_pub_` key is safe in client-side code. It can only send signals. It can't read data or modify your account.

## What each package does

| Package | What it does |
|---|---|
| `flusterduck` | Core browser SDK. On-device signal detection/classification and event buffering. |
| `@flusterduck/next` | `FlusterduckScript` component + `useFlusterduck` hook for Next.js App Router and Pages Router. |
| `@flusterduck/react` | `FlusterduckProvider` component + `useFlusterduck` hook. |
| `@flusterduck/vue` | Vue 3 plugin + `useFlusterduck` composable. |
| `@flusterduck/svelte` | SvelteKit module + Svelte store. |
| `@flusterduck/nuxt` | Nuxt 3 module. Auto-initializes on the client. |

## Using the CLI instead

The CLI handles installation and code injection in one command:

```bash
npx flusterduck-cli init
```

See [CLI reference](./cli) for options.

## Requirements

- Browser environment (the SDK doesn't run server-side)
- HTTPS in production
- A site and publishable key from the dashboard
