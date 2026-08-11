# AI detection

AI detection puts an AI analyst on your site. Each night it reads a compressed summary of the day's real sessions, then chooses what to investigate. It can pull the underlying sessions, page context, deploy timing, and cursor movement before deciding whether the evidence supports an issue.

It is on by default for every site. Built-in pattern detection remains as the fallback: if a paid month's AI allowance runs out early, it takes over filing new issues until the next month. You can opt a site out any time under Settings, then AI.

## What it does differently

Built-in detection recognizes friction patterns it has names for: rage clicks, dead clicks, abandoned forms. The AI analyst is not limited to named patterns. It can connect a checkout struggle and an upgrade-flow struggle into one underlying issue, notice sessions that quietly left the money path without tripping any detector, and use a page's actual content to tell exploration from confusion.

Every issue it files carries receipts. The analyst must open and cite at least two sessions, and Flusterduck verifies those citations against your data before anything is written. When a detector signal informs the finding, the service also checks that the signal occurred in those sessions and that the analyst read its complete interpretation and verification guidance. Claims that do not check out are rejected. One angry session never becomes an issue by itself, and severity is bounded by how many users the evidence actually shows.

## What it can see on a page

For a visible-state question, the analyst can request a fresh logged-out image of an exact public page it is already investigating, with a hard limit of four visual checks per investigation. A separate read-only visual reviewer sees the pixels. It cannot file issues or save investigation memory. The issue-writing analyst receives only a small set of structured visible-state facts, and raw image data never enters the saved investigation transcript.

When the current page behavior matters, the analyst can also request a fixed logged-out browser check at either 375 by 812 or 1366 by 900. One fresh Chromium page loads the exact route and captures one viewport at the top or a requested vertical position. It can then perform a bounded scroll, hover, and forward-and-backward Tab check. It returns constrained visual facts plus server-measured runtime failures, page vitals, request counts, numeric viewport and scroll samples, hover completion, focus traces, and basic form and primary-action facts. It does not run Flusterduck detectors. It cannot click, type, submit, inspect event handlers, sign in, or reproduce arbitrary touch, locale, extension, or visitor state. A bounded trace cannot prove what happens outside the tested steps. If a security boundary changes the page before capture or the browser misses the requested position, the image is withheld.

A screenshot can confirm visible facts such as a covered footer, wrapped text, or a blurred result. The fixed browser check can report what its own bounded scroll, hover, and Tab walk measured. Neither proves why a real user clicked, which code caused a problem, or behavior outside the tested viewport and steps. Customer-uploaded images for authenticated pages are never sent to the detecting model.

## The analyst remembers your site

The analyst keeps a working model of each site: what the product is, its money paths, its weak points, and a ledger of hypotheses it is watching. A pattern too thin to file tonight accumulates evidence across nights and gets filed when it clears the bar. Issues it files again update in place rather than duplicating.

## Run history and running on demand

Settings, then AI shows every investigation: when it ran, how many steps it took, and how many issues it filed. Owners and admins also get a Run now button there, so you can watch a run land right after a deploy or a traffic spike instead of waiting for tonight. On-demand runs use the same allowance and the same verification bar as nightly ones.

## What happens when the allowance runs out

AI detection runs on Flusterduck's infrastructure inside your AI allowance. There is nothing to configure and no per-run bill. On a paid plan, the allowance resets each calendar month. If it runs out early, built-in detection takes over filing new issues until the next month. Real-time scores, alerts, and deploy verification are unaffected either way; they never depend on the nightly run.

The three-day trial has its own fixed allowance. It lasts for the whole trial, even when the trial crosses into a new month. When it is used, built-in detection keeps filing issues for the rest of the trial.

## What it never does

- It never creates a visitor-session recording. The analyst reads structured behavioral events and, when useful, bounded rendered text and fresh logged-out screenshots fetched separately from the configured public page. It gets no replay, keystrokes, visitor-entered form values, or visitor-typed text. A public screenshot can contain anything visibly rendered to a logged-out visitor, including public default or prefilled values.
- It never files an issue from one session. Every issue names at least two examined sessions behind it.
- It never clicks, types, submits, signs in, changes site data, or changes your code. Its browser check is limited to loading, scrolling, hovering, and walking Tab order. Filing issues is the whole job; Autofix stays a separate, separately-permissioned feature.
