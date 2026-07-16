# Issues

An issue is what Flusterduck produces when enough users hit the same friction pattern on the same element. Not a raw signal count. A clustered, evidence-backed problem with a specific location, a root cause hypothesis, and a lifecycle that tracks whether it got fixed.

Issues work like tickets. They get statuses, triage notes, assignees, and verification after you ship a fix.

## How issues are created

The scoring engine watches for signal clusters: the same signal type on the same element, repeated across sessions from different users. When a cluster crosses the confidence threshold, an issue is created.

One user rage-clicking a button 40 times doesn't create an issue. Forty different users rage-clicking the same button over three days does.

The clustering algorithm weighs:
- Signal type match (same type, or co-occurring types on the same element)
- Element selector match (fuzzy-matched to handle minor DOM changes between deploys)
- Session diversity (signals from distinct sessions, not a single frustrated user)
- Time window (7 days for initial creation, longer for ongoing patterns)

## Issue fields

| Field | Description |
|---|---|
| `id` | Unique identifier with `iss_` prefix |
| `title` | Generated description of the problem |
| `page` | Page path where the issue occurs |
| `selector` | CSS selector of the affected element |
| `signal_type` | Dominant signal type driving the cluster |
| `signal_count` | Total signals in the cluster |
| `severity` | 0-100 score based on signal tier, volume, page importance, and revenue exposure |
| `status` | Current lifecycle state |
| `hypothesis` | Engine-generated root cause guess |
| `sessions` | Session IDs containing this friction pattern |
| `verifications` | Deploy-correlated verification records |

## Issue lifecycle

**`open`**: the issue exists and nobody has acted on it. New issues land here automatically.

**`triaged`**: someone reviewed it and confirmed it's real. Add a note with context before moving it here. "Confirmed on mobile Safari. The submit button is obscured by the cookie banner at 375px." is more useful than just a status change.

**`in_progress`**: someone is actively working a fix. This prevents duplicate effort when multiple engineers are looking at the same issue list.

**`verified`**: the issue was resolved and the scoring engine confirmed the fix held after the next deploy. The friction pattern didn't return.

**`regressed`**: the issue was resolved, but the confusion pattern came back. The engine re-opens it automatically. This happens when a fix gets reverted, a related change reintroduces the problem, or the original fix was partial.

**`ignored`**: known issue, won't fix. Alert rules won't fire for ignored issues, and they won't count toward your open issue total. Use this deliberately, not as a way to clear your queue.

### Moving issues forward

Via the API:

```bash
curl -X POST https://api.flusterduck.com/v1/issues/iss_xxxxxxxxxxxx \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "triaged",
    "note": "Confirmed on mobile. Submit button overlaps cookie banner at 375px.",
    "assigned_to": "maya"
  }'
```

Via MCP:

```
Triage the open issues and tell me what to fix first.
Mark issue iss_xxxxxxxxxxxx as in_progress and assign it to alex.
```

## Severity scoring

Severity (0-100) reflects how much a specific issue is hurting your users. Higher severity means more sessions affected, higher-tier signals, more revenue exposure.

It incorporates:
- Signal tier of the dominant signal type
- Signal volume and percentage of sessions affected
- Page importance (checkout and pricing have higher baseline weights)
- Revenue exposure (whether conversion events were tracked near this element in affected sessions)
- Recency (signals from the last 48 hours weight more than older ones)

Use severity for relative prioritization within your open issue list. Don't treat it as an absolute scale. A severity 40 issue on /checkout probably matters more than a severity 70 issue on your admin changelog page.

## Verification

After each deploy, the scoring engine runs verification on all open and recently-resolved issues.

For **resolved** issues: checks whether the signal cluster has declined. If signals dropped, it marks the issue `verified`. If signals are still at pre-fix levels, it marks it `regressed`.

For **open** issues: recalculates evidence, updates `signal_count` and `severity`, and checks whether the pattern is intensifying or fading.

Verification needs deploy records to work correctly. Without them, the engine can't know when to run the verification cycle. See [Deploy Correlation](./deploy-correlation).

## Evidence sessions

Every issue includes a `sessions` array: session IDs where the friction pattern appears. These are the most useful thing in an issue for diagnosing root cause.

```bash
curl "https://api.flusterduck.com/v1/session?session_id=ses_xxxxxxxxxxxx" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

The session endpoint returns the full chronological event timeline: page views, signal types, selectors, timestamps. You can see exactly what the user did before and after the friction moment.

Via MCP:

```
Investigate session ses_xxxxxxxxxxxx.
Which sessions show rage clicks on the upgrade button?
```

## Root cause hypotheses

The engine generates a hypothesis for every issue based on signal type and element context:

- Dead clicks on a button: "Element appears clickable but navigates to an unexpected destination or produces no visible response."
- Rage clicks on a form field: "Field appears interactive but input is blocked or significantly delayed."
- Form abandonment at a specific step: "Required information may be unclear, the step label may not match user expectation, or a validation message is obscured."

Hypotheses are starting points. They're generated from behavioral signals, not from visual inspection of your UI. Confirm or reject them with the session evidence and your own knowledge of the page.

## Getting all issues via the API

```bash
curl "https://api.flusterduck.com/v1/issues?status=open" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

```json
{
  "issues": [
    {
      "id": "iss_xxxxxxxxxxxx",
      "title": "Dead clicks on complete purchase button",
      "page": "/checkout",
      "selector": "button[type='submit']",
      "signal_type": "dead_click",
      "signal_count": 47,
      "severity": 84,
      "status": "open",
      "created_at": "2026-06-08T11:30:00Z"
    }
  ],
  "total": 1
}
```

Filter by any status: `open`, `triaged`, `in_progress`, `verified`, `resolved`, `regressed`, `ignored`.

## Getting full issue detail

```bash
curl "https://api.flusterduck.com/v1/issues/iss_xxxxxxxxxxxx" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

Returns the full issue object including `hypothesis`, `sessions`, and `verifications`.

## What issues don't cover

Issues require signal clusters from multiple users. Single-session bugs, one-off failures, and problems that affect a small percentage of users on a specific device or browser may not generate enough signal volume to create an issue automatically.

For those cases, use `signal()` manually to attach extra metadata to auto-detected signals. The more context you give the engine, the faster it can cluster related signals:

```ts
signal('error_recovery_loop', {
  form: 'payment',
  step: 'card-entry',
  error_code: 'card_declined',
})
```

Custom metadata gets attached to the issue evidence when a cluster forms.
