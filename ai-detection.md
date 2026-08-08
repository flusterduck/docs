# AI detection

AI detection puts an AI analyst on your site. Each night it reads a compressed summary of the day's real sessions, investigates whatever looks off, and files issues itself: pulled sessions, page content, deploy timing, and cursor movement all feed the same investigation a good UX analyst would run by hand.

It is on by default for every site. Built-in pattern detection remains as the fallback: if a paid month's AI allowance runs out early, it takes over filing new issues until the next month. You can opt a site out any time under Settings, then AI.

## What it does differently

Built-in detection recognizes friction patterns it has names for: rage clicks, dead clicks, abandoned forms. The AI analyst is not limited to named patterns. It can connect a checkout struggle and an upgrade-flow struggle into one underlying issue, notice sessions that quietly left the money path without tripping any detector, and use a page's actual content to tell exploration from confusion.

Every issue it files carries receipts. The analyst must cite the sessions it examined, and Flusterduck verifies those citations against your data before anything is written. Claims that do not check out are rejected. One angry session never becomes a critical issue: severity is bounded by how many users the evidence actually shows.

## The analyst remembers your site

The analyst keeps a working model of each site: what the product is, its money paths, its weak points, and a ledger of hypotheses it is watching. A pattern too thin to file tonight accumulates evidence across nights and gets filed when it clears the bar. Issues it files again update in place rather than duplicating.

## Run history and running on demand

Settings, then AI shows every investigation: when it ran, how many steps it took, and how many issues it filed. Owners and admins also get a Run now button there, so you can watch a run land right after a deploy or a traffic spike instead of waiting for tonight. On-demand runs use the same allowance and the same verification bar as nightly ones.

## What happens when the allowance runs out

AI detection runs on Flusterduck's infrastructure inside your AI allowance. There is nothing to configure and no per-run bill. On a paid plan, the allowance resets each calendar month. If it runs out early, built-in detection takes over filing new issues until the next month. Real-time scores, alerts, and deploy verification are unaffected either way; they never depend on the nightly run.

The three-day trial has its own fixed allowance. It lasts for the whole trial, even when the trial crosses into a new month. When it is used, built-in detection keeps filing issues for the rest of the trial.

## What it never does

- It never creates a visitor-session recording. The analyst reads structured behavioral events and, when available, bounded content fetched from the configured public page. It gets no replay, keystrokes, form values, or visitor-typed text.
- It never files an uncited issue. Every issue names the sessions behind it.
- It never touches your site or your code. Filing issues is the whole job; Autofix stays a separate, separately-permissioned feature.
