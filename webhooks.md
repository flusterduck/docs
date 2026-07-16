# Webhooks

Flusterduck can push events to your own server as they happen. Configure a webhook endpoint in the dashboard under Settings > Integrations > Webhooks.

## Events

| Event | Fires when |
|---|---|
| `issue.created` | A new UX issue is detected |
| `issue.updated` | An issue's status changes (triaged, resolved, regressed, etc.) |
| `issue.verified` | A fix is measured working after a deploy; carries the affected user refs for the [resolution loop](/resolution-loop) |
| `alert.triggered` | An alert rule fires |
| `alert.acknowledged` | An alert is acknowledged |
| `alert.resolved` | An alert is resolved |
| `score.spike` | A page confusion score increases sharply |
| `deploy.recorded` | A deploy is recorded and scored |

## Payload shape

Every webhook delivery is a POST request with a JSON body:

```json
{
  "event": "issue.created",
  "site_id": "7f2c9d4a-4b5e-4c3a-9b1d-2e6a4f8c7d5b",
  "timestamp": "2026-06-10T14:22:05Z",
  "data": { ... }
}
```

The `data` object contains the full resource that changed. For `issue.created` it's the full issue object. For `alert.triggered` it's the full alert with the rule that fired it.

### issue.created

```json
{
  "event": "issue.created",
  "site_id": "7f2c9d4a-4b5e-4c3a-9b1d-2e6a4f8c7d5b",
  "timestamp": "2026-06-10T14:22:05Z",
  "data": {
    "id": "iss_3a7f2c9d4e1b",
    "title": "Dead clicks on upgrade button",
    "page": "/pricing",
    "selector": "[data-cta='upgrade']",
    "signal_type": "dead_click",
    "signal_count": 47,
    "severity": 72,
    "status": "open",
    "created_at": "2026-06-10T14:22:05Z"
  }
}
```

### alert.triggered

```json
{
  "event": "alert.triggered",
  "site_id": "7f2c9d4a-4b5e-4c3a-9b1d-2e6a4f8c7d5b",
  "timestamp": "2026-06-10T14:22:05Z",
  "data": {
    "id": "alt_9b5e1f4c2d8a",
    "rule_id": "rul_4c8b2e7a5f1d",
    "rule_name": "Checkout rage click spike",
    "page": "/checkout",
    "trigger_type": "spike",
    "score_before": 31,
    "score_after": 68,
    "status": "open",
    "triggered_at": "2026-06-10T14:22:05Z"
  }
}
```

### deploy.recorded

```json
{
  "event": "deploy.recorded",
  "site_id": "7f2c9d4a-4b5e-4c3a-9b1d-2e6a4f8c7d5b",
  "timestamp": "2026-06-10T14:22:05Z",
  "data": {
    "id": "dep_6e3d1a8f7c2b",
    "version": "2026.06.10",
    "environment": "production",
    "confusion_before": 42,
    "confusion_after": 31,
    "recorded_at": "2026-06-10T14:22:05Z"
  }
}
```

## Signature verification

Every webhook delivery includes two headers:

```
X-Flusterduck-Signature: sha256=<hex>
X-Flusterduck-Timestamp: <unix timestamp seconds>
```

The signature is HMAC-SHA256 over `timestamp.raw_body` using your webhook secret. Verify it on your server before trusting the payload.

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
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex')

  // Constant-time comparison prevents timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  )
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
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex')

  const valid = crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  )

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

Failed deliveries (non-2xx response or timeout) are retried up to 5 times with exponential backoff:

| Attempt | Delay |
|---|---|
| 1 | Immediate |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

After 5 failures the delivery is marked failed and won't be retried automatically. View failed deliveries and retry them manually under Settings > Integrations > Webhooks > Delivery History.

Deliveries are deduplicated: the same event won't be delivered twice to the same endpoint within a 10-minute window, even across retries.

## Testing

Send a test event from the dashboard to verify your endpoint before going live. It uses the same signature mechanism as live events.

Your endpoint must return a 2xx status within 10 seconds. Longer than that counts as a timeout and is treated as a failure.
