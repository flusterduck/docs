# CLI Reference

```bash
npx flusterduck-cli init
```

Detects your framework, installs the right packages, and injects the init call into the correct entry file. With no `--key`, it offers to **sign you in with your browser**: a tab opens, you pick or create a site on flusterduck.com, and the key comes back to the terminal automatically: no dashboard round-trip, nothing to copy. (Creating a new site returns its key directly; an existing site asks you to paste the `fd_pub_` key already in its script tag.)

At the end, init offers to **verify the install live**: start your app, click around, and it watches for the first event to arrive, so a wrong-but-well-formed key can never fail silently.

You don't need to install the CLI globally. `npx` pulls the latest version each time.

## Options

### `--key fd_pub_xxxx`

Skip the key prompt:

```bash
npx flusterduck-cli init --key fd_pub_xxxxxxxxxxxx
```

Useful in CI pipelines, onboarding scripts, and anywhere you want a non-interactive run.

### `--skip-install`

Inject the setup code without installing packages. Use this if you've already installed the SDK separately or if your package manager workflow requires a separate step.

```bash
npx flusterduck-cli init --skip-install --key fd_pub_xxxxxxxxxxxx
```

## Framework detection

The CLI reads your `package.json` and project structure to determine your framework, then installs and injects accordingly:

| Detected | Packages installed | File injected |
|---|---|---|
| Next.js | `flusterduck` `@flusterduck/next` | `app/layout.tsx` (App Router) or `pages/_app.tsx` (Pages Router) |
| SvelteKit | `flusterduck` `@flusterduck/svelte` | `src/routes/+layout.svelte` |
| Nuxt | `flusterduck` `@flusterduck/nuxt` | `nuxt.config.ts` |
| React (Vite / CRA) | `flusterduck` | `src/main.tsx` or `src/index.tsx` |
| Generic / unknown | `flusterduck` | No injection |

If the CLI can't determine your framework, it installs the core SDK and exits with a code snippet to add manually.

## What gets injected

The injected code is a single import and a single init call. For Next.js App Router:

```tsx
// app/layout.tsx
import { FlusterduckScript } from '@flusterduck/next'

// Added inside your RootLayout body:
<FlusterduckScript apiKey={process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!} />
```

For vanilla React:

```tsx
// src/main.tsx
import { init } from 'flusterduck'
init({ key: process.env.VITE_FLUSTERDUCK_KEY! })
```

The CLI doesn't rewrite existing imports or touch your component tree. If Flusterduck is already present in the target file, it skips injection and tells you.

## Environment variable

The injected code references an environment variable, not a literal key. After running the CLI, set the variable:

```bash
# .env.local (Next.js)
NEXT_PUBLIC_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx

# .env (Vite / SvelteKit)
VITE_FLUSTERDUCK_KEY=fd_pub_xxxxxxxxxxxx
```

The CLI output tells you the exact variable name for your framework.

## Reading your data

Beyond `init`, the CLI reads and manages friction data straight from your terminal. The binary answers to two names (`flusterduck` and `duck`), so `duck status` and `duck issues` both work.

Sign in once and every command authenticates automatically:

```bash
duck login    # browser sign-in (mints a key, nothing to copy) or paste with hidden input
duck logout   # removes the saved key
```

The browser option opens flusterduck.com, you pick the site this terminal should manage, and a freshly minted key is sent straight to the CLI on your machine (never through a URL). Either way `login` verifies the key against the API before saving, stores it in a `0600` file under your config directory (`~/.config/flusterduck/credentials.json`), and remembers the key's site, so a site-scoped key makes `--site` optional everywhere. Precedence when both exist: `--key` flag, then `FLUSTERDUCK_SECRET_KEY`, then the saved login. Override the API base with `--api` or `FLUSTERDUCK_API_URL`. Every command takes `--json` for raw output.

```bash
duck login
duck issues --status open     # no --key, no --site
duck deploy notify            # same
```

### `scores`

Per-page confusion scores for a site, ranked by friction.

```bash
flusterduck scores --site <site_id>
```

### `issues`

UX issues detected on a site. Filter with `--status` (`open`, `triaged`, `in_progress`, `resolved`, `verified`, `ignored`, `regressed`) and cap with `--limit`.

```bash
flusterduck issues --site <site_id> --status open --limit 20
```

### `insights`

The confused-vs-calm conversion gap: how much less confused sessions convert than calm ones, the pages and traffic sources hit hardest, and the ranked insights. Pass `--days` (1-90, default 7) to set the window.

```bash
flusterduck insights --site <site_id> --days 30
```

This needs a conversion event wired so Flusterduck knows what "success" is. See [Conversion trigger](./conversion-trigger).

### `issue`

Manage a single issue from the terminal. Verbs: `resolve`, `ignore`, `reopen`, `start` (moves it to in progress). Attach context with `--note`.

```bash
flusterduck issue resolve 7f2c9d4a-... --note "Fixed in #142"
flusterduck issue ignore 7f2c9d4a-... --note "Third-party widget, can't fix"
```

### `status`

Is Flusterduck receiving data for a key? Authenticates with the **publishable** key (`--key fd_pub_...` or `FLUSTERDUCK_PUBLISHABLE_KEY`), so it works before you've ever opened the dashboard. `--wait` polls for up to 3 minutes until the first event lands.

```bash
flusterduck status --key fd_pub_xxxxxxxxxxxx --wait
```

### `deploy notify`

Record a deploy so Flusterduck captures before/after confusion and verifies your fixes. Run it from CI after each production deploy. Commit hash, author, and PR number are auto-detected on GitHub Actions, Vercel, GitLab, and Bitbucket.

```bash
flusterduck deploy notify --site <site_id>
# or outside CI:
flusterduck deploy notify --site <site_id> --commit "$(git rev-parse HEAD)" --env production
```

Same thing, no npx: POST to [`/v1/deploys`](./deploy-correlation) directly.

## Troubleshooting

**"Could not automatically inject code"**: the CLI found the entry file but couldn't parse it (unusual syntax, non-standard structure). Wire the init call manually following the [quickstart](./quickstart).

**"Flusterduck is already configured"**: the SDK was already detected in the target file. Nothing changed.

**"Failed to install dependencies"**: package installation failed. The CLI prints the exact install command to run yourself.
