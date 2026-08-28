# Autofix

Autofix turns an issue's evidence into a proposed fix: the supported diagnosis, the specific change to make, likely places to look, a way to verify it, and a prompt you can paste straight into your coding agent (Claude Code, Cursor). When you connect the Flusterduck GitHub App, Autofix goes further: it researches your repository and opens a **draft pull request** with the change.

## Two modes

**Brief (when an Autofix is available).** Autofix reads the bounded behavioral evidence already on the issue and proposes a change. It receives no form values or user-typed text. You (or your agent) ship the change, and Flusterduck never sees your repository in this mode.

**Draft PR (GitHub App connected).** With the GitHub App installed, Autofix researches the connected repo, reads the files most likely at fault, plans the smallest safe change, and opens a **draft** pull request. It only edits files it read, never creates files, and never merges. If it isn't confident it can fix the issue safely from what it can see, it falls back to the brief rather than pushing a guess.

The model that plans repo-level fixes is Claude Sonnet.

## Using it

Open any issue and press **Generate fix** on the Autofix card. You get:

- **Diagnosis**: the specific UX or code cause when the evidence supports one; otherwise the observed failure only.
- **The change**: what to change, specific enough to implement.
- **Likely places to look**: best-guess file/component locations (never asserted as fact).
- **How to verify**: a manual check or test to confirm the fix worked.
- **Agent prompt**: a self-contained instruction to paste into your coding agent.
- **Draft PR**: when the GitHub App is connected and the fix is confident, a link to the opened pull request and the files it changed.

Generate again while your plan has an Autofix remaining. Each successful generation counts as one fix.

## Limits and allowance

The three-day trial includes **one manual Autofix**. It is one fix for the lifetime of the trial, not one per month. Only a person pressing **Generate fix** can use it; scheduled Autofix Autopilot requires paid billing.

Paid Autofix is a **count of fixes** included by plan: Grow (2/month), Scale (10/month), Pro (30/month), Enterprise (unlimited). The count only decrements on a successful generation and resets monthly.

If the trial Autofix has been used, Billing shows **Start my plan now**. Starting the plan ends the trial and charges the saved card immediately, after you confirm the plan and price there. If a paid plan needs more fixes, upgrade the plan. There are no add-on packs to buy.

Autofix has its own allowance, separate from issue diagnosis and Guide, so heavy Autofix use can never use up either of those allowances. Usage shows the fix count.

## Safety boundary

The draft-PR boundary is deliberate: a wrong draft PR is a shrug, a wrong auto-merge is an outage. Autofix always opens PRs as drafts and never merges. You review and ship.
