# Detectors, signals, and friction types

Three words come up everywhere in Flusterduck (detectors, signals, and friction types) and they're easy to mix up. Here's the difference in plain English, with one example carried all the way through.

## The one-sentence version

**Detectors watch. Signals report. Friction types organize.** Repeated signals then cluster into **issues**: the ranked list you actually work from.

## Detectors: the watchers

A detector is a small piece of code inside the SDK that watches for one specific pattern of struggle. Nothing more.

The rage-click detector watches for someone clicking the same spot again and again in frustration. The dead-click detector watches for taps on things that look clickable but aren't. The ghost-toggle detector watches for switches that get clicked but never change. About a hundred of these watchers ship in the SDK, and every one of them runs automatically from the single script tag. There is nothing to configure, instrument, or turn on.

Think of detectors as a hundred quiet observers, each trained to notice exactly one way a person can get stuck.

## Signals: the reports

A signal is what a detector sends the moment it sees its pattern: one timestamped observation from one real person's session:

> Rage click on **"Apply coupon"** · `/checkout` · 2:51 PM

That's a single signal: which pattern fired, on which element (its visible label, PII-redacted in the browser before anything is sent), on which page, at what moment, with a confidence score for how certain the detector is. Signals are the raw sightings: plentiful, small, and individually not very dramatic. One person rage-clicking once might mean nothing. The value comes from what happens next.

## Friction types: the filing system

A friction type is the named category a signal belongs to: the label Flusterduck's scoring, dashboard, and AI use to group, weight, and rank what the detectors saw. `rage_click`, `dead_click`, `form_abandonment`, `no_op_interaction`: there are 127 of them, each with a weight reflecting how strongly it predicts a lost conversion (a dead-end checkout submit weighs far more than a stray scroll bounce).

Friction types are why the dashboard can say something useful instead of showing you a firehose: every sighting is filed, weighted, and comparable.

## Why are there more detectors than friction types?

Because several watchers can file reports under one category. Different detectors catch mobile taps that miss their target, thumb-zone misses, and near-miss clicks from different angles, and their reports can land in the same friction type. The detector count grows every time we learn a new way users get stuck (it's an iterative process: every real-world bug we see becomes a detector); the friction-type list grows more slowly, only when a genuinely new *category* of struggle shows up.

## From signals to issues

Signals don't reach your issue list one at a time. When the same friction type keeps firing on the same page or element across multiple sessions, Flusterduck clusters those signals into a single **issue**, with a plain-language title, a severity bounded by how many users it actually reached, a confidence grade, a revenue estimate, and (on paid plans) an AI diagnosis grounded in your page's actual content. That ranked issue list is the product; detectors, signals, and friction types are the machinery underneath it.

Read more: [Signals reference](/signals) · [Issues](/issues) · [Scoring](/scoring)

## FAQ

**Do I need to configure any of this?** No. All detectors run automatically from one script tag. You can turn individual detectors off in the SDK config if you ever need to, but the default is everything on.

**Does more detectors mean more data sent from my site?** Barely: signals are tiny structured events (a name, a selector, a redacted label, timestamps), not recordings. There is no session replay and no page content captured from your visitors' sessions.

**Can one user action fire multiple detectors?** Yes, and that's useful: a broken button might draw rage clicks *and* no-op interactions *and* an error signal. Co-occurring signals on the same element are strong evidence they share one root cause, and the clustering treats them that way.
