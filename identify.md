# Identifying Sessions

Tag sessions with safe user, account, or cohort properties so friction data is segmentable. Once you're calling `identify()`, you can ask whether checkout friction is worse for Grow users than Scale users, whether a redesign cohort is hitting the same issues as the control group, or whether enterprise accounts behave differently during onboarding.

Without it, you're looking at aggregate confusion scores with no way to slice them.

## identify()

Call it after login, when you have stable properties:

```ts
import { identify } from 'flusterduck'

identify({
  user_id: 'usr_8f3a2c91',
  plan: 'scale',
  org_id: 'org_4b2e9f1c',
  account_age_days: '42',
})
```

`identify()` accepts one object. Values can be strings, numbers, or booleans; each is coerced to a string and truncated to 256 characters. Keys are capped at 64 characters, and only the first 20 keys in a single call are kept. Keep IDs opaque. A database primary key or UUID is fine. An email address, display name, or phone number is not.

Two filters run in the browser before any of it ships. A trait is dropped, silently, if its key reads as PII (anything containing `email`, `phone`, `name`, `address`, `ssn`, `dob`, `card`, `password`, `token`, `auth`, or similar segments) or its value looks like contact information (an `@`, or a long digit run). If a trait you passed isn't showing up, check the key name first: `identify({ email: 'wrong' })` doesn't error, it just never leaves the browser.

## The string shorthand

When all you want is to tag the session with your own user reference, pass a single string:

```ts
identify('usr_8f3a2c91')
```

This sets the session's `user_ref` and merges with any properties set earlier, it never clobbers them. The ref must be opaque: values that look like contact information (anything containing an `@`, or a long digit run like a phone or card number) are refused in the browser and never sent.

The `user_ref` is what powers the resolution loop: when Flusterduck verifies that a fix worked after a deploy, the `issue.verified` webhook delivers the refs of exactly the users who hit that issue, so your own systems can tell them it's fixed. See [Resolution loop](/resolution-loop). Setting the ref through the object form of `identify()` instead of the string shorthand works too: use the key `user_ref` (`user` and `uid` are also recognized).

## What properties to pass

Pass things you'd want to filter by when investigating a friction pattern:

```ts
identify({
  user_id: 'usr_8f3a2c91',
  plan: 'scale',
  billing: 'annual',
  org_size: '11-50',
  role: 'admin',
  account_age_days: '42',
  experiment_variant: 'checkout-v2',
  feature_flag_new_nav: 'true',
})
```

Never pass: email, name, username, IP address, display handle, or any field that links directly to a real person.

## When to call it

Call `identify()` as early as possible after login and after the SDK has initialized. Properties apply to the current session metadata used for future flushes.

In practice: put it in a component or hook that runs immediately after your auth state resolves.

### React

```tsx
import { useEffect } from 'react'
import { identify } from 'flusterduck'

export function IdentityBridge({ userId, plan, orgId }: {
  userId?: string
  plan?: string
  orgId?: string
}) {
  useEffect(() => {
    if (!userId) return
    identify({
      user_id: userId,
      plan: plan ?? 'trial',
      org_id: orgId ?? '',
    })
  }, [userId, plan, orgId])

  return null
}
```

Render this inside your auth provider where `userId` is reliably available.

### Next.js

```tsx
'use client'
import { useEffect } from 'react'
import { identify } from 'flusterduck'

export function IdentifyUser({ userId, plan }: { userId?: string; plan?: string }) {
  useEffect(() => {
    if (!userId) return
    identify({ user_id: userId, plan: plan ?? 'trial' })
  }, [userId, plan])

  return null
}
```

### Vue

```vue
<script setup lang="ts">
import { watch } from 'vue'
import { identify } from 'flusterduck'

const props = defineProps<{ userId?: string; plan?: string }>()

watch(
  () => props.userId,
  (userId) => {
    if (!userId) return
    identify({ user_id: userId, plan: props.plan ?? 'trial' })
  },
  { immediate: true },
)
</script>
```

## The segment config option

For properties that apply to every session regardless of who's logged in, use `segment` in your SDK config instead of `identify()`:

```ts
init({
  key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY!,
  segment: {
    app_version: process.env.NEXT_PUBLIC_APP_VERSION ?? 'unknown',
    region: 'us-east-1',
    experiment_group: 'pricing-v2-b',
  },
})
```

`segment` properties are attached starting at SDK init. `identify()` properties are attached after the call, and the object form replaces the whole trait set each time, including whatever `segment` set at init. Call `identify()` with an object and your `segment` config is gone from that point on, not just the keys that collided.

Use `segment` for: deploy tags, region, A/B test cohorts, feature flag variants at the session level.

Use `identify()` for: plan tier, organization, user role, account age, anything tied to a specific authenticated user. If you want both to survive, fold your `segment` values into the object you pass to `identify()`.

## Using segments in investigations

With properties in place, you can filter friction data by segment values. Via MCP:

```
Are rage clicks on the checkout button worse for Grow plan users than Scale?
Which sessions hitting form abandonment on onboarding are enterprise accounts?
Compare confusion scores between experiment_group: pricing-v2-a and pricing-v2-b.
Show me sessions where role: admin hit the dead click on the settings page.
```

## Multiple identify() calls

Only the string shorthand merges. `identify('usr_8f3a2c91')` preserves whatever traits are already set and only touches `user_ref`.

The object form does not merge: each call replaces the entire trait set, including anything set by an earlier `identify()` call or by `segment` in your config. Calling it twice with different objects loses the first call's traits, not just the keys that didn't repeat.

```ts
// Wrong: the second call wipes out user_id and plan
identify({ user_id: 'usr_8f3a2c91', plan: 'scale' })
identify({ org_id: 'org_4b2e9f1c', org_size: '11-50' })
// session ends up with only org_id and org_size

// Right: pass everything you want kept, every time
let traits = { user_id: 'usr_8f3a2c91', plan: 'scale' }
identify(traits)

traits = { ...traits, org_id: 'org_4b2e9f1c', org_size: '11-50' }
identify(traits)
```

Keep your own copy of the traits object in application state (a ref, a store, whatever you already use) and call `identify()` with the full set each time you load new data, not just the fields that changed.
