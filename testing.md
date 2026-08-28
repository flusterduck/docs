# Testing Your Integration

`@flusterduck/test-suite` is a Playwright-based CLI that drives a real headless Chromium browser through your app and performs the physical actions of a frustrated user: rapid clicking, dead-end clicking, cursor thrashing, scroll bouncing, tab thrashing, hesitating on form fields, and abandoning forms. It confirms the SDK is installed and detectors are wired up before you rely on them in production.

It does not talk to the scoring engine. It performs the actions and tells you whether it found something to click, scroll, or fill. Whether Flusterduck actually detected and stored each signal is something you confirm in the dashboard's Live signals view, or by watching your browser console with `debug: true` on `init()`.

## Install

```bash
pnpm add -D @flusterduck/test-suite
```

Installing gives you the `flusterduck-test` binary. Or run without installing:

```bash
npx -y @flusterduck/test-suite --url http://localhost:3000
```

The package installs Playwright as a dependency. If Chromium isn't already on your machine, the CLI will tell you to run `npx playwright install chromium`.

## Running a simulation

Point it at your dev server:

```bash
npx flusterduck-test --url http://localhost:3000 --key fd_pub_xxxxxxxxxxxx
```

One thing matters here: declare a development environment on the app you're testing. Flusterduck treats automated browsers and local hosts as noise in production. Events sent from `localhost` or a browser that identifies itself as automated (which Playwright does) are acknowledged but never stored, so a verification run can't burn your session quota or create phantom issues. Add `data-env="development"` to your script tag (or `environment: 'development'` in your `init()` call) on the app you point the CLI at, and both filters stand down for that traffic. Accepted names: `development`, `dev`, `test`, `testing`, `staging`, `local`.

The CLI opens a headless Chromium browser, loads the URL you gave it, and runs every applicable simulation on that one page in sequence. It's a single-page tool: point it at a specific route (like `/checkout`) to test that page, not your whole site.

## Options

```
Usage: flusterduck-test --url <url> [options]

Options:
  --url <url>          Target URL to test (required)
  --key <fd_pub_xxx>   Publishable API key for SDK detection verification
  --mobile             Simulate mobile device with touch
  --headed             Show browser window
  --verbose            Show per-element details
  --json               Output machine-readable JSON
  --threshold <n>      Fail (exit 1) if fewer than n signals simulated
  --help, -h           Show this message
```

`--url` must be `http://` or `https://` with a real hostname (or `localhost` / `127.0.0.1`). `--key`, if passed, must look like a publishable key (`fd_pub_` prefix); the CLI checks the shape but doesn't authenticate it, since the whole point is to test whether that key actually works.

## Simulations

Every run performs this fixed set of simulations on the page. There's no way to pick a subset. They run in this order:

### rage_click

Clicks every button, link, and submit input on the page 5 times in rapid succession (up to 20 elements).

### dead_click

Finds non-interactive elements (`div`, `span`, `p`, headings, `section`, `article`, `header`, `footer`, `main`, filtered to exclude anything containing a link, button, input, or other interactive control) and clicks each one 3 times, spaced about 1.1 seconds apart, up to 20 elements.

### thrash_cursor

Moves the mouse through a 20-step spiral around the center of the viewport, then does 8 fast horizontal reversals.

### scroll_bounce

Scrolls to the middle of the page, back to the top, down to the middle again, and back to the top.

### tab_thrash

Presses Tab repeatedly (at least 15 times, or twice the number of focusable elements on the page, whichever is larger), then Shift+Tab 5 times.

### form_hesitation

Focuses every visible input, select, and textarea on the page for 3.5 seconds each without typing, then blurs.

### form_abandon

Fills about 40% of the fields in each form (or, if there's no `<form>` element, 40% of the loose inputs on the page) and blurs without submitting.

### loop_nav

Finds the first internal link on the page, opens it in a new tab, and navigates back and forth between it and the original URL 4 times.

## Mobile-only simulations

Pass `--mobile` to simulate a 375x812 touch viewport (iPhone-shaped) and add three more simulations to the run:

### tap_miss

Taps 8px off the edge of interactive elements instead of on them, cycling through offsets above, left, and right of the target.

### pinch_zoom

Simulates 4 pinch gestures plus a ctrl+wheel zoom in and out.

### swipe_miss

Swipes in all four directions (left, right, up, down) from the center of the viewport.

## Output

```
Flusterduck Test Suite
=====================
Target: http://localhost:3000
Mode:   desktop
Key:    fd_pub_xxx...

Loading http://localhost:3000...
Page loaded. Running simulations...

Results
-------

SIMULATED (8)  rage_click
SIMULATED (12)  dead_click
SIMULATED (1)  thrash_cursor
SIMULATED (1)  scroll_bounce
SIMULATED (1)  tab_thrash
SIMULATED (6)  form_hesitation
SIMULATED (1)  form_abandon
SIMULATED (1)  loop_nav

Signals simulated: 31
Signals skipped:   0
Signal types:      8
Elements tested:   18

SDK key provided. Check your Flusterduck dashboard to verify detected signals.
```

Pass `--verbose` to see the specific element and outcome behind each line. Pass `--json` to get the same data as a structured report instead (useful for piping into your own CI checks).

A simulation shows as `SKIPPED` when it couldn't find anything to act on (no forms on the page, no internal links to loop through) rather than failing the run.

## CI integration

Run the test suite after your dev server starts to verify the integration hasn't broken:

```yaml
# .github/workflows/test.yml
- name: Start dev server
  run: pnpm dev &

- name: Wait for server
  run: npx wait-on http://localhost:3000

- name: Run Flusterduck test suite
  run: |
    npx flusterduck-test \
      --url http://localhost:3000 \
      --key ${{ secrets.FLUSTERDUCK_TEST_KEY }} \
      --threshold 10
```

`--threshold <n>` exits with code 1 if fewer than `n` signals were simulated across the whole run. It's a check that the page had enough interactive surface for the run to mean anything, not a check that Flusterduck received the signals. Use a low number for a thin page and a higher one for something like a checkout form with several fields and buttons.

Use a separate `fd_pub_` key for CI testing. Create one under Settings > API Keys and label it "test." Data from this key won't affect your production scores if you scope your dashboard view to the production key.

## Debugging a missed signal

If a signal you expected doesn't show up in the dashboard's Live signals view after a run:

1. Confirm the page actually has `data-env="development"` (or `environment: 'development'` in `init()`). Without it, a Playwright-driven browser is filtered as automated traffic and never reaches storage.
2. Check `ignoreElements` in your SDK config. The element the simulation clicked might match a suppression rule.
3. Check `ignorePages`. The page path might be on the skip list.
4. Check `sampleRate`. If it's set below `1.0`, some sessions are dropped before any detector runs.
5. Open the page yourself with `--headed` to watch the browser drive through the simulations, and open your own browser console with `debug: true` passed to `init()` to see the SDK's own logging.

```bash
npx flusterduck-test --url http://localhost:3000 --key fd_pub_xxxxxxxxxxxx --headed --verbose
```
