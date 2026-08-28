# CLI Reference

```bash
npm install -g flusterduck-cli
```

That puts two binaries on your PATH, `flusterduck` and the shorter `duck`. They're the same program; type whichever you like. If you'd rather not install anything, every command also runs through `npx flusterduck-cli <command>`, which pulls the latest version each time. The examples below use the installed binary.

```bash
flusterduck init
```

Detects your framework, installs the right packages, and injects the init call into the correct entry file. With no `--key`, it offers to **sign you in with your browser**: a tab opens, you pick or create a site on flusterduck.com, and the key comes back to the terminal automatically: no dashboard round-trip, nothing to copy. (Creating a new site returns its key directly; an existing site asks you to paste the `fd_pub_` key already in its script tag.)

At the end, init offers to **verify the install live**: start your app, click around, and it watches for the first event to arrive, so a wrong-but-well-formed key can never fail silently.

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
flusterduck init --skip-install --key fd_pub_xxxxxxxxxxxx
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
<FlusterduckScript apiKey="fd_pub_xxxxxxxxxxxx" />
```

For vanilla React:

```tsx
// src/main.tsx
import { init } from 'flusterduck'
init({ key: 'fd_pub_xxxxxxxxxxxx' })
```

The key is written as a literal. That's fine: a publishable key is a public identifier, the same value that sits in a script tag on any site using the snippet install. It can only send events, never read data. If you'd rather load it from an environment variable, swap the literal for your framework's public env reference (`process.env.NEXT_PUBLIC_...`, `import.meta.env.VITE_...`) after the CLI runs.

The CLI doesn't rewrite existing imports or touch your component tree. If Flusterduck is already present in the target file, it skips injection and tells you.

## Reading your data

Beyond `init`, the CLI reads and manages friction data straight from your terminal. The binary answers to two names (`flusterduck` and `duck`), so `duck status` and `duck issues` both work.

Sign in once and every command authenticates automatically:

```bash
duck login    # browser sign-in (mints a key, nothing to copy) or paste with hidden input
duck logout   # removes the saved key
```

The browser option opens flusterduck.com, you pick the site this terminal should manage, and a freshly minted key is sent straight to the CLI on your machine (never through a URL). Either way `login` verifies the key against the API before saving, stores it in a `0600` file under your config directory (`~/.config/flusterduck/credentials.json`), and remembers the key's site, so a site-scoped key makes `--site` optional everywhere. Precedence when both exist: `--key` flag, then `FLUSTERDUCK_SECRET_KEY`, then the saved login. Override the API base with `--api` or `FLUSTERDUCK_API_URL`. Every command takes `--json` for raw output.

The CLI is built to be driven by agents as much as by people. `--json` returns the full API payload for every command, exit codes are honest (0 on success, 1 on any failure, with the error on stderr), and piped output drops colors and wrapping so plain-text parsing works too. An agent can run `duck issues --json`, pick an id, read the whole diagnosis with `duck issue show <id> --json`, ship a fix, then `duck issue resolve <id> --note "..."` and `duck deploy notify`, and the next deploy verification closes the loop.

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

UX issues detected on a site. Filter with `--status` (`open`, `triaged`, `in_progress`, `resolved`, `verified`, `ignored`, `regressed`) and cap with `--limit`. Every issue prints its full title and its UUID, which the `issue` verbs below take. In a terminal, long titles wrap to your window; piped output keeps each field group on one full line, so nothing is ever cut off for a script or an agent.

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

Inspect or manage a single issue from the terminal. `show` prints everything the dashboard knows: the full write-up, the recommended fix, evidence session ids, linked tracker issues (GitHub, Linear), and deploy verification verdicts with before/after confusion. The other verbs change status: `resolve`, `ignore`, `reopen`, `start` (moves it to in progress). Attach context with `--note`.

```bash
flusterduck issue show 7f2c9d4a-...
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

If you have connected a repository, you do not need this: Flusterduck already watches it and records deploys on its own. See [deploy correlation](./deploy-correlation). Running both is harmless, they are matched by commit.

```bash
flusterduck deploy notify --site <site_id>
# or outside CI:
flusterduck deploy notify --site <site_id> --commit "$(git rev-parse HEAD)" --env production
```

Same thing, no CLI: POST to [`/v1/deploys`](./deploy-correlation) directly.

## Troubleshooting

**"Could not automatically inject code"**: the CLI found the entry file but couldn't parse it (unusual syntax, non-standard structure). Wire the init call manually following the [quickstart](./quickstart).

**"Flusterduck is already configured"**: the SDK was already detected in the target file. Nothing changed.

**"Failed to install dependencies"**: package installation failed. The CLI prints the exact install command to run yourself.

If none of these come up, the install worked. That outcome is more common than a troubleshooting section makes it sound.
