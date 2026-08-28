# REST API Reference

Base URL: `https://api.flusterduck.com/v1`

Read routes live under `/query`, write routes under `/manage`.

## Authentication

Pass your key in the Authorization header:

```
Authorization: Bearer fd_sec_your_key_here
```

`fd_sec_` keys work for read endpoints with the `query:read` scope and for write endpoints with the `manage:write` scope. A user JWT issued during login works everywhere, same header format. Browser code must never use `fd_sec_` keys.

## Rate limits

| Surface | Limit |
|---|---|
| Read API (`/query`) | 100 requests/minute per org |
| Write API (`/manage`) | 30 requests/minute per org |

Responses over the limit return `429` with a `reset_at` timestamp in `error.details` telling you when the window opens again.

## Response envelope

Every JSON response is wrapped the same way. Success:

```json
{ "data": { "...": "..." }, "error": null }
```

Failure:

```json
{
  "data": null,
  "error": {
    "code": "not_found",
    "message": "UX issue not found",
    "status": 404
  }
}
```

`error.status` repeats the HTTP status code. An optional `error.details` field carries extra context when there is any. Every success response is wrapped this way too, with your payload under `data` and `error` set to `null`. The read examples below show the full envelope; the write examples focus on the request body.

---

## Read endpoints

### GET /query/scores

All tracked pages with their current confusion scores.

```bash
curl "https://api.flusterduck.com/v1/query/scores?site_id=your-site-id" \
  -H "Authorization: Bearer fd_sec_xxxx"
```

```json
{
  "data": {
    "site": {
      "site_id": "your-site-id",
      "overall_score": 34.2,
      "overall_trend": "up",
      "pages": [
        {
          "page": "/checkout",
          "score": 72,
          "trend": "up",
          "dominant_signal": "rage_click"
        }
      ],
      "total_active_users": 1204,
      "total_affected_users": 87
    },
    "duck_state": "watching"
  },
  "error": null
}
```

`trend` is `up`, `down`, or `stable` relative to the 7-day baseline. Each page row carries the full `page_scores` record, so it has more fields than shown here.

### GET /query/page

Full data for one page: current score, score history, element friction, open issues, recent deploys, active alerts, annotations, and confusion budgets.

```bash
curl "https://api.flusterduck.com/v1/query/page?site_id=your-site-id&page=/checkout" \
  -H "Authorization: Bearer fd_sec_xxxx"
```

Returns `score`, `history`, `elements`, `active_alerts`, `issues`, `recent_deploys`, `annotations`, and `budgets`.

### GET /query/issues

UX issues, filterable by status, with offset paging.

```bash
curl "https://api.flusterduck.com/v1/query/issues?site_id=your-site-id&status=open&limit=20" \
  -H "Authorization: Bearer fd_sec_xxxx"
```

```json
{
  "data": {
    "issues": [
      {
        "id": "9be2f9d4-3c1a-4e8b-9d2f-6a7b5c4d3e2f",
        "title": "Dead clicks on upgrade CTA",
        "page": "/pricing",
        "element_selector": "[data-cta='upgrade']",
        "type": "dead_click",
        "severity": "high",
        "status": "open"
      }
    ],
    "total": 1,
    "offset": 0,
    "limit": 20
  },
  "error": null
}
```

The issue's friction category is `type` (values like `dead_click`, `rage_click`, `form_friction`), not `signal_type`. `severity` is `low`, `medium`, `high`, or `critical`. Each row carries the full issue record, so expect more fields than shown. Status options: `open`, `triaged`, `in_progress`, `verified`, `resolved`, `ignored`, `regressed`

### GET /query/issues/{issue_id}

Full detail on one issue: evidence, linked sessions, verification history, and deploy correlation.

```bash
curl "https://api.flusterduck.com/v1/query/issues/9be2f9d4-...?site_id=your-site-id" \
  -H "Authorization: Bearer fd_sec_xxxx"
```

### GET /query/alerts

```bash
curl "https://api.flusterduck.com/v1/query/alerts?site_id=your-site-id&status=fired" \
  -H "Authorization: Bearer fd_sec_xxxx"
```

Returns `alerts` plus `total`, `offset`, and `limit`. Status options: `fired`, `acknowledged`, `investigating`, `resolved`. Fetch one alert with its incident timeline at `GET /query/alerts/{alert_id}`.

### GET /query/trends

Confusion score history. `days` accepts 1-90, defaults to 7. Add `page` to scope to one path.

```bash
curl "https://api.flusterduck.com/v1/query/trends?site_id=your-site-id&page=/checkout&days=14" \
  -H "Authorization: Bearer fd_sec_xxxx"
```

Returns `history`: score_history rows, newest first.

### GET /query/deploys

```bash
curl "https://api.flusterduck.com/v1/query/deploys?site_id=your-site-id" \
  -H "Authorization: Bearer fd_sec_xxxx"
```

Returns `deploys` (with `confusion_before` and `confusion_after` per deploy) plus `total`, `offset`, and `limit`. `confusion_after` is `null` until the scoring engine has enough post-deploy data (typically 5 minutes). Fetch one deploy with its related issues at `GET /query/deploys/{deploy_id}`.

### GET /query/session

Full event timeline for a session.

```bash
curl "https://api.flusterduck.com/v1/query/session?site_id=your-site-id&session_id=f3a9c1d24e8b76051a2b3c4d5e6f7081" \
  -H "Authorization: Bearer fd_sec_xxxx"
```

Session ids are 32-character hex strings, no prefix. Returns `session` (timing, device, pages visited, signal counts), `events` (every event in order), and a plain-language `narrative`.

### Other read endpoints

| Endpoint | Returns |
|---|---|
| `GET /query/elements?page=/checkout` | Element-level friction breakdown |
| `GET /query/element-heatmap?page=/checkout&selector=button.submit` | Click positions within one element |
| `GET /query/page-heatmap?page=/checkout` | Page-level friction click map |
| `GET /query/flows` | Page-to-page navigation edges |
| `GET /query/compare?a=/pricing&b=/checkout` | Side-by-side score comparison |
| `GET /query/recommendations` | Prioritized fix list |
| `GET /query/revenue` | Revenue impact estimates (requires conversion tracking) |
| `GET /query/raw?table=signals&limit=100` | Raw table rows |
| `GET /query/export/events.csv` | CSV export of event rows |
| `GET /query/mcp/context` | Full site context snapshot |

All take `site_id` unless the key is scoped to a single site. See the [REST API guide](./rest-api) for the complete route list with parameters.

---

## Write endpoints

Write endpoints require `Content-Type: application/json` and either a user JWT or a key with the `manage:write` scope.

### PATCH /manage/issues/{issue_id}

Update status, add a note, or assign.

```bash
curl -X PATCH https://api.flusterduck.com/v1/manage/issues/9be2f9d4-... \
  -H "Authorization: Bearer fd_sec_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "triaged",
    "note": "Confirmed on Safari iOS 17. Related to the disabled-state border color.",
    "assigned_to": "alex"
  }'
```

Status options: `open`, `triaged`, `in_progress`, `verified`, `resolved`, `ignored`

### PATCH /manage/alerts/{alert_id}

```json
{
  "status": "resolved",
  "resolved_reason": "Deploy #4421 fixed the broken CTA selector"
}
```

Status options: `acknowledged`, `investigating`, `resolved`

### POST /manage/annotations

```json
{
  "message": "Redesigned checkout flow, monitoring score"
}
```

### POST /manage/alert-rules

```json
{
  "name": "Checkout rage click spike",
  "trigger_type": "spike",
  "threshold": 25,
  "cooldown_minutes": 60,
  "channels": ["email", "slack"],
  "page_pattern": "/checkout*",
  "slack_channel": "#incidents",
  "enabled": true
}
```

See [Alerts](./alerts) for all trigger types and channel options.

### PATCH /manage/alert-rules/{rule_id}

Partial update. Use `"enabled": false` to silence a rule without deleting it.

```json
{
  "enabled": false
}
```

### DELETE /manage/alert-rules/{rule_id}

Permanent. Use PATCH with `enabled: false` if you might want it back.

---

## Errors

Errors use the same envelope as every response, with `data: null` and a populated `error` object:

```json
{
  "data": null,
  "error": {
    "code": "not_found",
    "message": "Issue not found or does not belong to this site",
    "status": 404
  }
}
```

`error.details` appears only when the error carries extra context (for example, which field failed validation).

`error.code` is a stable string you can match on. It's specific to what went wrong, so one status can carry several codes. The common ones:

| Status | Codes you'll see | Meaning |
|---|---|---|
| 400 | `invalid_uuid`, `missing_site_id`, `invalid_status`, `empty_body`, `invalid_json` | Missing or malformed input |
| 401 | `missing_auth`, `invalid_key`, `expired_key`, `invalid_jwt`, `jwt_required` | No valid credential |
| 403 | `forbidden`, `missing_scope`, `insufficient_role`, `org_scope_required` | Authenticated, but not allowed to do this |
| 404 | `not_found` | Resource doesn't exist or belongs to another site |
| 429 | `rate_limited` | Too many requests. `error.details.reset_at` says when the window reopens |
| 500 | `internal_error` | Something broke on our end |
