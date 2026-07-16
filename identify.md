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

`identify()` accepts one object: `Record<string, string>`. Values are stringified, truncated, and attached to the current session. Keep IDs opaque. A database primary key or UUID is fine. An email address, display name, or phone number is not.

## The string shorthand

When all you want is to tag the session with your own user reference, pass a single string:

```ts
identify('usr_8f3a2c91')
```

This sets the session's `user_ref` and merges with any properties set earlier, it never clobbers them. The ref must be opaque: values that look like contact information (anything containing an `@`, or a long digit run like a phone or card number) are refused in the browser and never sent.

The `user_ref` is what powers the resolution loop: when Flusterduck verifies that a fix worked after a deploy, the `issue.verified` webhook delivers the refs of exactly the users who hit that issue, so your own systems can tell them it's fixed. See [Resolution loop](/resolution-loop).

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

`segment` properties are attached from the first signal in the session. `identify()` properties are attached after the call.

Use `segment` for: deploy tags, region, A/B test cohorts, feature flag variants at the session level.

Use `identify()` for: plan tier, organization, user role, account age, anything tied to a specific authenticated user.

If you pass the same key in both `segment` and `identify()`, the `identify()` value wins after the call.

## Using segments in investigations

With properties in place, you can filter friction data by segment values. Via MCP:

```
Are rage clicks on the checkout button worse for Grow plan users than Scale?
Which sessions hitting form abandonment on onboarding are enterprise accounts?
Compare confusion scores between experiment_group: pricing-v2-a and pricing-v2-b.
Show me sessions where role: admin hit the dead click on the settings page.
```

## Multiple identify() calls

Calling `identify()` multiple times in the same session merges properties.

```ts
// On login
identify({ user_id: 'usr_8f3a2c91', plan: 'scale' })

// After loading org data
identify({ org_id: 'org_4b2e9f1c', org_size: '11-50' })
```

The session ends up with all four properties. Use this pattern when you load user data in stages.
