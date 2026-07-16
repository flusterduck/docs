# REST API

Every piece of data in Flusterduck is accessible through two HTTP endpoints. The query endpoint reads data. The manage endpoint writes it. Both accept a secret key in the Authorization header, so you can call them from scripts, CI pipelines, internal tools, or any language with an HTTP client.

## Authentication

Create a secret key in Settings > API Keys. It starts with `fd_sec_`. Pass it as a Bearer token:

```
Authorization: Bearer fd_sec_your_key_here
```

The key must have the right scopes. A key with `query:read` can read scores, issues, alerts, deploys, and raw data. A key with `manage:write` can create sites, update alert rules, triage issues, and manage members. You set scopes when creating the key.

MCP keys (`fd_mcp_`) also work for read operations.

## Base URL

```
https://api.flusterduck.com/v1
```

All routes are under `/query` (read) or `/manage` (write).

## Reading data

Query routes take a `site_id` query parameter. If your key is scoped to a single site (most are), you can omit it: the route uses the key's own site. Keys scoped to the whole org must say which site they mean.

Not sure what a key can see? Ask it:

```
GET /query/whoami
```

Returns the key's org, site, and scopes. Handy in setup scripts to validate a key before using it.

### Scores

```
GET /query/scores?site_id={id}
```

Returns confusion scores for every page on the site, sorted by score descending.

### Issues

```
GET /query/issues?site_id={id}
GET /query/issues?site_id={id}&status=open&limit=20
GET /query/issues/{issue_id}?site_id={id}
```

List or fetch individual UX issues. Filter by status: `open`, `triaged`, `in_progress`, `verified`, `resolved`, `ignored`.

### Alerts

```
GET /query/alerts?site_id={id}
GET /query/alerts?site_id={id}&status=fired
GET /query/alerts/{alert_id}?site_id={id}
```

### Deploys

```
GET /query/deploys?site_id={id}
GET /query/deploys/{deploy_id}?site_id={id}
```

Returns deploys with confusion_before and confusion_after scores, plus related issues.

### Page detail

```
GET /query/page?site_id={id}&page=/checkout
```

Full detail for a single page: score, history, top elements, open issues, recent deploys, active alerts.

### Elements

```
GET /query/elements?site_id={id}&page=/checkout
```

Top elements by friction signal count on a page. Includes dominant signal type, affected users, and recommendation.

### Element heatmap

```
GET /query/element-heatmap?site_id={id}&page=/checkout&selector=button.submit
```

Returns up to 300 click positions on a single element, collected from `rage_click`, `dead_click`, and `disabled_element_attempt` signals. Both `page` and `selector` are required.

Each point uses a **0 to 1 coordinate system** relative to the element's bounding box. `rx: 0, ry: 0` is the top-left corner. `rx: 1, ry: 1` is the bottom-right. No pixel coordinates ever leave the browser.

The SDK records these positions in two ways. Newer builds emit `hx`/`hy` as integers from 0 to 1000 (divided by 1000 on the server to get the 0-1 fraction). Older builds emit a center-relative offset (`click_dx`/`click_dy`) plus the element dimensions (`el_w`/`el_h`), and the server recovers the fraction from those. You don't need to worry about which format your SDK uses. The API always returns normalized `rx`/`ry` values.

```json
{
  "data": {
    "page": "/checkout",
    "selector": "button.submit",
    "count": 47,
    "points": [
      { "rx": 0.512, "ry": 0.483, "signal": "rage_click" },
      { "rx": 0.891, "ry": 0.102, "signal": "dead_click" },
      { "rx": 0.334, "ry": 0.721, "signal": "disabled_element_attempt" }
    ]
  },
  "error": null
}
```

| Field | Type | Description |
|---|---|---|
| `rx` | number | Horizontal position within the element, 0 (left edge) to 1 (right edge) |
| `ry` | number | Vertical position within the element, 0 (top edge) to 1 (bottom edge) |
| `signal` | string | The signal type that generated this point |

To render a heatmap overlay, multiply `rx` by the element's rendered width and `ry` by its rendered height. The positions stay accurate at any zoom level or screen size because they're fractions, not pixels.

### Page heatmap

```
GET /query/page-heatmap?site_id={id}&page=/checkout
```

Returns up to 500 friction positions across the entire viewport for a page. Same signal types as the element heatmap (`rage_click`, `dead_click`, `disabled_element_attempt`), but coordinates are relative to the viewport instead of a single element.

The SDK records `vx`/`vy` as integers from 0 to 1000 (the click's position as a fraction of the viewport width and height). The server normalizes these to 0-1 before returning them.

```json
{
  "data": {
    "page": "/checkout",
    "count": 183,
    "points": [
      { "vx": 0.724, "vy": 0.312, "signal": "rage_click", "el": "button.submit" },
      { "vx": 0.501, "vy": 0.887, "signal": "dead_click", "el": "a.terms-link" },
      { "vx": 0.223, "vy": 0.145, "signal": "dead_click" }
    ]
  },
  "error": null
}
```

| Field | Type | Description |
|---|---|---|
| `vx` | number | Horizontal position in the viewport, 0 (left) to 1 (right) |
| `vy` | number | Vertical position in the viewport, 0 (top) to 1 (bottom) |
| `signal` | string | The signal type that generated this point |
| `el` | string or absent | CSS selector of the element that was clicked, when available |

Points without an `el` field come from signals where the element couldn't be identified. This is rare but possible with dynamically removed DOM nodes.

### Friction map

```
GET /query/journeys/friction?site_id={id}
GET /query/journeys/friction?site_id={id}&min_friction_weight=5&signal_type=rage_click&limit=500
```

Returns page-to-page navigation edges weighted by the friction signals that occurred along them. This powers the friction map panel in the dashboard, showing where users hit trouble as they move through your site.

The endpoint joins session navigation paths with signals that qualify as edge decorators (the friction-relevant subset of all signal types). Each edge accumulates a `friction_weight` from the weight and confidence of every qualifying signal on either the source or destination page.

| Parameter | Default | Description |
|---|---|---|
| `limit` | 250 | Max sessions to analyze (1 to 1000) |
| `min_friction_weight` | 1 | Minimum combined friction weight for an edge to appear (0 to 1000) |
| `signal_type` | all | Filter to a single signal type, e.g. `rage_click` |

```json
{
  "data": {
    "site_id": "your-site-id",
    "edges": [
      {
        "from": "/pricing",
        "to": "/checkout",
        "sessions": 34,
        "friction_weight": 127.4,
        "signals": {
          "rage_click": 18,
          "dead_click": 12,
          "form_restart": 4
        },
        "examples": [
          {
            "session_id": "sess_abc123",
            "page": "/checkout",
            "signal_type": "rage_click",
            "element_selector": "button.submit",
            "occurred_at": "2026-06-21T14:32:01Z"
          }
        ]
      },
      {
        "from": "/checkout",
        "to": "/checkout",
        "sessions": 12,
        "friction_weight": 89.2,
        "signals": {
          "form_restart": 9,
          "error_encounter": 3
        },
        "examples": []
      }
    ]
  },
  "error": null
}
```

| Field | Type | Description |
|---|---|---|
| `from` | string | Source page path |
| `to` | string | Destination page path. Same as `from` when users reload or loop on one page. |
| `sessions` | number | Distinct sessions that traversed this edge with friction |
| `friction_weight` | number | Combined weight of all qualifying signals, rounded to one decimal |
| `signals` | object | Count of each signal type observed on this edge |
| `examples` | array | Up to 5 concrete signal instances for debugging. Each has `session_id`, `page`, `signal_type`, `element_selector`, and `occurred_at`. |

Results are sorted by `friction_weight` descending and capped at 100 edges. Edges with a `friction_weight` below `min_friction_weight` are excluded before sorting.

### Trends

```
GET /query/trends?site_id={id}&days=7
GET /query/trends?site_id={id}&page=/checkout&days=30
```

Score history over time. Defaults to 7 days.

### Revenue

```
GET /query/revenue?site_id={id}
```

Revenue at risk across all open issues, based on the revenue config you set in Settings.

### Recommendations

```
GET /query/recommendations?site_id={id}
```

Ranked list of fix recommendations derived from the current issue set.

### Raw data

```
GET /query/raw?site_id={id}&table=signals&limit=100&sort=occurred_at&order=desc
GET /query/raw?site_id={id}&table=events&limit=50
GET /query/raw?site_id={id}&table=sessions
GET /query/raw?site_id={id}&table=page_scores
GET /query/raw?site_id={id}&table=ux_issues
```

Direct table access with sorting and pagination. Available tables: `events`, `signals`, `sessions`, `page_scores`, `score_history`, `ux_issues`, `deploys`, `alerts`.

### CSV export

```
GET /query/export/events.csv?site_id={id}
```

Downloads raw events as CSV.

### MCP context

```
GET /query/mcp/context?site_id={id}
```

A single-call summary designed for AI assistants: top scores, open issues, recent deploys, active alerts, and recommendations in one response.

## Writing data

Manage routes use POST, PATCH, or DELETE. All require a key with `manage:write` scope.

### Sites

```
POST   /manage/site        { name, url }
PATCH  /manage/site/{id}   { name, url }
DELETE /manage/site/{id}
```

### Alert rules

```
GET    /manage/alert-rules?site_id={id}
POST   /manage/alert-rules  { site_id, trigger_type, threshold, channels, config }
PATCH  /manage/alert-rules/{id}  { threshold, channels, enabled }
DELETE /manage/alert-rules/{id}
```

Trigger types: `spike`, `anomaly`, `new_page`, `trend`, `co_occurrence`, `positive`, `budget`.
Channels: `email`, `slack`, `webhook`, `mcp`, `pagerduty`.

### Issues

```
PATCH /manage/issues/{id}  { status, assigned_to, severity }
```

Triage, assign, and resolve issues.

### Members

```
GET    /manage/members
POST   /manage/members  { email, role }
PATCH  /manage/members/{id}    { role }
DELETE /manage/members/{id}
```

Roles: `owner`, `admin`, `member`, `viewer`.

### Webhooks

```
GET    /manage/webhooks?site_id={id}
POST   /manage/webhooks      { site_id, url, events, secret }
PATCH  /manage/webhooks/{id} { url, events, enabled }
DELETE /manage/webhooks/{id}
```

### SDK config

```
GET   /manage/sdk-config?site_id={id}
PATCH /manage/sdk-config/{environment}  { revenue_config, ... }
```

### Integrations

```
GET    /manage/integrations?site_id={id}
POST   /manage/integrations  { site_id, provider, config }
DELETE /manage/integrations/{id}
```

Providers: `slack`, `pagerduty`, `linear`, `github`.

### API keys

```
GET    /manage/keys?site_id={id}
POST   /manage/keys  { site_id, key_type, scopes }
DELETE /manage/keys/{id}
```

Key types: `secret`, `mcp`. Scopes: `ingest:write`, `query:read`, `manage:write`, `mcp:read`, `webhook:write`.

## Rate limits

| Endpoint | Limit |
|---|---|
| query | 100 RPM per org |
| manage | 30 RPM per org |
| guide | 200 RPM per site |
| ingest | 10,000 RPM per publishable key |

## Response format

All responses wrap data in `{ "data": { ... }, "error": null }`. On error, `data` is null and `error` carries a machine-readable `code`, a human-readable `message`, and the HTTP `status` repeated in the body. A `details` field appears only when the error has extra context (for example, which field failed validation):

```json
{
  "data": null,
  "error": {
    "code": "invalid_status",
    "message": "Invalid alert status",
    "status": 400
  }
}
```

Want a little cheer on a bad day? Error responses can carry a `duck` field with an unrelated duck fact. It is off by default and never changes the error itself. Turn it on for your whole workspace with the Duck facts toggle in Settings, or control it per request: `?ducks=on` / `?ducks=off` (or the `x-fd-ducks` header) always wins over the workspace setting.

## /fun

No auth, no data, no rate limit. A GET returns a little note about our duck, and sometimes a real duck fact rides along.

```
GET /fun
```

```json
{ "data": { "duck": "The duck reads every rage click so you never have to.", "fact": "A group of ducks resting on water is called a raft." }, "error": null }
```
