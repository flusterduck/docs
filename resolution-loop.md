# Resolution Loop

Close the loop with the exact users who hit an issue, without Flusterduck ever knowing who they are.

The loop has three steps:

1. **Tag sessions** with your own opaque user reference via the SDK: `identify('usr_8f3a2c91')`. The ref means something in your database and nothing in ours.
2. **Fix the issue.** When a deploy ships and Flusterduck's verification measures confusion actually dropping on that page, the issue flips to verified. Not "the PR merged": measured, on real traffic.
3. **Tell exactly the affected users.** The `issue.verified` webhook fires with the list of user refs whose sessions hit the issue. Your server maps refs back to real users and sends the message from your own email stack: "That checkout bug you ran into on Tuesday? Fixed, and we checked."

Almost nobody does this, because almost nobody knows which users hit which issue. Support teams apologize to whoever writes in; everyone else just remembers your product being broken. The resolution loop turns a silent fix into a retention moment.

## Why there's no email field

Flusterduck stores behavioral signals, never identities. That doesn't change here:

- Refs are opaque strings you choose. A primary key or UUID, never an email or name.
- The SDK refuses PII-shaped refs in the browser (anything containing `@`, or a long digit run) before they're sent.
- The API drops PII-shaped values on the way out too, as a second gate, so even a mistakenly stored ref can't leave.
- The mapping from ref to person lives only in your systems, and any sending happens from your email tools.

You get the precision of session-level identity with none of the data liability. Your privacy policy doesn't grow a new paragraph because of us.

## Setting it up

**1. Tag sessions after login:**

```ts
import { identify } from 'flusterduck'

// after your auth state resolves
identify(currentUser.id) // opaque ref, sets user_ref
```

The shorthand merges with any segment properties you already pass; see [Identifying sessions](/identify).

**2. Subscribe a webhook to `issue.verified`** under Settings > Webhooks. The payload:

```json
{
  "event": "issue.verified",
  "data": {
    "issue_id": "iss_3a7f2c9d4e1b",
    "title": "Continue button looks enabled but isn't",
    "page": "/checkout",
    "confusion_before": 61.4,
    "confusion_after": 18.2,
    "reduction_pct": 70,
    "revenue_recovered_cents": 84000,
    "affected_user_refs": ["usr_8f3a2c91", "usr_1d4e7b22"],
    "identified_sessions": 214,
    "unidentified_sessions": 96,
    "refs_truncated": false
  }
}
```

`affected_user_refs` is deduped and capped at 1,000; `refs_truncated` tells you when a very high-volume issue exceeded the scan caps. `unidentified_sessions` counts sessions that hit the issue before you started identifying (or from logged-out users): those users exist, you just can't message them, which is itself a good argument for calling `identify()` early.

**3. Send from your side.** A minimal handler:

```ts
app.post('/webhooks/flusterduck', verifySignature, async (req, res) => {
  const { event, data } = req.body
  if (event === 'issue.verified' && data.affected_user_refs.length) {
    const users = await db.users.findMany({ where: { id: { in: data.affected_user_refs } } })
    await email.sendBatch(users.map((u) => fixedItNote(u, data)))
  }
  res.sendStatus(200)
})
```

Deliveries are HMAC-signed and retried like every other Flusterduck webhook; see [Webhooks](/webhooks) for signature verification.

## Reading the list in the dashboard

Each issue's detail page shows an Affected users card once identified sessions exist: how many sessions were identified vs anonymous, with the refs one click away. Useful for a quick "who do I owe an apology to" even before the webhook plumbing exists.

## Good taste with the loop

- Send the note only when the fix is meaningful to the user. A rage-clicked broken Upgrade button deserves a message; a mild layout quirk doesn't.
- Include the measurement if you want the message to land: "confusion on that page dropped 70% after the fix" reads better than "should be fixed now."
- Don't re-identify users across logout on shared devices; call `identify()` per authenticated session.
