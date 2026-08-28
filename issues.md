# Issues

An issue is what Flusterduck produces when repeated friction survives an evidence check. It's a specific, located problem with cited sessions and a lifecycle that tracks whether it got fixed.

Issues work like tickets. They get statuses, triage notes, assignees, and verification after you ship a fix.

## How issues are created

Signals and scores tell the nightly AI investigator where to look. They don't create an issue by themselves while AI detection is enabled. The investigator must open the underlying sessions, check the page and detector evidence, and pass the server's write gate before an issue reaches your board.

Every new AI issue needs at least two opened sessions with events on the claimed page. A cited detector must occur in those sessions, and the investigator must read that detector's interpretation and verification rules. Time-based claims need matching bounded episode receipts from the same detector in both sessions. Severity and confidence are capped by the evidence shown.

If the AI allowance runs out, new filing pauses. Raw signals and known issue counts continue updating, but the scoring path can't write a new issue, change an existing diagnosis, or reopen a verified issue. Sites that explicitly turn AI detection off use the deterministic scoring path instead. See [AI detection](./ai-detection) for the full evidence rules.

## Issue fields

| Field | Description |
|---|---|
| `id` | Unique identifier with `iss_` prefix |
| `title` | Generated description of the problem |
| `page` | Page path where the issue occurs |
| `selector` | CSS selector of the affected element |
| `signal_type` | Dominant signal type driving the cluster |
| `signal_count` | Total signals in the cluster |
| `severity` | `low`, `medium`, `high`, or `critical`. Derived from the page's confusion score, then capped by how many distinct users the issue actually reached, so a single-session blip can't read as critical |
| `status` | Current lifecycle state |
| `hypothesis` | Evidence-checked diagnosis; cause details are omitted when unsupported |
| `revenue_impact` | Estimated monthly cost, when Settings > Revenue has a monthly revenue or average order value configured. See [Revenue impact](./revenue) |
| `sessions` | Session IDs containing this friction pattern |
| `verifications` | Deploy-correlated verification records |

Severity is a volume-checked label, not a raw number. The scoring engine derives it from the page's confusion score (80+ is `critical`, 60+ is `high`, 35+ is `medium`, below that is `low`), then clamps it down based on how many distinct users the cluster actually reached: one affected user caps at `medium`, two to four cap at `high`, and only five or more can read `critical`. A page can score 90 from a single unlucky session; the cap keeps that from reading as your worst problem. The clamp is skipped for a hard technical failure (a captured JS error or 5xx response), since a single user hitting a real server error is worth escalating regardless of reach.

## Issue lifecycle

**`open`**: the issue exists and nobody has acted on it. New issues land here automatically.

**`triaged`**: someone reviewed it and confirmed it's real. Add a note with context before moving it here. "Confirmed on mobile Safari. The submit button is obscured by the cookie banner at 375px." is more useful than just a status change.

**`in_progress`**: someone is actively working a fix. This prevents duplicate effort when multiple engineers are looking at the same issue list.

**`resolved`**: you've shipped a fix and marked it done. This is a claim, not a confirmation. The next deploy's verification cycle checks whether the friction pattern actually declined and moves the issue to `verified` or `regressed` accordingly. Issues sitting in `resolved` without a deploy behind them stay `resolved` until one arrives.

**`verified`**: the issue was resolved and the scoring engine confirmed the fix held after the next deploy. The friction pattern didn't return.

**`regressed`**: the issue was resolved, but fresh investigation found the problem again. Reopening uses the same cited-session and claim checks as every other AI issue update.

**`ignored`**: known issue, won't fix. Alert rules won't fire for ignored issues, and they won't count toward your open issue total. Use this deliberately, not as a way to clear your queue.

You can set an issue's status to `triaged`, `in_progress`, `resolved`, `verified`, or `ignored` directly. `regressed` is set automatically, by the verification cycle, never by hand.

### Duplicate and merged issues

The engine occasionally finds that two issues describe the same underlying friction, usually after a cluster splits or reforms across a deploy. When that happens, one issue absorbs the other: the survivor keeps the evidence and history, and the absorbed issue gets a `merged_into` pointer to the survivor's id. A merged issue stops appearing in your open issue list and stops counting toward alerts or revenue totals on its own; look at the issue it points to instead.

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

Severity starts as a read of the page's confusion score at the moment the cluster formed: 80+ maps to `critical`, 60+ to `high`, 35+ to `medium`, anything below that to `low`. The page score already bakes in signal tier, volume, and 14-day recency decay, so severity inherits all of that for free instead of recomputing it.

That first pass is then capped by how many distinct users the cluster actually reached, because a page score is an intensity, not a headcount, and a single confused visitor can push a low-traffic page's score into the 80s on its own:

| Distinct affected users | Severity ceiling |
|---|---|
| 1 | `medium` |
| 2-4 | `high` |
| 5+ | `critical` (uncapped) |

A hard technical failure, a captured JavaScript error or a 5xx response, skips the cap: one user hitting a real server error on checkout is still worth escalating even if nobody else has hit it yet.

Use severity for relative prioritization within your open issue list, not as a page-independent scale. A `medium` issue on `/checkout` probably matters more than a `high` issue on your admin changelog page.

## Verification

After each deploy, the scoring engine runs verification on all open and recently-resolved issues.

For **resolved** issues: checks whether the signal cluster has declined. If signals dropped, it marks the issue `verified`. If signals are still at pre-fix levels, it marks it `regressed`.

For **open** issues: recalculates evidence, updates `signal_count` and `severity`, and checks whether the pattern is intensifying or fading.

Verification needs deploy records to work correctly. Without them, the engine can't know when to run the verification cycle. See [Deploy Correlation](./deploy-correlation).

## Evidence sessions

Every issue includes a `sessions` array: session IDs where the friction pattern appears. Use them to check the diagnosis and any cause claim.

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

## Diagnosis and hypotheses

A behavioral signal proves the behavior it measured. It doesn't prove the technical cause. Flusterduck rejects causal wording and mechanism-specific repair advice unless the cited evidence supports them. When the evidence supports only the observed pattern, the issue says only what happened.

Bounded page inspection and episode receipts can support narrower facts, including a known response status, a sampled loading-state transition, a mismatched click target, or an overlap that remained after layout activity settled. Unsupported details are left out.

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
      "severity": "critical",
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
