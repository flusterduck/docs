# Reacting to signals

Flusterduck normally works in the background: it watches behavior, clusters friction into issues, and verifies your fixes. But sometimes you want to act on a friction signal **the moment it happens** in the browser, before it is ever sent or aggregated.

`onSignal` gives you that. The instant a detector fires (a rage click, a dead click, form hesitation, scroll thrash, and so on) your code runs, synchronously, with the signal in hand.

## Why react in real time

A detected signal is a person struggling **right now**. That is a moment to help:

- Pop a help widget or live chat when someone rage-clicks a checkout button.
- Surface an inline hint when a form field triggers hesitation.
- Offer a discount or a callback when an exit-intent signal fires on pricing.
- Power a live, in-page demo of Flusterduck (this is how our own site does it).

The signal stays on the device until you decide what to do with it. No round trip, no waiting for aggregation.

## The callback

Pass `onSignal` to `init`:

```ts
import { init } from 'flusterduck'

init({
  key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!,
  onSignal(signal) {
    if (signal.name === 'rage_click') {
      openHelpWidget({ near: signal.element })
    }
  },
})
```

Or subscribe at any time after init, which returns an unsubscribe function:

```ts
import { onSignal } from 'flusterduck'

const off = onSignal((signal) => {
  console.log(signal.name, 'on', signal.element)
})

// later
off()
```

Both run synchronously when the signal is detected, before it is batched or sent. Listeners are torn down automatically on `destroy()`.

## The signal payload

Your listener receives an `EmittedSignal`:

| Field | Type | Description |
| --- | --- | --- |
| `name` | `string` | The signal type, e.g. `rage_click`, `dead_click`, `form_hesitation`. |
| `element` | `string` | CSS selector of the element involved (empty if not element-bound). |
| `weight` | `number` | Severity contribution, 0 to 100. |
| `meta` | `object` | Detector metadata (counts, distances, timings). |
| `ts` | `number` | Timestamp in milliseconds. |

It is behavioral only. It never contains form values, text content, or any PII, the same guarantee as everything else Flusterduck collects.

## The window event

If you would rather listen without wiring a callback through `init`, opt in with `emitSignalEvents` and listen for a `flusterduck:signal` event on `window`:

```ts
init({
  key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!,
  emitSignalEvents: true,
})

window.addEventListener('flusterduck:signal', (event) => {
  const signal = event.detail // EmittedSignal
  showToast(`${signal.name} on ${signal.element}`)
})
```

This is handy for decoupled code, analytics bridges, or dropping a listener in from a separate script. It is **off by default** so signals are not exposed to other scripts on the page unless you ask for it.

## Notes

- `onSignal` fires for automatically detected friction signals and for manual `signal()` calls.
- It does not change what gets sent to Flusterduck. Reacting in the browser and tracking the issue are independent.
- Keep listeners fast. They run on the detection path, so a slow listener can affect responsiveness. Flusterduck swallows listener errors so a thrown error can never break ingestion.
- If `sampleRate` is below 1, sampled-out sessions emit nothing at all, including `onSignal`. Use `sampleRate: 1` when you depend on it for UX.
