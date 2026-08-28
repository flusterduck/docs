# CLI Reference

```bash
npm install -g flusterduck-cli
```

That puts two binaries on your PATH, `flusterduck` and the shorter `duck`. They're the same program; type whichever you like. If you'd rather not install anything, every command also runs through `npx flusterduck-cli <command>`, which pulls the latest version each time. The examples below use the installed binary.

## Bare `duck`

Run `duck` with no command and you get your product's health, the way `git status` gives you your tree:

```
Moda · watching · score 20.9 ↑ · 49 sessions · 328 events · last 24h

Needs you · 5 open
  HIGH    State-selector combobox on /states silently eats clicks
          duck fix fb2a80e3-... · /states · impact 75 · 26d ago

Last deploy 4h ago (a1b2c3d) · 2 issues checked · confusion 44 → 41
```

One screen: is data flowing, what needs a human, did the last deploy help. Every id on it is pasteable into `duck fix` or `duck issue show`.

Which site? An explicit `--site` wins, then the site remembered by `duck login`, then the git remote of the directory you're standing in: if your org has connected the repository to a site, `duck` inside that repo resolves to that site with zero configuration (the match is cached locally after the first lookup).

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

The CLI is built to be driven by agents as much as by people. `--json` returns the full API payload for every command, read commands also take `--md` for clean markdown, exit codes are honest (0 on success, 1 on any failure, with the error on stderr), and piped output drops colors and wrapping so plain-text parsing works too. The short version of the agent loop: `duck context` to get oriented, `duck fix --prompt` for a complete fix brief, `duck issue resolve <id> --note "..."` when done, `duck deploy watch` for the measured verdict.

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

### `fix`

The whole resolution loop in one command. With no argument it takes the highest-impact open issue; with an id it takes that one. Either way it prints the diagnosis and the recommended fix, marks the issue in progress so your team sees it's claimed, and shows the pinned-commit code locations when code evidence exists.

```bash
duck fix                     # worst open issue
duck fix 7f2c9d4a-...        # a specific one
duck fix 7f2c9d4a-... --prompt | claude   # hand the whole brief to a coding agent
```

`--prompt` renders the same material as a markdown brief for a coding agent: the behavioral evidence, the recommended direction, the code pointers, and the exact commands to run when done (`duck issue resolve`, `duck deploy watch`), so the agent closes the loop itself.

### `sites` and `use`

```bash
duck sites          # every site in the org: state, score, trend, sparkline
duck use moda       # set the default site by name or id
```

`sites` needs an org-wide key; a site-scoped key already knows its site.

### `top`

A live view: pages sorted by confusion score, refreshing every 5 seconds, `q` to quit. Watch a launch, a deploy, or a traffic spike in real time.

```bash
duck top
```

### `context`

Everything about the site's UX health as one markdown document: worst pages, open issues with recommendations, active alerts, recent deploys. Built for priming an agent session in one command.

```bash
duck context > ux-context.md
duck context | claude "read this, then fix the worst issue"
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

### `deploy watch`

After you ship, watch until the verdict lands: Flusterduck captures confusion before the deploy, measures it after, and this command sits on the newest deploy (or `--commit <sha>`) until the before/after numbers resolve. Green means the deploy helped; red means it made friction worse, and the command exits non-zero so CI can catch a regression.

```bash
flusterduck deploy watch
flusterduck deploy watch --commit "$(git rev-parse HEAD)" --timeout 45
```

## Troubleshooting

**"Could not automatically inject code"**: the CLI found the entry file but couldn't parse it (unusual syntax, non-standard structure). Wire the init call manually following the [quickstart](./quickstart).

**"Flusterduck is already configured"**: the SDK was already detected in the target file. Nothing changed.

**"Failed to install dependencies"**: package installation failed. The CLI prints the exact install command to run yourself.

If none of these come up, the install worked. That outcome is more common than a troubleshooting section makes it sound.
