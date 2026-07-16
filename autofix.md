# Autofix

Autofix turns an issue's evidence into a concrete fix: a root cause, the specific change to make, likely places to look, a way to verify it, and a prompt you can paste straight into your coding agent (Claude Code, Cursor). When you connect the Flusterduck GitHub App, Autofix goes further: it researches your repository and opens a **draft pull request** with the change.

## Two modes

**Brief (always available).** Autofix reads the behavioral evidence already on the issue (the same no-PII data used for diagnosis) and proposes a change. You (or your agent) ship it. Flusterduck never sees your repository in this mode.

**Draft PR (GitHub App connected).** With the GitHub App installed, Autofix researches the connected repo, reads the files most likely at fault, plans the smallest safe change, and opens a **draft** pull request. It only edits files it read, never creates files, and never merges. If it isn't confident it can fix the issue safely from what it can see, it falls back to the brief rather than pushing a guess.

The model that plans repo-level fixes is Claude Sonnet.

## Using it

Open any issue and press **Generate fix** on the Autofix card. You get:

- **Root cause**: the specific UX/code cause the evidence supports.
- **The change**: what to change, specific enough to implement.
- **Likely places to look**: best-guess file/component locations (never asserted as fact).
- **How to verify**: a manual check or test to confirm the fix worked.
- **Agent prompt**: a self-contained instruction to paste into your coding agent.
- **Draft PR**: when the GitHub App is connected and the fix is confident, a link to the opened pull request and the files it changed.

Regenerate any time to get a fresh proposal.

## Limits and cost

Autofix is a **count of fixes**, included by plan: Scale (5/mo), Pro (20/mo), Enterprise (unlimited). Grow includes none. The count only decrements on a successful generation, and resets each billing cycle.

Need more than your monthly allowance? Autofix scales with your plan, not à la carte: **upgrade your plan** for more fixes. There are no add-on packs to buy.

Autofix runs on its own AI budget, separate from issue diagnosis and Guide, so heavy autofix use can never starve diagnosis (and vice versa). You never see a dollar meter, just the fix count.

## Safety boundary

The draft-PR boundary is deliberate: a wrong draft PR is a shrug, a wrong auto-merge is an outage. Autofix always opens PRs as drafts and never merges. You review and ship.
