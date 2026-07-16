# Privacy and Data Collection

Flusterduck measures behavioral patterns, not personal data. No session replay, no form values, no text content, no raw IPs. This page is the full accounting of what actually gets collected.

## What the SDK collects

**Interaction events.** Clicks, taps, scrolls, focus events, and keyboard interactions. Specifically: which element was interacted with (CSS selector), coordinates relative to the element, and timestamp. Not what was typed.

**Timing data.** How long users spend on pages, how long they pause on form fields, how long loads take. Millisecond durations.

**Navigation data.** Page paths and the sequence of pages within a session. Query string values are not captured by default.

**Derived signals.** The SDK processes raw interaction events locally to detect friction patterns before sending anything. What leaves the browser is the signal type and any metadata you provide, not the raw event stream.

**Session identifiers.** A random string used to group signals. By default it is stored in a Flusterduck session cookie for continuity; with `cookieless: true`, it is memory-only and resets on page load. Not tied to any user identity unless you call `identify()`.

## What the SDK never collects

No passwords, credit card numbers, names, addresses, or any input a user types. This is enforced at the SDK level: the SDK doesn't attach the kind of listeners that could capture keystrokes or form values.

No text content. The SDK doesn't read what's on your pages. No labels, headings, paragraphs, or DOM text of any kind.

No session replay. No video recording, no screenshot capture, no DOM serialization.

No raw IP addresses. IPs are hashed before storage and are never written to disk in recoverable form.

No emails or names via `identify()`. If you attach an identifier, use an opaque internal ID.

## IP addresses

When events reach the server, the IP is hashed using a one-way function. The original is never stored. The hash is used only for session deduplication and rate limiting.

## Session IDs

Session IDs are random strings the SDK generates. They're not tied to device fingerprints. In default mode the SDK stores the ID in a first-party Flusterduck cookie; in cookieless mode it stays in memory only.

## The identify() method

`identify()` tags the current session with safe segment properties. If you include a user identifier, use an opaque value: a database primary key or UUID, not an email or full name.

```ts
// Do this
identify({ user_id: 'usr_8f3a2c91', plan: 'scale' })

// Not this
identify({ user_id: 'alice@example.com' })
identify({ name: 'Alice Johnson' })
```

Flusterduck doesn't validate what you pass. That's your responsibility.

## Consent and opt-out

### Cookie consent flow

For zero collection before consent, wait to initialize until the user accepts, or use a framework wrapper with `enabled: false`:

```ts
if (userAcceptedAnalytics) {
  init({ key: process.env.NEXT_PUBLIC_FLUSTERDUCK_KEY! })
}
```

If the SDK is already initialized, call `setConsent(false)` to flush the buffer and stop collection. Call `setConsent(true)` only when you intentionally want to reinitialize with the previous config.

```ts
setConsent(true)   // user accepted
setConsent(false)  // user rejected or revoked
```

### User opt-out

```ts
import { optOut } from 'flusterduck'
optOut()
```

Stops collection immediately, clears the session buffer. The SDK stays loaded but inactive for the rest of the session.

## GDPR

Flusterduck doesn't process personal data as defined under GDPR because it doesn't collect any. IP addresses are hashed at the edge before storage. No name, email, or identifier is collected unless you pass one to `identify()`.

If you use `identify()` with internal user IDs that trace back to real people, include Flusterduck behavioral data in your data processing records and honor deletion requests by contacting support with the IDs to purge.

## What you put into track() and signal()

The `track()` and `signal()` methods accept arbitrary metadata. Don't put PII in them.

Safe for `track()`: `plan_id`, `amount_cents`, `billing`, `currency`, `quantity`, `product_id`.

Safe for `signal()`: CSS selectors, element labels, page section names, plan IDs, product IDs.

Never pass: names, emails, phone numbers, addresses, order notes, or any text the user typed.

## Subprocessors

Supabase (database and edge functions), Cloudflare (MCP worker and CDN), Resend (transactional email), Stripe (billing). None receive your users' personal data from Flusterduck's data pipeline.

## Data retention

Raw events: 90 days. Aggregated scores and issue history: life of your account. On cancellation, all data deleted within 30 days.

## What to tell your users

This is accurate for most privacy policies:

> We use Flusterduck to detect usability issues on our site. Flusterduck collects anonymized behavioral signals (clicks, scroll patterns, navigation) but never records session replay, form values, or personal information.

If you're in a jurisdiction with specific disclosure requirements, check with your legal team.
