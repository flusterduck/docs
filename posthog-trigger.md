# PostHog trigger layer

PostHog bills by event volume. A typical session emits 60-200 events, and 95-98% of those sessions have zero friction. You're paying full price for data that confirms everything worked fine.

The trigger layer wraps `posthog.capture` and cuts ingestion 70-90% while keeping 100% of the events that explain why a user got frustrated. It ships in the `flusterduck` SDK as `initPostHogTrigger`.

## Why per-event sampling fails

Dropping 90% of events with `Math.random()` breaks two things.

**Funnels.** PostHog funnels need every step from a single user. Per-event sampling keeps step 1 for one user and step 3 for another. Funnel numbers become garbage.

**Context.** The events that explain a rage click happen *before* the rage click. If the drop decision happens at capture time, the lead-up is gone by the time you know you needed it.

## How it works

### Per-user deterministic sampling

The keep/drop decision hashes the PostHog `distinct_id` with FNV-1a, so a user is either fully in or fully out of the sampled cohort. Kept users keep every event. Funnels, paths, and session groupings stay internally coherent. Aggregate counts scale by `1 / sampleRate`, the standard sampled-cohort model.

### Retroactive capture

Dropped events aren't discarded. They sit in a rolling buffer (default: last 30 seconds, max 50 events). When Flusterduck detects a friction signal (rage click, dead click, error encounter, form validation loop, thrash cursor), the buffer flushes to PostHog. Every recovered event carries:

- `fd_recovered: true`
- `fd_recovered_reason`: the signal that triggered recovery (e.g. `"rage_click"`)
- `fd_original_ts`: the original capture timestamp

You pay for the lead-up events only when a user actually got frustrated. Full context on the 2% of sessions that matter, 10% sampling on the 98% that don't.

### Boost window

After a critical signal fires, everything is captured for 10 seconds (configurable) so the aftermath is preserved alongside the lead-up. Boosted events carry `fd_boosted: true`.

### Priority tiers

| Tier | Events | Behavior |
|---|---|---|
| P0 (critical) | `$rageclick`, `$dead_click`, `$exception`, `$identify`, `$create_alias`, `$groupidentify`, `$survey_sent`, `$survey_dismissed`, `$feature_flag_called` | Always captured |
| P1 (conversions) | Your `conversionEvents` list, plus pattern-matched names: signup, checkout, purchase, payment, upgrade, trial, order completed | Always captured |
| P2 (general) | Pageviews, custom events, everything else | Sampled at `baselineSampleRate` (default 10%) |
| P3 (noise) | Scroll and heatmap events (`$$heatmap*`, `$scroll*`) | Sampled at `scrollSampleRate` (default 1%) |

P0 and P1 events always pass. P2 and P3 events are sampled unless a boost window is active or the buffer flushes them retroactively.

### Signal annotation

Every captured event carries the user's recent Flusterduck signals as properties:

- `fd_recent_signals`: array of signal names from the last 60 seconds
- `fd_signal_count`: how many signals fired recently

You can segment any PostHog insight by frustration without leaving PostHog. Filter `fd_signal_count > 0` to see only frustrated-session data.

## Setup

Call `initPostHogTrigger` after `init`. It waits for PostHog to load on its own, so order doesn't matter and you don't need to coordinate script loading.

```typescript
import { init, initPostHogTrigger } from 'flusterduck';

init({ key: 'fd_pub_...' });

initPostHogTrigger({
  conversionEvents: ['plan_changed', 'seat_added'],
});
```

That's the whole integration. Two lines after your existing Flusterduck init.

### With React

```typescript
import { useEffect } from 'react';
import { initPostHogTrigger, destroyPostHogTrigger } from 'flusterduck';

function App() {
  useEffect(() => {
    initPostHogTrigger({
      conversionEvents: ['plan_changed'],
    });
    return () => destroyPostHogTrigger();
  }, []);

  return <>{/* your app */}</>;
}
```

### With Next.js

If you're using `@flusterduck/next`, call `initPostHogTrigger` in a client component or in a `useEffect` in your root layout. The `FlusterduckScript` component handles the core SDK. The trigger layer is a separate call because not every Flusterduck customer uses PostHog.

```typescript
'use client';

import { useEffect } from 'react';
import { initPostHogTrigger, destroyPostHogTrigger } from 'flusterduck';

export function PostHogTrigger() {
  useEffect(() => {
    initPostHogTrigger();
    return () => destroyPostHogTrigger();
  }, []);
  return null;
}
```

Drop `<PostHogTrigger />` in your root layout alongside `<FlusterduckScript />`.

## Options

| Option | Type | Default | What it does |
|---|---|---|---|
| `baselineSampleRate` | `number` | `0.1` | Keep rate for P2 events (general pageviews, custom events). Range: 0 to 1. |
| `scrollSampleRate` | `number` | `0.01` | Keep rate for P3 events (scroll and heatmap noise). Range: 0 to 1. |
| `conversionEvents` | `string[]` | `[]` | Event names added to the always-keep P1 tier, on top of the built-in conversion pattern match. |
| `criticalSignals` | `string[]` | `['rage_click', 'rage_click_repeat_target', 'dead_click', 'error_encounter', 'form_validation_loop', 'thrash_cursor']` | Flusterduck signals that trigger retroactive buffer flush and the boost window. |
| `bufferWindowMs` | `number` | `30000` | How far back retroactive capture reaches, in milliseconds. |
| `bufferMaxEvents` | `number` | `50` | Max events held in the retroactive buffer. |
| `boostWindowMs` | `number` | `10000` | How long everything is captured after a critical signal fires, in milliseconds. |

### Tuning the sample rates

The defaults (10% baseline, 1% scroll) work well for most sites. If you want to tune:

**Lower `baselineSampleRate`** to cut cost further. At `0.05` (5%), you're keeping conversions, critical events, and friction context but paying half as much for clean-session data. Funnel analysis still works because sampling is per-user, not per-event.

**Raise `scrollSampleRate`** if you rely on PostHog heatmaps. At `0.05` you'll get 5x more heatmap data while still cutting 95% of scroll noise.

**Add to `conversionEvents`** any custom event name that PostHog funnels depend on. The built-in pattern catches common names (signup, checkout, purchase, upgrade, trial, order placed), but your product-specific events like `workspace_created` or `first_query_run` need to be listed explicitly.

## Cost math

A site doing 100,000 sessions/month with an average of 120 PostHog events per session:

| Scenario | Monthly events | Cost at $0.00045/event |
|---|---|---|
| No trigger layer | 12,000,000 | $5,400 |
| Trigger layer, defaults | 1,800,000 | $810 |
| Trigger layer, 5% baseline | 1,200,000 | $540 |

The savings scale linearly with session volume. Heatmap-heavy sites see larger cuts because P3 dominates their event volume.

The 1.8M figure comes from: 100% of P0/P1 events (roughly 5% of total), 10% of P2 events, 1% of P3 events, plus retroactive flushes on the ~2-5% of sessions with friction signals.

## PostHog properties reference

Properties the trigger layer adds to captured events:

| Property | Type | When present |
|---|---|---|
| `fd_recent_signals` | `string[]` | Any event captured while recent Flusterduck signals exist (last 60 seconds) |
| `fd_signal_count` | `number` | Same as above |
| `fd_recovered` | `boolean` | Events flushed retroactively from the buffer after a critical signal |
| `fd_recovered_reason` | `string` | The Flusterduck signal name that caused the flush (e.g. `"rage_click"`) |
| `fd_original_ts` | `number` | Original capture timestamp (Unix ms) on recovered events |
| `fd_boosted` | `boolean` | Events captured during the boost window after a critical signal |

### Using these in PostHog

**Segment by frustration.** Create a cohort where `fd_signal_count > 0`. Apply it to any insight to see only frustrated-session behavior.

**Find recovered context.** Filter events where `fd_recovered = true` to see the lead-up to friction moments. The `fd_recovered_reason` tells you which signal triggered the recovery.

**Measure boost windows.** Filter `fd_boosted = true` to see what users did immediately after a friction moment. Useful for understanding whether frustrated users retry, leave, or contact support.

## Cleanup

Call `destroyPostHogTrigger()` to remove the wrapper and restore the original `posthog.capture`. The SDK checks that its wrapper is still installed before restoring, so if PostHog re-initialized in the meantime, it won't overwrite the new capture function.

```typescript
import { destroyPostHogTrigger } from 'flusterduck';

destroyPostHogTrigger();
```

## How it handles the PostHog stub

The posthog-js snippet installs a queueing stub on `window.posthog` before the real library loads. The real library then replaces the stub's methods. Wrapping the stub would be silently undone.

The trigger layer polls every 500ms until `posthog.__loaded` is `true`, then wraps the real `capture`. Polling stops after 30 seconds if PostHog never loads. The page is never affected: no errors thrown, no visible behavior change, no console noise.

## What this doesn't do

It doesn't replace PostHog. It assumes you keep PostHog and want it cheaper.

It's not lossless for aggregate counts. Sampled tiers undercount by design. Multiply by `1 / sampleRate` or segment on the kept cohort.

It does nothing without PostHog on the page. No `window.posthog`, no behavior, no errors.
