# Testing Your Integration

`@flusterduck/test-suite` is a Playwright-based CLI that simulates user frustration patterns against a running app. It verifies your Flusterduck setup before you go to production: signals are being detected, the scoring engine is receiving them, and issues will be created from real friction. The simulated user has exactly one job: getting frustrated. It is very good at it.

## Install

```bash
pnpm add -D @flusterduck/test-suite
```

Or run without installing:

```bash
npx @flusterduck/test-suite run --url http://localhost:3000
```

## Running a simulation

Point it at your dev server:

```bash
npx @flusterduck/test-suite run \
  --url http://localhost:3000 \
  --key fd_pub_xxxxxxxxxxxx \
  --scenarios all
```

One thing matters here: declare a development environment. Flusterduck treats
automation and local hosts as noise in production. Events sent from
`localhost` or an automated browser are acknowledged but never stored, so a
verification run can't burn your session quota or create phantom issues. Add
`data-env="development"` to your script tag (or `environment: 'development'`
in your init call) on the app you point the CLI at, and both filters stand
down for that traffic. Accepted names: `development`, `dev`, `test`,
`testing`, `staging`, `local`.

The CLI opens a headless Chromium browser, navigates through your app, and executes frustration scenarios on each page. Each scenario generates specific signal types. After the run, it reports which signals were emitted, whether they were received by the scoring engine, and which scenarios produced no signals.

## Scenarios

### rage-click

Clicks the same element 5 times in rapid succession. Targets interactive elements: buttons, links, form submits.

### dead-click

Clicks non-interactive elements: labels, images, decorative containers, text nodes near interactive elements.

### form-abandon

Fills in one or more fields of a form and navigates away without submitting.

### navigation-loop

Navigates forward in a multi-step flow, then immediately back. Repeats 3 times.

### scroll-thrash

Scrolls down rapidly, then back up, then down again, within a 3-second window.

### tap-miss

On mobile viewport (375px), taps adjacent to interactive elements rather than on them. Simulates small touch targets.

### idle-confusion

Loads a page and stays inactive for 15 seconds without interacting.

## Running specific scenarios

```bash
npx @flusterduck/test-suite run \
  --url http://localhost:3000/checkout \
  --key fd_pub_xxxxxxxxxxxx \
  --scenarios rage-click,form-abandon,dead-click
```

## Output

```
Flusterduck Test Suite v0.4.1
Target: http://localhost:3000
Scenarios: rage-click, form-abandon, dead-click

Running rage-click on /...
  Found 3 interactive elements
  Executed rage-click on button[type="submit"]   signal received
  Executed rage-click on [data-cta="upgrade"]    signal received
  Executed rage-click on nav a:nth-child(2)      signal received

Running form-abandon on /checkout...
  Found 2 forms
  Executed form-abandon on #checkout-form        signal received
  Executed form-abandon on #coupon-form          signal received (low confidence)

Running dead-click on /pricing...
  Found 6 non-interactive elements with click affordance
  Executed dead-click on .plan-card              signal received
  Executed dead-click on .feature-badge          no signal (expected: element filtered)

Summary
  Scenarios run:   3
  Signals emitted: 6
  Signals received by engine: 5
  Missed signals:  1 (.feature-badge matched ignoreElements pattern)
  Issues detected: 0 (volume too low for clustering)

All critical scenarios passed.
```

"Issues detected: 0" is expected during testing. Issue clustering requires signals from multiple distinct sessions. A single simulation run won't cross that threshold. The important check is that signals are emitted and received.

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
    npx @flusterduck/test-suite run \
      --url http://localhost:3000 \
      --key ${{ secrets.FLUSTERDUCK_TEST_KEY }} \
      --scenarios rage-click,form-abandon \
      --fail-on-missed-signals
```

`--fail-on-missed-signals` exits with code 1 if any expected signals weren't received by the engine. Use this to catch SDK initialization regressions.

Use a separate `fd_pub_` key for CI testing. Create one under Settings > API Keys and label it "test." Data from this key won't affect your production scores if you scope your dashboard view to the production key.

## Testing consent flows

Simulate a session where tracking starts paused:

```bash
npx @flusterduck/test-suite run \
  --url http://localhost:3000 \
  --key fd_pub_xxxxxxxxxxxx \
  --scenarios rage-click \
  --consent-state declined
```

With `--consent-state declined`, the runner calls `setConsent(false)` after page load. Signals should not be received by the engine. The test fails if they are.

```bash
--consent-state accepted    # setConsent(true), signals should flow (default)
--consent-state declined    # setConsent(false), signals should not flow
--consent-state unset       # no consent call, tests default behavior
```

## Testing with identify()

Pass segment properties to simulate an identified session:

```bash
npx @flusterduck/test-suite run \
  --url http://localhost:3000/checkout \
  --key fd_pub_xxxxxxxxxxxx \
  --scenarios rage-click \
  --identify '{"plan": "scale", "role": "admin"}'
```

The runner calls `identify()` with the provided properties before executing scenarios. Verify in your dashboard that the session has the expected segment properties attached.

## Debugging a missed signal

If a signal shows "no signal" in the output:

1. Check `ignoreElements` in your SDK config. The element selector might match a suppression rule.
2. Check `ignorePages`. The page path might be on the skip list.
3. Check `sampleRate`. If it's set below 1.0, some sessions won't be tracked. The test runner uses a fixed seed to ensure consistent sampling, but a very low sample rate can still produce skipped sessions.
4. Run with `--debug` to see the full SDK console output in the terminal.

```bash
npx @flusterduck/test-suite run \
  --url http://localhost:3000 \
  --key fd_pub_xxxxxxxxxxxx \
  --scenarios rage-click \
  --debug
```
