# Webhooks

Flusterduck can push events to your own server as they happen. Configure a webhook endpoint in the dashboard under Settings > Integrations > Webhooks.

## Events

Two events actually deliver today:

| Event | Fires when |
|---|---|
| `alert.fired` | An alert rule fires |
| `issue.verified` | A deploy is measured to have fixed an issue. Carries the affected user refs for the [resolution loop](/resolution-loop) |

A new endpoint subscribes to `alert.fired`, `issue.created`, and `issue.regressed` by default, but the pipeline only emits `alert.fired` and `issue.verified` right now. Subscribe to those two and you'll get everything that fires. Pass your own `events` array when you create the endpoint to narrow it.

## Payload shape

Every delivery is a POST with a JSON body of exactly two fields:

```json
{
  "event": "alert.fired",
  "data": {}
}
```

`event` is the event name. `data` is the resource that changed. There's no wrapper envelope, and no top-level `site_id` or `timestamp`. The delivery id, event name, and timestamp all ride in headers (below), so read those for routing and replay checks, not the body.

### alert.fired

`data` is the full alert row. `severity` is one of `warning`, `high`, `critical`. `status` starts at `fired`.

```json
{
  "event": "alert.fired",
  "data": {
    "id": "9b5e1f4c-2d8a-4f3e-8b6d-7c1a5e9f2b4d",
    "org_id": "7f2c9d4a-4b5e-4c3a-9b1d-2e6a4f8c7d5b",
    "site_id": "1a2b3c4d-5e6f-4a3b-8c1d-9e2f4a6b8c0d",
    "page": "/checkout",
    "issue_id": null,
    "rule_id": "4c8b2e7a-5f1d-42a9-9b3e-6d1c8a5f2e7b",
    "trigger_type": "spike",
    "severity": "high",
    "status": "fired",
    "score": 68,
    "threshold": 25,
    "message": "Checkout confusion spiked to 68",
    "channels": ["email", "webhook"],
    "created_at": "2026-06-10T14:22:05Z"
  }
}
```

### issue.verified

`data` is the verification receipt, including the opaque affected-user refs for the [resolution loop](/resolution-loop). Confusion scores run 0 to 100.

```json
{
  "event": "issue.verified",
  "data": {
    "issue_id": "3a7f2c9d-4e1b-42d8-9c5a-1f6b8e2d4a7c",
    "title": "Dead clicks on upgrade button",
    "page": "/pricing",
    "confusion_before": 42,
    "confusion_after": 31,
    "reduction_pct": 26,
    "revenue_recovered_cents": 48000,
    "affected_user_refs": ["u_8a3f2c", "u_1c9d4e"],
    "identified_sessions": 74,
    "unidentified_sessions": 12,
    "refs_truncated": false
  }
}
```

## Headers

Every delivery carries these headers:

```
X-Flusterduck-Event: alert.fired
X-Flusterduck-Delivery: 8a3f2c9d-4e1b-42d8-9c5a-1f6b8e2d4a7c
X-Flusterduck-Timestamp: 1749565325
X-Flusterduck-Signature: 4f3e8b6d7c1a5e9f2b4d9b5e1f4c2d8a4f3e8b6d7c1a5e9f2b4d9b5e1f4c2d8a
```

`X-Flusterduck-Event` is the event name. `X-Flusterduck-Delivery` is a stable id for this delivery: it stays the same across retries, so dedup on it if you ever process the same delivery twice. `X-Flusterduck-Timestamp` is Unix seconds.

## Signature verification

`X-Flusterduck-Signature` is HMAC-SHA256, hex-encoded, over the string `<timestamp>.<raw_body>` using your endpoint secret (it starts with `whsec_fd`). There's no `sha256=` prefix. It's the bare hex digest. Compare against the raw hex, and verify before trusting the payload.

### Node.js (Express)

```ts
import crypto from 'crypto'
import express from 'express'

const app = express()

app.post('/webhooks/flusterduck', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-flusterduck-signature'] as string
  const timestamp = req.headers['x-flusterduck-timestamp'] as string

  if (!verifySignature(req.body, timestamp, signature)) {
    return res.status(401).send('Invalid signature')
  }

  const event = JSON.parse(req.body.toString())
  // handle event...

  res.status(200).send('ok')
})

function verifySignature(body: Buffer, timestamp: string, signature: string): boolean {
  const secret = process.env.FLUSTERDUCK_WEBHOOK_SECRET!
  const payload = `${timestamp}.${body.toString()}`
  const expected = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex')

  const expectedBuf = Buffer.from(expected)
  const signatureBuf = Buffer.from(signature)
  // Length check first: timingSafeEqual throws on a length mismatch.
  if (expectedBuf.length !== signatureBuf.length) return false
  return crypto.timingSafeEqual(signatureBuf, expectedBuf)
}
```

### Next.js API route (App Router)

```ts
// app/api/webhooks/flusterduck/route.ts
import crypto from 'crypto'
import { NextRequest, NextResponse } from 'next/server'

export async function POST(req: NextRequest) {
  const body = await req.text()
  const signature = req.headers.get('x-flusterduck-signature') ?? ''
  const timestamp = req.headers.get('x-flusterduck-timestamp') ?? ''

  const secret = process.env.FLUSTERDUCK_WEBHOOK_SECRET!
  const payload = `${timestamp}.${body}`
  const expected = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex')

  const expectedBuf = Buffer.from(expected)
  const signatureBuf = Buffer.from(signature)
  const valid = expectedBuf.length === signatureBuf.length
    && crypto.timingSafeEqual(signatureBuf, expectedBuf)

  if (!valid) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 401 })
  }

  const event = JSON.parse(body)
  // handle event...

  return NextResponse.json({ received: true })
}
```

### Replay protection

The timestamp in `X-Flusterduck-Timestamp` is Unix seconds. Reject requests where the timestamp is more than 5 minutes old:

```ts
const fiveMinutes = 5 * 60
const age = Math.floor(Date.now() / 1000) - parseInt(timestamp, 10)
if (age > fiveMinutes) {
  return res.status(401).send('Request too old')
}
```

## Retry behavior

A delivery that gets a non-2xx response, a timeout, or a redirect is retried. Up to 10 attempts total, then it's marked dead. The delay after a failed attempt doubles each time, capped at 1 hour:

| After attempt | Next retry in |
|---|---|
| 1 | ~1 minute |
| 2 | ~2 minutes |
| 3 | ~4 minutes |
| 4 | ~8 minutes |
| 5 | ~16 minutes |
| 6 | ~32 minutes |
| 7 and up | 1 hour (the cap) |

Retries reuse the same `X-Flusterduck-Delivery` id, so if a slow 2xx makes us retry a delivery you already processed, dedup on that id. Once a delivery is dead, view it and retry it by hand under Settings > Integrations > Webhooks > Delivery History.

We never follow redirects. A 3xx from your endpoint is treated as a failed attempt, not a hop to chase. Respond directly with a 2xx.

## Testing

Send a test event from the dashboard to verify your endpoint before going live. It arrives as `test.ping` and is signed the same way as live events.

Your endpoint has 8 seconds to respond with a 2xx. Slower than that is a timeout and counts as a failed attempt.
