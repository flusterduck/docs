# Quickstart

The CLI does everything. Run it, paste your `fd_pub_` key, deploy.

```bash
npx flusterduck-cli init
```

It detects your framework (Next.js, React, Vue, SvelteKit, Nuxt), installs the right packages, and injects the init call into the correct entry file. Pass `--key fd_pub_xxxx` to skip the prompt entirely.

That's the whole quickstart for most apps. The rest of this page is for manual setup and verification.

---

## Manual setup

### Script tag

No build step: paste one tag before the closing `</body>`.

```html
<script src="https://flusterduck.com/d.js" data-key="fd_pub_xxxxxxxxxxxx" data-dnt="false" async></script>
```

The tag is `async` so a slow or unreachable CDN can never block the host page's parse, paint, or `DOMContentLoaded`. The runtime itself is fire-and-forget (`sendBeacon`/`fetch` with `keepalive`) with `init()` wrapped in a try/catch, so it can never throw into your page either. If you need zero third-party runtime dependency, or run under a CSP that forbids third-party script hosts, use the npm path below instead.

### npm

```bash
pnpm add flusterduck
```

Call `init()` once at app startup, before any user interaction:

```ts
import { init } from 'flusterduck'

init({ key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY! })
```

The SDK starts detecting behavioral signals from that point. No other configuration required to start collecting.

If you're using a framework, there's a wrapper that handles lifecycle automatically. See [Framework guides](./frameworks) for Next.js, React, Vue, SvelteKit, and Nuxt.

## Check the connection

Open your app and click around for 30 seconds. Hit a form, click some buttons. Signals appear in the dashboard under **Live signals** within a few seconds of being emitted.

To confirm the connection without deploying, fire a test signal from the browser console:

```ts
import { signal } from 'flusterduck'
signal('dead_click', { selector: 'button[type="submit"]' })
```

If it shows up in Live signals, you're live.

One thing that trips people up: the SDK only runs client-side. If you're server-rendering and checking for signals immediately, open the page in a browser first.

## Your key

`fd_pub_` keys are safe in client-side code. They can only emit signals. They can't read your data or change anything.

```bash
# Next.js
NEXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx

# Vite / Vue / SvelteKit
VITE_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx

# Nuxt
NUXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

`fd_sec_` and `fd_mcp_` keys are different. Both are server-side only. Secret keys can read your site data through the query API; MCP keys can read through the MCP bridge and may be granted additional tool scopes. Keep both out of the browser.

## What you'll see first

Within a few sessions of real traffic, you'll have confusion scores on your tracked pages. The first issues typically surface within 24-48 hours, once the scoring engine has enough signal clusters.

The signals you'll see most in early data: `rage_click`, `dead_click`, and `form_abandonment`. They're the clearest indicators of friction and tend to accumulate quickly on any app with real users. No app has ever had zero.

## Where to go from here

- [Signals](./signals): what each of the 128 signal types means and when to act on it
- [SDK reference](./sdk): every `init()` option and method, with guidance on what actually matters
- [Alerts](./alerts): get notified when a deploy breaks something before your users tell you
- [MCP setup](./mcp): ask your AI assistant about friction data directly instead of checking a dashboard
