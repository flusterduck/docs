# flusterduck-cli

The command line for Flusterduck: install the SDK, read scores and issues, manage issues, and record deploys, from your terminal or CI.

## Install

```bash
npx flusterduck-cli init
```

No global install is required. Run it from your project root with `npx`.

## Setup

```bash
# Detect framework, sign in with your browser, pick or create a site,
# install packages, inject init, then verify the first event arrives.
npx flusterduck-cli init

# Provide your publishable key non-interactively (CI, scripts).
npx flusterduck-cli init --key fd_pub_xxxxxxxxxxxx
```

Confirm data is flowing anytime:

```bash
npx flusterduck-cli status --key fd_pub_xxxxxxxxxxxx --wait
```

The CLI inspects your `package.json` and project files, picks the matching wrapper (React, Next.js, Vue, Svelte, Nuxt, or the core SDK), installs it with your package manager, and wires up initialization in the correct entry point.

## Sign in once

The binary answers to both `flusterduck` and `duck`. Run `duck login` to verify and save your `fd_sec_` key locally (0600 file, readable only by you). After that, no command needs `--key`, and a site-scoped key makes `--site` optional too. `duck logout` removes it. Explicit `--key` / `FLUSTERDUCK_SECRET_KEY` always take precedence.

## Read

Add `--json` to any command for machine-readable output.

```bash
duck scores            # site remembered from login
duck scores --site <site_id>   # or explicit
npx flusterduck-cli issues --site <site_id> --status open --limit 20
npx flusterduck-cli insights --site <site_id> --days 30
```

## Manage

```bash
npx flusterduck-cli issue resolve <issue_id> --note "Fixed in #142"
npx flusterduck-cli issue ignore <issue_id> --note "Third-party widget"
npx flusterduck-cli issue reopen <issue_id>
npx flusterduck-cli issue start <issue_id>   # moves it to in progress
```

## Record deploys

```bash
npx flusterduck-cli deploy notify --site <site_id>
```

Run it from CI after each production deploy. Commit hash, author, and PR number are auto-detected on GitHub Actions, Vercel, GitLab, and Bitbucket; override with `--commit`, `--message`, `--author`, `--env`. Flusterduck captures confusion before and after the deploy and verifies whether your fixes actually reduced friction. See [Deploy correlation](./deploy-correlation).

## Links

Published on npm as `flusterduck-cli`, with `@flusterduck/cli` as a supported scoped alias (same versions, same commands). Install pulls the latest published version. Full command reference: [CLI](./cli).
