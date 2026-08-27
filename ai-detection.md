# AI detection

AI detection puts an AI analyst on your site. Each night it reads a compressed summary of the day's real sessions, then chooses what to investigate. It can pull the underlying sessions, page context, deploy timing, cursor movement, and any valid episode receipts before deciding whether the evidence supports an issue.

It is on by default for every site. Built-in pattern detection continues to power real-time signals, scores, and alerts. If the AI analyst cannot run, unreviewed fallback findings are held instead of being filed as customer issues. You can opt a site out any time under Settings, then AI.

## What it does differently

Built-in detection recognizes friction patterns it has names for: rage clicks, dead clicks, abandoned forms. The AI analyst is not limited to named patterns. It can connect a checkout struggle and an upgrade-flow struggle into one underlying issue, notice sessions that quietly left the money path without tripping any detector, and use a page's actual content to tell exploration from confusion.

Every issue it files carries receipts. The analyst must open and cite at least two sessions, and Flusterduck verifies those citations against your data before anything is written. When a detector signal informs the finding, the service also checks that the signal occurred in those sessions and that the analyst read its complete interpretation and verification guidance. Claims that do not check out are rejected. Time-based claims need episode receipts from the same cited detector in at least two cited sessions and must use wording the receipts can directly support; otherwise the analyst must use a narrower, non-temporal observation or abstain. One angry session never becomes an issue by itself, and severity is bounded by how many users the evidence actually shows.

## What it can see on a page

For a visible-state question, the analyst can request a fresh logged-out image of an exact public page it is already investigating, with a hard limit of four visual checks per investigation. A separate read-only visual reviewer sees the pixels. If a small detail is unclear, it can request a bounded higher-resolution crop and inspect that close-up without changing the page's layout or browser zoom. It cannot file issues or save investigation memory. The issue-writing analyst receives only a small set of structured visible-state facts and crop coordinates. Raw image data never enters the saved investigation transcript.

When the current page behavior matters, the analyst can also request a fixed logged-out browser check at either 375 by 812 or 1366 by 900. One fresh Chromium page loads the exact route and captures one viewport at the top or a requested vertical position. It can then perform a bounded scroll, hover, and forward-and-backward Tab check. It returns constrained visual facts plus server-measured runtime failures, page vitals, request counts, numeric viewport and scroll samples, hover completion, focus traces, and basic form and primary-action facts. It does not run Flusterduck detectors. It cannot click, type, submit, inspect event handlers, sign in, or reproduce arbitrary touch, locale, extension, or visitor state. A bounded trace cannot prove what happens outside the tested steps. If a security boundary changes the page before capture or the browser misses the requested position, the image is withheld.

A screenshot can confirm visible facts such as a covered footer, wrapped text, or a blurred result. A close-up proves only what appears inside its selected rectangle. The fixed browser check can report what its own bounded scroll, hover, and Tab walk measured. None of these checks proves why a real user clicked, which code caused a problem, or behavior outside the tested viewport and steps. Customer-uploaded images for authenticated pages are never sent to the detecting model.

## Optional code evidence

Code evidence is generally available as an optional per-site connection, and you choose how much of it Flusterduck may read.

**Link only** connects the site to one exact GitHub repository without reading source. **Read and use for issues** is the recommended setting: it lets the analyst search that repository and read up to three source files during one investigation, so an issue can name the file and line behind it. A third level, reads that file issues found in the code itself, is held back until the private precision canary passes.

Every source read is pinned to the commit that was live while the sessions happened: the most recent deploy at or before the end of the investigated window, whether that shipped last night or last month. Flusterduck resolves abbreviated SHAs through GitHub before reading. Connecting a repository is usually enough to get this, because Flusterduck watches it for deploys. With no recorded deploy at all, the analyst may inspect the current default-branch tip, but the receipt says `pinned=false` and the result cannot stand alone as proof for a code-found issue.

The code tools return server-made receipts with the repository, full SHA, file, line number, and content hash. The analyst may see a short source excerpt for one turn. Source text and diff patches are removed before the investigation transcript is saved. A literal search that cannot account for generated class names, delegated handlers, feature flags, or runtime state returns inconclusive instead of turning uncertainty into a public issue.

When a receipt supports a session-backed issue, the issue keeps that receipt id. Owners and admins can open the code location on issue detail. Flusterduck fetches the file again from the recorded commit, checks its content hash, and shows the recorded line with a small amount of surrounding code. The source is fetched live and is not added to the issue record. If GitHub access, the site mapping, the commit, or the hash no longer matches, the viewer shows no code.

A complete no-match can mark one proposed JavaScript-error source diagnosis false. The searched error must be an exact error captured in the cited sessions, the repository search must read at least one source file without a failed or truncated read, and every original issue session must be reopened. A repeated hard browser failure always wins over missing source text. Existing issues qualify only while they are untouched open AI findings. An assignment, status action, freeze, note, re-scan, Autofix request, or tracker action protects the issue from automatic dismissal.

Only the detecting AI can create a false-by-code verdict. The database independently rechecks the active investigation, organization, site, mapped repository, deployed commit, receipt, and complete-search facts before recording it. A person can ignore or reopen an issue, but cannot label an issue false by code. The issue detail page names the AI verdict and how many files were searched. Repository and commit coordinates remain in the private proof ledger, while owners and admins can open supporting source locations through the protected inline viewer. The cited sessions stay on the record.

Code-only filing is not active yet. Flusterduck records no-matches from selector and error-string searches, then manually checks a fixed sample on at least two sites. The gate requires enough decided audits from both search tools and a false-absence rate at or below the configured ceiling. Until that real-world gate passes, code can support or narrowly disprove a session-backed diagnosis, but code alone cannot create an issue.

## The analyst remembers your site

The analyst keeps a working model of each site: what the product is, its money paths, its weak points, and a ledger of hypotheses it is watching. A pattern too thin to file tonight accumulates evidence across nights and gets filed when it clears the bar. Issues it files again update in place rather than duplicating.

## Run history and running on demand

Settings, then AI shows every investigation: when it ran, how many steps it took, and how many issues it filed. Owners and admins also get a Run now button there, so you can watch a run land right after a deploy or a traffic spike instead of waiting for tonight. On-demand runs use the same allowance and the same verification bar as nightly ones.

## What happens when the allowance runs out

AI detection runs on Flusterduck's infrastructure inside your AI allowance. There is nothing to configure and no per-run bill. On a paid plan, the allowance resets each calendar month. If it runs out early, new AI-reviewed issues wait until the next allowance period. Built-in signals, real-time scores, alerts, and deploy verification are unaffected; they never depend on the nightly run.

The three-day trial has its own fixed allowance. It lasts for the whole trial, even when the trial crosses into a new month. When it is used, unreviewed fallback findings remain held rather than appearing as customer issues.

## What it never does

- It never creates a visitor-session recording. The analyst reads structured behavioral events and, when useful, bounded rendered text and fresh logged-out screenshots fetched separately from the configured public page. It gets no replay, keystrokes, visitor-entered form values, or visitor-typed text. A public screenshot can contain anything visibly rendered to a logged-out visitor, including public default or prefilled values.
- It never files an issue from one session. Every issue names at least two examined sessions behind it.
- It never guesses a repository or falls back to another GitHub connection. Missing or revoked source access keeps the investigation session-only.
- It never clicks, types, submits, signs in, changes site data, or changes your code. Its browser check is limited to loading, scrolling, hovering, and walking Tab order. Investigating and maintaining issue evidence is the whole job; Autofix stays a separate, separately-permissioned feature.
