# QA

QA is a test browser that uses your site the way a first-time visitor would, on a phone and on desktop. It finds problems your visitors haven't reported and some they haven't hit yet: a button that does nothing, a result that lands off screen, text too faint to read, a layout that slides sideways on a small phone.

It files a problem only when the browser measured it. Then it measures it twice more in fresh browsers, and a second reviewer checks it before it reaches your issue list. A guess never gets filed.

QA is in testing. It runs for a small group of workspaces first, and the dashboard pages below show up when Experimental features is on in Settings, General.

## What it does on your site

Each run opens your busiest pages and the pages with forms, and tries them as a visitor would: scroll, hover, tab through the controls, fill a field, press the main button, check what changed and where it landed.

## What it never does

These limits are enforced by the browser, not by asking the model nicely:

- No signing in, no payments, no purchases.
- No form submissions that leave the page, no signups, no contact messages.
- No other sites. It stays on your domain.
- Every request that would change something on your server (anything but a plain page or file load) is blocked.
- Anything it types is made up: digits, letters, and example.com addresses. Nothing comes from your visitors.

## When it runs

- **Once after install**, within about 15 minutes of the first events from your site.
- **Weekly**, at a set hour for each site. A week where your site hasn't changed since the last run is skipped, with a full run at least every 30 days.
- **After a deploy**, at most once a day.
- **By hand**, with Run QA now on the QA page or in Settings, AI. Owners and admins only, up to three a day per site, and not within 30 minutes of the last run.

## The allowance

QA has its own monthly allowance, separate from AI detection, diagnosis, Guide and Autofix. The dashboard shows it as a percent used and a run count. When it runs out, QA pauses until the allowance resets on the 1st, and owners and admins get one email saying so. Issues it already filed stay where they are. A trial has its own smaller allowance that doesn't reset.

## Where you see it

- **QA page** (under Fix in the sidebar): every run with what started it, when, how it ended and what it filed. Open a run to see the pages it opened, and for a run that played a visitor flow, each step it took. Step-by-step details are kept for 30 days.
- **Issues**: QA's findings carry a Found by QA badge. Filter the list with Found by QA, or by who found an issue (QA, AI detection, visitor signals, your team).
- **Issue page**: a Found by QA card that says where it was measured, the steps that reproduced it when the run recorded them, and a link to the run.
- **Home**: a QA card with the last run, what it filed and the next run.
- **Settings, AI**: turn QA on or off per site, see when it runs, the allowance, and Run QA now.
- **Email**: the first-issue email and the weekly digest say when QA found an issue.

## Turning it off

Settings, AI, QA. The switch is per site. It stops new runs and the rest of a run in progress. Issues QA filed stay on your board.
