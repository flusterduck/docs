# Deploy Correlation

Flusterduck needs to know when you shipped. Without it, the engine can't tell a friction spike caused by a broken release from one caused by a traffic surge or a bad email campaign, it can't verify that your fixes worked, and it reads your code at whatever is on your default branch today instead of at the commit that was live when the friction happened.

## If your repository is connected, this is already done

Connect a repository under Settings, Sites and Flusterduck watches it. Nothing to install and nothing to run in CI.

It reads two things, in this order:

1. **GitHub Deployments.** Vercel, Netlify, Render, Fly and most deploy actions create one of these on every ship. A deployment whose latest status is `success`, in a production environment, is recorded as a deploy.
2. **Your default branch moving.** Used when there are no GitHub Deployments to read. For the many teams whose `main` branch is what runs in production, this is the same answer.

Which one produced a given record is stored on it, so you can always tell an observed deploy from an inferred one.

A deployment that failed is not recorded. Neither is a preview environment. Nothing is recorded twice: if you also run the CLI or post to the API, the commit is matched and you get one deploy, not two.

If your default branch is **not** what you ship, turn watching off on the source card in Settings, Sites and record deploys yourself with either method below. The repository stays connected for everything else.

For the fastest path, subscribe the Flusterduck GitHub App to the `deployment_status` and `push` events and grant it `deployments: read`. Deploys then land within seconds instead of at the next hourly check.

## Recording a deploy yourself

```bash
curl -X POST https://api.flusterduck.com/v1/deploys \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "version": "v2.14.3",
    "environment": "production"
  }'
```

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
      -d "{\"version\": \"${{ github.sha }}\", \"environment\": \"production\"}"
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
      version: process.env.VERCEL_GIT_COMMIT_SHA,
      environment: 'production',
    }),
  })
  return new Response('ok')
}
```

### SDK-level version tagging

If CI integration isn't available, set `segment.app_version` in your SDK config. The engine uses it to associate sessions with deploy records and improve correlation accuracy:

```ts
init({
  key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!,
  segment: {
    app_version: process.env.NEXT_PUBLIC_APP_VERSION ?? 'unknown',
  },
})
```

Set `NEXT_PUBLIC_APP_VERSION` to your git SHA or version string at build time. This doesn't replace the API call for recording deploys, but it improves session-to-deploy correlation when version strings are consistent.

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
      "version": "v2.14.3",
      "environment": "production",
      "confusion_before": 41,
      "confusion_after": 67,
      "delta": 26,
      "recorded_at": "2026-06-10T09:00:00Z"
    },
    {
      "id": "dep_xxxxxxxxxxxx",
      "version": "v2.14.2",
      "environment": "production",
      "confusion_before": 45,
      "confusion_after": 41,
      "delta": -4,
      "recorded_at": "2026-06-08T14:00:00Z"
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
    "version": "v2.14.4-rc1",
    "environment": "staging"
  }'
```

Filter deploys by environment when reading:

```bash
curl "https://api.flusterduck.com/v1/deploys?environment=production" \
  -H "Authorization: Bearer fd_sec_xxxxxxxxxxxx"
```
