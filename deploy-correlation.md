# Deploy Correlation

Flusterduck needs to know when you shipped. Without it, the engine can't tell a friction spike caused by a broken release from one caused by a traffic surge or a bad email campaign, it can't verify that your fixes worked, and it reads your code at whatever is on your default branch today instead of at the commit that was live when the friction happened.

## If your repository is connected, this is already done

Connect a repository under Settings, Sites and Flusterduck watches it. Nothing to install and nothing to run in CI.

It reads two things, in this order.

1. **GitHub deployments.** Vercel, Netlify, Render, Fly and most deploy actions create one of these on every ship. A deployment whose latest status is `success`, in a production environment, is a real deployment: something told us this went live.
2. **Commits on your default branch.** Used when there are no GitHub deployments to read. This is a commit, not a deployment. For the many teams whose `main` branch is what runs in production it points at the same code, but nothing has confirmed that it shipped.

Those are two different things and Flusterduck keeps them apart. Which one produced a given record is stored on it, and one place treats them differently: an issue can only be automatically retracted on the evidence that your source does **not** contain something, and that requires a real deployment. A commit is not enough to overrule a problem a visitor actually hit.

A deployment that failed is not recorded. Neither is a preview environment. Nothing is recorded twice: if you also run the CLI or post to the API, the commit is matched and you get one deploy, not two.

If your default branch is **not** what you ship, turn watching off on the source card in Settings, Sites and record deploys yourself with either method below. The repository stays connected for everything else.

For the fastest path, subscribe the Flusterduck GitHub App to the `deployment_status` and `push` events and grant it `deployments: read`. Deploys then land within seconds instead of at the next hourly check.

## Recording a deploy yourself

```bash
curl -X POST https://api.flusterduck.com/v1/deploys \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "commit_hash": "a1b2c3d4e5f6",
    "environment": "production"
  }'
```

There's no `version` field. Identify the deploy with `commit_hash` (what shows up as the pinned commit for code evidence) or, for a provider that doesn't give you a commit SHA, `external_deploy_id`. The full set of optional fields:

| Field | Type | Notes |
|---|---|---|
| `environment` | string | Defaults to `production` if omitted |
| `commit_hash` | string | The commit that's live. Used for code evidence and cross-provider dedup |
| `external_deploy_id` | string | Your own deploy identifier, if you don't have a commit hash to hand |
| `commit_message` | string | Shown alongside the deploy in the dashboard |
| `author` | string | Who shipped it |
| `pr_number` | string | Linked pull request, if any |
| `provider` | string | One of `github`, `vercel`, `gitlab`, `bitbucket`, `custom`. Defaults to `custom` |
| `metadata` | object | Free-form, stored as-is |
| `deployed_at` | string | ISO timestamp. Defaults to now |

A deploy is deduplicated within a site by `(provider, external_deploy_id)`, so retried webhook calls for the same deploy upsert instead of creating duplicates. Set `external_deploy_id` even when you also pass `commit_hash`; without it, every call with the same commit still creates a distinct row.

The engine captures `confusion_before` at the moment of the call. After 5 or more minutes of post-deploy traffic, it calculates `confusion_after` and `delta`. Both fields are `null` until that data accumulates.

`delta` is `confusion_after - confusion_before`. Positive means friction went up. Negative means it went down.

## CI integration

Run the record call after your deploy is live and serving traffic, not before. Calling it before traffic hits means `confusion_before` will be accurate but `confusion_after` will include signals from the old version.

### GitHub Actions

```yaml
- name: Record deploy
  run: |
    curl -X POST https://api.flusterduck.com/v1/deploys \
      -H "Authorization: Bearer ${{ secrets.FLUSTERDUCK_SECRET_KEY }}" \
      -H "Content-Type: application/json" \
      -d "{\"commit_hash\": \"${{ github.sha }}\", \"environment\": \"production\"}"
```

Store your `fd_sec_` key in GitHub Secrets. Never put it in the workflow file itself.

### Vercel post-deploy hook

In your Vercel project settings, add a Deploy Hook. Create a lightweight serverless function that calls the Flusterduck API and point the hook at it. The function receives a POST from Vercel when the deploy completes, then records the deploy:

```ts
// api/deploy-hook.ts
export async function POST() {
  await fetch('https://api.flusterduck.com/v1/deploys', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.FLUSTERDUCK_SECRET_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      commit_hash: process.env.VERCEL_GIT_COMMIT_SHA,
      environment: 'production',
    }),
  })
  return new Response('ok')
}
```

## What happens after a deploy is recorded

1. `confusion_before` is captured immediately from the current site score.
2. After 5 minutes of post-deploy traffic, `confusion_after` and `delta` are calculated.
3. All open issues are queued for verification in the next scoring cycle.
4. Issues where the signal cluster declined significantly are marked `verified`.
5. Issues that were previously resolved but show renewed signal activity are marked `regressed`.
6. If `delta` exceeds your spike alert threshold, an alert fires.

## Reading deploy data

```bash
curl "https://api.flusterduck.com/v1/deploys" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```

```json
{
  "deploys": [
    {
      "id": "dep_xxxxxxxxxxxx",
      "commit_hash": "a1b2c3d4e5f6",
      "environment": "production",
      "provider": "github",
      "confusion_before": 41,
      "confusion_after": 67,
      "delta": 26,
      "deployed_at": "2026-06-10T09:00:00Z"
    },
    {
      "id": "dep_xxxxxxxxxxxx",
      "commit_hash": "f6e5d4c3b2a1",
      "environment": "production",
      "provider": "github",
      "confusion_before": 45,
      "confusion_after": 41,
      "delta": -4,
      "deployed_at": "2026-06-08T14:00:00Z"
    }
  ]
}
```

A delta of +26 on a site averaging 41 would trigger most spike rules. That's a deploy you want to look at immediately. A delta of -4 means the fix you shipped last time worked.

## Issue verification details

Each issue includes a `verifications` array that shows the verification history:

```json
{
  "id": "iss_xxxxxxxxxxxx",
  "title": "Dead clicks on complete purchase button",
  "status": "verified",
  "verifications": [
    {
      "deploy_id": "dep_xxxxxxxxxxxx",
      "deploy_version": "v2.14.4",
      "result": "verified",
      "signal_count_before": 47,
      "signal_count_after": 3,
      "verified_at": "2026-06-11T10:15:00Z"
    }
  ]
}
```

`signal_count_after` of 3 vs 47 before is a verified fix. The few remaining signals are likely residual noise.

If a fix looks verified but `signal_count_after` is higher than expected, check whether a related issue was created separately. The engine clusters by element and signal type, so a fix that resolves rage clicks but introduces dead clicks on the same button would close the rage click issue and open a new dead click issue.

## Post-deploy checks via MCP

The `post_deploy_check` prompt runs the full correlation workflow automatically: finds your latest deploy, compares scores, identifies affected pages, and returns PASS / WARN / FAIL:

```
Run a post-deploy check.
Did the deploy this morning cause any regressions?
Show me confusion_before and confusion_after for my last 5 deploys.
```

See [MCP](./mcp) for setup.

## Staging environments

Pass `"environment": "staging"` when recording deploys to your staging environment. Staging deploy data is tracked separately from production. Your production scores won't be affected by staging friction.

```bash
curl -X POST https://api.flusterduck.com/v1/deploys \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "commit_hash": "b2c3d4e5f6a1",
    "environment": "staging"
  }'
```

Each deploy in the response carries its own `environment` field, so you can tell staging and production apart client-side. There's no server-side `environment` filter on `GET /v1/deploys`; use `limit` and `offset` (default 100, max 200 per page) to page through the full list and filter locally.
