# flusterduck-cli

The command line for Flusterduck: install the SDK, read scores and issues, manage issues, and record deploys, from your terminal or CI.

## Install

```bash
npm install -g flusterduck-cli
```

You get two binaries, `flusterduck` and `duck`, same program either way. For a one-off run without installing anything (CI, a machine you don't own), `npx flusterduck-cli <command>` works for every command.

## Setup

```bash
# Detect framework, sign in with your browser, pick or create a site,
# install packages, inject init, then verify the first event arrives.
flusterduck init

# Provide your publishable key non-interactively (CI, scripts).
npx flusterduck-cli init --key fd_pub_xxxxxxxxxxxx
```

Confirm data is flowing anytime:

```bash
flusterduck status --key fd_pub_xxxxxxxxxxxx --wait
```

The CLI inspects your `package.json` and project files, picks the matching wrapper (React, Next.js, Vue, Svelte, Nuxt, or the core SDK), installs it with your package manager, and wires up initialization in the correct entry point.

## Sign in once

The binary answers to both `flusterduck` and `duck`. Run `duck login` to verify and save your `fd_sec_` key locally (0600 file, readable only by you). After that, no command needs `--key`, and a site-scoped key makes `--site` optional too. `duck logout` removes it. Explicit `--key` / `FLUSTERDUCK_SECRET_KEY` always take precedence.

## Read

Add `--json` to any command for machine-readable output, or `--md` for markdown.

```bash
duck                   # bare: site health, top issues, last deploy verdict
duck scores            # site remembered from login or the git remote
duck scores --site <site_id>   # or explicit
duck issues --status open --limit 20
duck insights --days 30
duck sites             # every site in the org (org-wide keys)
duck top               # live: pages by confusion, refreshing until q
duck context           # the whole picture as markdown, for agent sessions
```

## Inspect and manage

```bash
duck fix                      # worst open issue: diagnosis, fix, marked in progress
duck fix <issue_id> --prompt  # agent-ready markdown brief, pipe it anywhere
duck issue show <issue_id>    # full write-up, recommended fix, evidence, verifications
duck issue resolve <issue_id> --note "Fixed in #142"
duck issue ignore <issue_id> --note "Third-party widget"
duck issue reopen <issue_id>
duck issue start <issue_id>   # moves it to in progress
duck deploy watch             # sit on the newest deploy until the verdict lands
```

## Record deploys

```bash
npx flusterduck-cli deploy notify --site <site_id> --key fd_sec_xxxx
```

Run it from CI after each production deploy. Commit hash, author, and PR number are auto-detected on GitHub Actions, Vercel, GitLab, and Bitbucket; override with `--commit`, `--message`, `--author`, `--env`. Flusterduck captures confusion before and after the deploy and verifies whether your fixes actually reduced friction. See [Deploy correlation](./deploy-correlation).

## Links

Published on npm as `flusterduck-cli`, with `@flusterduck/cli` as a supported scoped alias (same versions, same commands). Install pulls the latest published version. Full command reference: [CLI](./cli).
