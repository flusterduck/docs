# Scoring engine internals

The [confusion scores](/docs/scoring) page covers what scores mean and how to read them. This page covers how they're computed: the exact formula, every signal weight, the confidence system, baseline math, and normalization.

You're here because a score moved and you want to know why. Good.

## The formula

Every page score follows this pipeline:

1. Sum the weighted contribution of each signal: `count * weight * confidence`
2. Divide by active users on the page (volume normalization)
3. Multiply by the co-occurrence multiplier
4. Normalize against the page's baseline (if one exists)
5. Clamp to 0-100

In code, the core calculation is two lines:

```typescript
const rawScore = totalWeightedScore / activeUsers;
const score = rawScore * coOccurrenceMultiplier;
```

If a baseline exists with 7+ samples, the raw score gets z-score normalized:

```typescript
const normalizedScore = ((score - baselineMean) / baselineStdDev) * 25 + 50;
```

That maps scores into a distribution centered at 50, where each standard deviation equals 25 points. A page performing at its historical average lands at 50. One standard deviation worse lands at 75. Two worse hits 100.

Without a baseline, the raw score is clamped directly to 0-100.

## Signal weights

Every signal has a fixed weight that reflects how strong an indicator of friction it is. Higher weight means a single fire moves the score more. The engine uses these weights unless you've set custom weights via the SDK config.

### Tier 1 (high-confidence friction, weights 12-38)

| Signal | Weight | What it detects |
|---|---|---|
| `active_funnel_rejection` | 38 | High-intent user hits friction at terminal step, leaves without converting |
| `dead_end_submit` | 30 | Submit button clicked, no network request or DOM change follows |
| `layout_shift_rage` | 28 | Click lands where a target was before a layout shift moved it |
| `rage_click` | 25 | 3+ clustered clicks within two seconds |
| `error_recovery_loop` | 24 | 3+ submit attempts with repeated errors on the same form |
| `silent_failure_retry` | 22 | Repeated clicks on a primary action while loading stays unresolved |
| `overlay_dismiss_struggle` | 22 | Repeated Escape presses and close clicks while a modal stays open |
| `form_abandonment` | 20 | User enters form data, exits without submitting |
| `disabled_element_attempt` | 20 | User clicks a disabled control more than once |
| `focus_trap` | 20 | Keyboard focus can't escape a container despite Tab and Escape |
| `accidental_click_bounce` | 18 | Navigation reversed almost immediately after the click that caused it |
| `dead_click` | 12 | Click on a non-interactive element with no response |

### Tier 2 (behavioral patterns, weights 10-18)

| Signal | Weight |
|---|---|
| `u_turn_navigation` | 18 |
| `information_maze` | 18 |
| `multi_step_reset` | 18 |
| `close_click_reversal` | 18 |
| `load_abandonment` | 16 |
| `query_thrashing` | 15 |
| `filter_spiral` | 15 |
| `tab_thrash` | 15 |
| `copy_paste_rework` | 14 |
| `repeat_form_input` | 14 |
| `bulk_action_miss` | 14 |
| `sticky_obstruction_click` | 14 |
| `post_action_anxiety` | 13 |
| `settings_hop` | 12 |
| `false_swipe` | 12 |
| `keyboard_nav_frustration` | 10 |

### Tier 3 (ambient signals, weights 6-15)

| Signal | Weight |
|---|---|
| `thrash_cursor` | 15 |
| `help_hunt` | 14 |
| `confidence_collapse` | 13 |
| `target_near_miss` | 12 |
| `user_confusion_idle` | 12 |
| `thrash_hover` | 10 |
| `text_selection_thrash` | 10 |
| `jerky_scrolling` | 9 |
| `viewport_thrashing` | 9 |
| `visibility_thrashing` | 9 |
| `passive_drift` | 8 |
| `scroll_depth_abandon` | 6 |

### Revenue tier (weights 34-40)

| Signal | Weight |
|---|---|
| `value_collapse_downgrade` | 40 | 
| `layout_exhaustion_settling` | 34 |

Revenue signals carry the heaviest weights because they represent friction that directly costs money. A single `value_collapse_downgrade` fire (user intended a higher plan, hit friction, bought a lower one) contributes 40 points per fire before normalization.

### Custom weights

You can override any signal weight through the SDK config. The engine merges your custom weights on top of the defaults:

```typescript
const weights = { ...SIGNAL_WEIGHTS, ...customWeights };
```

If you set `rage_click` to 50, it'll carry double the default influence. If you set `passive_drift` to 0, it won't contribute at all.

Unknown signal types that somehow reach the engine (shouldn't happen, but the code is defensive) get a fallback weight of 10.

## The confidence system

Each signal fire can carry a confidence value between 0 and 1. Confidence scales the weighted contribution of that specific fire:

```typescript
function scoreFire(weight: number, count: number, confidence = 1): number {
  const cf = Math.max(0, Math.min(1, confidence));
  return count * weight * cf;
}
```

A rage click detected from a clear 3-click cluster in 800ms might fire with confidence 0.95. A borderline detection (exactly 3 clicks at exactly 2000ms) might fire at 0.4. The score reflects that difference.

Key properties of the confidence system:

- **Defaults to 1.** If no confidence is provided, the signal contributes at full weight. All pre-confidence-system behavior is preserved.
- **Clamped to [0, 1].** Confidence can never amplify a signal beyond its base weight. A value of 5 gets clamped to 1. A value of -1 gets clamped to 0.
- **NaN falls back to 1.** Malformed confidence values are treated as full confidence rather than zeroed out, so a bug in confidence reporting can't silently suppress real signals.

In the edge function, confidence works slightly differently. Instead of averaging confidence per signal type, the server sums per-fire confidence values:

```typescript
agg.confidenceSum += cf;
// ...
weighted_score = agg.confidenceSum * weight;
```

When every fire has confidence 1.0, this equals `count * weight`, which is the same as the legacy formula. When fires have mixed confidence, the sum naturally weights higher-confidence detections more heavily.

## Volume normalization

Raw weighted scores are divided by the number of active users on the page:

```
rawScore = totalWeightedScore / activeUsers
```

Ten rage clicks across 20 sessions is a 12.5 raw score (`10 * 25 / 20`). Ten rage clicks across 2,000 sessions is 0.125 (`10 * 25 / 2000`). The rate matters, not the count.

If `activeUsers` is zero, negative, or NaN, the engine returns an empty score (score 0, no breakdown, no trend). Division by zero doesn't happen.

## Co-occurrence multiplier

When multiple users are frustrated on the same page simultaneously, the score increases. The multiplier is a step function based on how many users showed frustration signals within the scoring window:

| Frustrated users in window | Multiplier |
|---|---|
| 0-9 | 1.0 (no amplification) |
| 10-19 | 1.5 |
| 20-49 | 2.0 |
| 50+ | 2.5 |

This is applied after volume normalization but before baseline normalization:

```
score = rawScore * coOccurrenceMultiplier
```

The logic: if 50 users are all hitting friction on the same page at the same time, something is probably broken. That's qualitatively different from 50 users hitting friction over a week. The multiplier captures that distinction.

## Baseline computation

A baseline is the page's historical "normal." It's computed from score history once a page has 7+ days of data.

The algorithm collects all score history entries, then computes:

- **Mean:** arithmetic average of all scores in the history
- **Standard deviation:** sample standard deviation (divides by `n-1`)
- **Time-of-day means:** average score per UTC hour (0-23), so the engine knows this page is normally noisier at 2pm than 3am
- **Day-of-week means:** average score per weekday (0=Sunday through 6=Saturday)

```typescript
const mean = scores.reduce((a, b) => a + b, 0) / scores.length;
const variance = scores.reduce((sum, s) => sum + (s - mean) ** 2, 0) 
  / (scores.length > 1 ? scores.length - 1 : 1);
const stdDev = Math.sqrt(variance);
```

The minimum data requirement is 7 days of elapsed time between the oldest and newest score entries. This isn't 7 data points; it's 7 calendar days of coverage. A page could have 672 data points (one per 15 minutes) or 14 (twice a day), but the span must be at least 7 days.

### Time-adjusted baselines

The `getTimeAdjustedBaseline` function shifts the baseline mean based on the current hour and day:

```
adjusted = mean + (hourMean - mean) + (dayMean - mean)
```

If the page's overall mean is 25 but its Monday-2pm mean is 35, the adjusted baseline for a Monday at 2pm is 35. Anomaly detection uses this adjusted value, so a spike that's normal for Monday afternoon won't trigger a false alarm.

## Baseline normalization (z-score mapping)

When a page has a mature baseline (7+ samples), the raw score is mapped to a 0-100 scale using z-score normalization:

```
normalizedScore = ((score - baselineMean) / max(baselineStdDev, 1)) * 25 + 50
```

The `max(std_dev, 1)` floor prevents division by zero when a page has constant scores (zero variance). Without it, a page that always scores exactly 20 would produce infinity when it scores 21.

What this formula produces:

| Raw score vs baseline | Normalized score |
|---|---|
| 2 std devs below mean | 0 |
| 1 std dev below mean | 25 |
| At the mean | 50 |
| 1 std dev above mean | 75 |
| 2 std devs above mean | 100 |

The result is clamped to [0, 100] after normalization. A page that's 3 standard deviations above its baseline still reads as 100.

## Deviation and trend

Deviation is the z-score before clamping. It tells you how many standard deviations the current score is from the baseline mean:

```typescript
deviation = (normalizedScore - 50) / 25;
```

Trend uses deviation to classify the direction:

| Deviation | Trend |
|---|---|
| > 1.0 | `up` (friction is increasing) |
| < -1.0 | `down` (friction is decreasing) |
| -1.0 to 1.0 | `stable` |

Pages without a mature baseline always report `stable`. The engine won't guess at a trend without enough history.

## Signal breakdown

Every page score includes a breakdown array sorted by weighted contribution (highest first). Each entry contains:

```typescript
{
  signal: 'rage_click',    // which signal
  count: 12,               // how many fires
  weight: 25,              // the weight used (default or custom)
  weighted_score: 300,     // count * weight * confidence
  percentage: 64           // share of totalWeightedScore
}
```

The `dominant_signal` on the PageScore is the first entry in this sorted breakdown: the signal contributing the most to this page's score right now.

Percentages are rounded integers and sum to approximately 100. They tell you which signal type is driving the score. If `rage_click` is at 64% and `dead_click` is at 22%, the score is mostly about rage clicks.

## Budget evaluation

Confusion budgets work independently from the baseline system. A budget defines an absolute ceiling for a specific page pattern:

```typescript
{
  page_pattern: '/checkout*',
  target_score: 50,
  period: 'weekly',
  escalation_action: 'alert'
}
```

The budget check counts how much time within the current period the page spent above the target score. It walks the score history, calculates the milliseconds over budget, and reports consumption as a percentage:

```typescript
const consumptionPercent = Math.round((overBudgetMs / totalPeriodMs) * 100);
```

| Consumption | Status |
|---|---|
| < 70% | Within budget |
| 70-99% | Warning |
| 100% | Exhausted |

Budgets also track the number of calendar days the page exceeded the target, reported in the alert message.

Period bounds are computed in UTC: daily resets at midnight UTC, weekly resets on Sunday, monthly resets on the first of the month.

## Anomaly detection

The `isAnomaly` function compares the current score against the time-adjusted baseline plus a configurable standard deviation multiplier (default 2):

```typescript
function isAnomaly(currentScore, baseline, stdDevMultiplier = 2) {
  const adjustedBaseline = getTimeAdjustedBaseline(baseline);
  const threshold = adjustedBaseline + baseline.std_dev * stdDevMultiplier;
  return currentScore > threshold;
}
```

At the default multiplier, a score needs to exceed the adjusted baseline by 2 standard deviations to qualify as an anomaly. This means roughly 2.5% of natural variation would trigger it, assuming normal distribution.

## Trend detection

The `isTrending` function checks whether recent scores represent a sustained increase over the baseline:

```typescript
function isTrending(recentScores, baseline, increasePercent = 30) {
  if (recentScores.length < 7) return false;
  const recentMean = recentScores.reduce((a, b) => a + b, 0) / recentScores.length;
  const percentIncrease = ((recentMean - baseline.mean) / max(baseline.mean, 1)) * 100;
  return percentIncrease >= increasePercent;
}
```

At least 7 recent data points are required. The default threshold is a 30% increase in the recent mean compared to the historical mean. This catches slow drift that wouldn't trigger a spike alert but represents a sustained regression.

## Score labels

Scores map to human-readable labels and duck states:

| Score | Label | Duck state |
|---|---|---|
| 0-20 | Calm. Everything is normal. | `calm` |
| 21-40 | Mild friction. Worth watching. | `watching` |
| 41-60 | Noticeable confusion. Something changed. | `alert` |
| 61-80 | Significant frustration. Investigate. | `flustered` |
| 81-100 | Critical. Something is broken. Fix it now. | `critical` |

## Worked example

A checkout page has 100 active users. Five users rage-clicked the submit button (confidence 0.9 each), and eight users abandoned the form (confidence 1.0 each). No co-occurrence spike. The page's baseline mean is 30 with a standard deviation of 8.

Step 1, weighted contributions:

```
rage_click:      5 * 25 * 0.9 = 112.5
form_abandonment: 8 * 20 * 1.0 = 160.0
total = 272.5
```

Step 2, volume normalization:

```
rawScore = 272.5 / 100 = 2.725
```

Step 3, co-occurrence (no spike, multiplier is 1.0):

```
score = 2.725 * 1.0 = 2.725
```

Step 4, baseline normalization:

```
normalizedScore = ((2.725 - 30) / 8) * 25 + 50
                = (-27.275 / 8) * 25 + 50
                = -3.409 * 25 + 50
                = -85.23 + 50
                = -35.23
```

Step 5, clamp:

```
score = max(0, min(100, -35.23)) = 0
```

The page scores 0. That raw activity level is far below the page's normal friction. The baseline mean of 30 represents much heavier historical signal volume, so current activity looks calm by comparison.

If the same page had no baseline yet, the raw score of 2.725 would clamp directly to 2.7. Without historical context, the engine reports the absolute value.

## Edge function vs package

The scoring package (`@flusterduck/scoring`) and the `compute-scores` edge function implement the same algorithm with one difference in how they aggregate confidence.

The package's `computePageScore` takes pre-aggregated signal counts with an averaged confidence per signal type. The edge function aggregates raw signal events from the database, summing per-fire confidence values directly. Both produce `confidenceSum * weight` as the weighted score for a signal type, which equals `count * weight` when all fires have confidence 1.0.

The edge function also writes scores to the `page_scores` table (upserted by site and page) and appends to `score_history` for baseline computation.
