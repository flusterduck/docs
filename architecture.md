# Architecture overview

Flusterduck runs entirely on edge functions. The browser SDK collects behavioral signals, sends them to an ingestion endpoint, and a server-side pipeline turns raw events into scores, issues, and alerts. Your frontend never talks to the database.

This page maps the full data flow for teams doing compliance reviews, integration planning, or just wanting to know where their data goes.

## The pipeline

Every piece of friction data moves through five stages:

1. **SDK** collects behavioral signals in the browser
2. **ingest** validates, deduplicates, and stores events
3. **compute-scores** calculates per-page confusion scores
4. **process-issues** clusters signals into UX issues
5. **process-alerts** evaluates alert rules and queues notifications

After alerts fire, two delivery paths carry notifications out: the **webhooks** function delivers signed HTTP payloads to your endpoints, and the **slack** function handles slash commands and interactive buttons.

A sixth function, **deploy-correlation**, sits outside the main pipeline. It records deploys and captures before/after confusion snapshots so the engine can verify whether a fix worked.

## How data enters

The SDK batches behavioral events and POSTs them to the `ingest` edge function. Each batch contains a session ID, the publishable key (`fd_pub_`), the current URL, viewport dimensions, and up to 250 events.

```ts
// What the SDK sends (simplified)
{
  sid: "session-uuid",
  key: "fd_pub_xxxxxxxxxxxx",
  url: "https://yourapp.com/checkout",
  ts: 1719014400000,
  events: [
    { type: "signal", signal_type: "dead_click", element: "button.submit", ts: 1719014399800 },
    { type: "signal", signal_type: "rage_click", element: "button.submit", ts: 1719014399950 }
  ]
}
```

The SDK runs client-side only. It attaches listeners for clicks, scroll, keyboard, touch, form focus/blur, navigation, and errors. It doesn't attach listeners that would capture keystrokes, form values, or page text. That constraint is enforced at the code level.

## ingest

The first edge function in the chain. It does six things, in order:

1. Validates the `fd_pub_` key against the `api_keys` table using HMAC-SHA256 lookup
2. Enforces rate limits (10,000 requests per minute per publishable key)
3. Checks session limits against the org's billing plan
4. Upserts the session record with device type, browser, country, and screen bucket
5. Normalizes each event, hashes the IP address, generates a dedupe key, strips sensitive fields, and inserts into the `events` table
6. Extracts typed signals from events, assigns weights from the signal registry, and inserts into the `signals` table

IP addresses are hashed with SHA-256 plus `IP_HASH_SALT` and truncated to 32 hex characters before storage. The raw IP never hits the database.

After writing signals, ingest fires an async call to `compute-scores`. It doesn't wait for a response. The browser gets back `{ ok: true, accepted: 12, rejected: 0 }` within milliseconds.

## compute-scores

Runs on a 15-minute window by default. For each active site, it:

1. Loads all signals and sessions from the scoring window
2. Groups signals by page and signal type
3. Calculates a weighted score per page: `sum(signal_weight * confidence) / active_sessions`
4. Compares the raw score against the page's baseline (rolling mean and standard deviation)
5. Normalizes the score to a 0-100 scale using z-score math: `((raw - mean) / std_dev) * 25 + 50`
6. Writes scores to `page_scores` and appends to `score_history`

Each score row includes a `duck_state` (calm, uneasy, frustrated, panicking), a trend direction, the dominant signal type, a signal breakdown with percentages, and a co-occurrence multiplier that boosts the score when many different users hit the same friction.

The baseline requires at least 7 samples before it kicks in. Until then, scores use raw values.

After scoring, `compute-scores` also back-fills `confusion_after` on recent deploys. Any deploy older than 5 minutes but younger than 48 hours gets a snapshot of current page scores, so deploy-correlation can compare before and after.

## process-issues

Runs on a 60-minute window. This is where individual signals become actionable issues.

The function clusters signals by a composite key: `page + signal_type + element`. If a cluster has at least 3 fires or spans 2+ sessions, it becomes a UX issue.

For each cluster, process-issues:

1. Generates a fingerprint hash from `site_id:page:signal:element`
2. Looks up existing open issues with the same fingerprint
3. Updates the existing issue if found, or creates a new one
4. Calculates issue confidence from signal count, session breadth, and per-fire confidence
5. Estimates revenue impact using the site's revenue config (if `track()` is wired)
6. Writes signal evidence rows linking the issue to its source events

Issues have a lifecycle: `open`, `triaged`, `in_progress`, `verified`, `resolved`, `ignored`, `regressed`. If new signals match an issue that was previously `verified`, its status flips to `regressed`. That's how you know a fix didn't hold.

## process-alerts

Evaluates alert rules against the latest page scores. Each rule has a trigger type:

| Trigger | Fires when |
|---|---|
| `budget` | Score exceeds threshold |
| `spike` | Score jumps above baseline by threshold amount |
| `anomaly` | Deviation factor exceeds threshold (encoded as threshold/10) |
| `trend` | Score trend is "up" and score exceeds threshold |
| `new_page` | Page has no baseline and score exceeds threshold |
| `co_occurrence` | Multiple users affected and count exceeds threshold |
| `positive` | Score drops below threshold (for tracking improvements) |

Before firing, the function claims a cooldown lock using an atomic database RPC. Two overlapping runs can't fire the same alert twice. Cooldown is configurable per rule, from 1 minute to 24 hours.

When an alert fires, four things happen:

1. An alert row is created with severity (warning, high, critical), score snapshot, and signal breakdown
2. An incident event is logged
3. Delivery queue rows are created for each configured channel (email, Slack, webhook, PagerDuty)
4. Webhook delivery rows are created for any registered endpoints subscribed to `alert.fired`

## webhooks

Processes the `webhook_deliveries` queue. For each pending delivery:

1. Claims the delivery with a processing lock (stale locks auto-expire after 10 minutes)
2. Looks up the endpoint and decrypts its stored secret
3. Signs the payload with HMAC-SHA256, including a timestamp for replay protection
4. POSTs to the endpoint URL with an 8-second timeout
5. On success, marks the delivery as sent and resets the endpoint's failure counter
6. On failure, schedules a retry with exponential backoff (30s, 60s, 120s, ... up to 1 hour)

Deliveries are retried up to 10 times. After 10 failures, the delivery is marked `dead`.

Every webhook request includes these headers:

```
x-flusterduck-event: alert.fired
x-flusterduck-delivery: delivery-uuid
x-flusterduck-timestamp: 1719014400
x-flusterduck-signature: sha256=...
```

## deploy-correlation

Sits outside the scoring pipeline. You call it by POSTing to the `deploy-correlation` endpoint with an `fd_sec_` key.

When a deploy is recorded:

1. The current `page_scores` are captured as `confusion_before`
2. A deploy row is written with commit hash, author, PR number, and provider
3. Verification records are created for every open issue on the site

Later, when `compute-scores` runs, it fills in `confusion_after` on deploys that are at least 5 minutes old. Comparing the two snapshots tells you whether confusion went up or down after the deploy.

The build plugins for Vite and webpack call this endpoint automatically at the end of each production build.

## slack

Handles two interaction types:

1. **Slash commands** (`/flusterduck scores`, `/flusterduck issues`, `/flusterduck help`) that query live data and respond ephemerally
2. **Block actions** from interactive buttons on alert messages (acknowledge, resolve)

All requests are verified against Slack's signing secret with a 5-minute timestamp tolerance.

## Read and write APIs

Two more edge functions handle all external data access:

**query** is the read API. It accepts user JWTs, `fd_sec_` secret keys, or `fd_mcp_` MCP keys. Routes include scores, issues, alerts, deploys, trends, sessions, recommendations, revenue, raw event export, and MCP context. Rate limited to 100 requests per minute per org.

**manage** is the write API. JWT only, no API keys. It handles CRUD for orgs, sites, environments, API keys, SDK configs, alert rules, issues, members, webhooks, integrations, and billing. Rate limited to 30 requests per minute per org. Admin-only routes verify the user's role before proceeding.

## What the frontend can and can't do

The frontend app (`apps/web`) uses the Supabase client for one thing: authentication. Every `supabase.auth.*` call is allowed. Every data query goes through the `query` edge function. Every write goes through `manage`.

This is enforced by RLS. All 23 tables have row-level security enabled. The `anon` role is revoked on all data tables. The `authenticated` role is revoked on service-only tables (alert delivery queue, Slack messages, scheduled tasks, Stripe events, waitlist). The only way to read or write data is through the edge functions, which authenticate every request and verify org membership before touching a row.

## Key types

| Prefix | Where it's used | What it can do |
|---|---|---|
| `fd_pub_` | Browser SDK | Send signals to ingest. Nothing else. |
| `fd_sec_` | Your server | Read data via query API, record deploys |
| `fd_mcp_` | AI assistants | Read data via MCP bridge, scoped per key |

All keys are stored as HMAC-SHA256 hashes. The raw key is shown once at creation and never stored.

## Data flow diagram

```
Browser SDK
    |
    | POST /ingest (fd_pub_ key, event batch)
    v
[ingest] --> events table, signals table, sessions table
    |
    | async invoke
    v
[compute-scores] --> page_scores table, score_history table
    |                   |
    |                   | back-fills confusion_after on recent deploys
    |                   v
    |               deploys table
    | async invoke
    v
[process-issues] --> ux_issues table, issue_signal_evidence table
    |
    v
[process-alerts] --> alerts table, alert_delivery_queue, webhook_deliveries
    |                   |
    +-------------------+
    |                   |
    v                   v
[webhooks]          [slack]
    |                   |
    v                   v
Your endpoint       Slack channel
```

## Where things run

| Component | Runtime | Location |
|---|---|---|
| Browser SDK | Browser JS | Your users' browsers |
| Edge functions | Deno (Supabase) | Supabase edge network |
| Database | PostgreSQL 15 | Supabase managed |
| Frontend shell | Next.js App Router | Vercel edge |
| MCP worker | Cloudflare Worker | Cloudflare edge |

The `apps/web` frontend on Vercel serves the auth callback, middleware, an `/api/v1` proxy to edge functions, and a `/d.js` redirect to the CDN-hosted SDK script. It doesn't render a dashboard UI directly. The proxy layer means your frontend code never needs to know about Supabase URLs.
