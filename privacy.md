# Privacy and Data Collection

Flusterduck measures behavioral patterns without session replay. It does not collect form values or user-typed text, and it never stores raw IP addresses. This page accounts for what the product does collect, including the narrow text context that makes an issue readable.

## What the SDK collects

**Interaction events.** Clicks, taps, scrolls, focus events, and keyboard interactions. Specifically: which element was involved, coordinates relative to it, a timestamp, and, when useful, a short accessible label such as "Apply coupon." Labels are capped and PII-redacted before they leave the browser. Input values and user-typed text are not read.

**Timing data.** How long users spend on pages, how long they pause on form fields, how long loads take. Millisecond durations.

**Navigation data.** Page paths and the sequence of pages within a session. Query string values are not captured by default.

**Structured events and derived signals.** The SDK turns interactions into small typed records and detects friction patterns locally. Some features keep a short sequence of structured breadcrumbs so an issue can show what happened. Flusterduck does not produce replay video, DOM snapshots, or a copy of the page.

**Optional episode evidence.** When a site explicitly enables this private-pilot feature, the SDK may send one small state packet around a detected friction incident. The packet can contain enum or boolean element states, rounded geometry changes, count-only DOM mutations, passive request status and duration, the selector that received a click, and a form submit receipt with invalid selectors and field lengths. It never contains DOM content, form values, typed text, request bodies, headers, or query strings. It emits no episode packet for calm sessions and is off by default.

**Session identifiers.** A random string used to group signals. By default it is stored in a Flusterduck session cookie for continuity; with `cookieless: true`, it is memory-only and resets on page load. Not tied to any user identity unless you call `identify()`.

## What the SDK never collects

No input values or user-typed text, including passwords and payment details. This is enforced at the SDK level: the SDK does not attach the kind of listeners that could capture keystrokes or form values.

No arbitrary page copy or user-entered text. The SDK may read a short label from the specific element involved in an interaction so evidence is understandable. It never reads an input's value or text inside a content-editable field. Labels are length-capped and PII-redacted in the browser, then scrubbed again at ingestion.

The label redactor targets email addresses, web addresses, IBANs, and phone- or card-length digit runs. It cannot reliably infer every personal name or free-form postal address in static interface copy. Do not render personal data into labels on tracked controls; use `data-fd-ignore` on any area that can contain it.

No session replay. No video recording, no visitor-session screenshot capture, no DOM serialization. Optional episode evidence is a bounded structured receipt, not a page reconstruction.

No raw IP addresses. IPs are hashed before storage and are never written to disk in recoverable form.

No names or emails are added automatically. If you call `identify()`, use an opaque internal ID rather than a name or email address.

## Public page context used by AI

When AI detection or diagnosis is available, Flusterduck may fetch bounded rendered text and a logged-out screenshot from a configured site's publicly reachable page. This context is fetched separately from visitor telemetry and is not reconstructed from a visitor session. A public screenshot can contain anything visibly rendered to a logged-out visitor, including public default or prefilled form values. It never contains values entered in a tracked visitor session.

A detecting run can request at most four fresh public-page screenshots. A separate read-only visual reviewer receives each image as base64 data, returns a constrained set of visible-state facts, and has no issue-writing or memory tools. Raw pixels do not enter the issue-writing analyst's saved transcript. Customer-uploaded images for authenticated pages are never sent to either detecting model. For a current-page check, a separate logged-out Chromium worker loads the exact public page at one requested mobile or desktop size and captures the top or one requested vertical position. It can then scroll, hover visible controls, and walk forward and backward through a bounded part of the Tab order, returning numeric outcomes from that fixed visit. It does not run Flusterduck detectors, click, type, submit, sign in, or receive visitor-session data. The image is withheld if a security boundary changes the page before capture or the browser misses the requested position.

A screenshot can confirm only what was visibly rendered at capture time. The browser check can confirm only what happened during its fixed visit. Neither proves user intent, a click path it did not run, a browser-extension or locale state, or a root cause.

Guide sends only the hovered element's role, state, selector, short accessible label, and nearest labeled parent. It does not send the full DOM or a screenshot of the visitor's page.

## Source code used by AI detection

Source access is off by default and is granted per site. A site set to Link only stores which repository it ships from and nothing is read. Once source reads are approved for a site, Flusterduck reads only the GitHub repository mapped to it. The read token is limited to that one repository and requests read access to repository contents.

When a production deploy includes a commit SHA, code checks read that deployed commit. Short SHAs are resolved to the full GitHub SHA first. If the deploy has no commit SHA, a read may use the current default-branch tip, but the result is marked unpinned and cannot be the sole proof for a code-found issue.

The detection model may receive a short source excerpt while it investigates a receipt. Flusterduck stores the repository, ref, full SHA, file, line number, and a server-computed content hash. It does not store the source excerpt or a diff patch in the investigation transcript. Search results also state when a bounded literal search is inconclusive, since generated class names and delegated event handlers can evade that search.

The GitHub App may separately request write permission for Autofix. AI detection never receives that write-capable token. Autofix uses a separate token and can only open a draft pull request through its own approval and safety checks.

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

The browser backstops this: any key that reads as a contact field (`email`, `phone`, `address`, `name`, and similar) is dropped, and any value shaped like an email address or a phone-length digit run is dropped too. Pass a bare string instead of an object, like `identify('usr_8f3a2c91')`, and the same check applies: anything containing `@` or a long digit run is refused.

That catches the obvious cases, not the subtle ones. A value that doesn't look like PII to a regex, an internal username, a full postal code, can still identify someone, so choosing opaque values stays your responsibility.

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

Flusterduck is designed to minimize personal data, not to make a blanket claim that pseudonymous analytics data falls outside GDPR. The regulation includes online identifiers in its definition of personal data and treats pseudonymisation as processing: [GDPR Article 4](https://eur-lex.europa.eu/eli/reg/2016/679/oj).

Treat session identifiers, IP-derived hashes, and any internal user ID you pass to `identify()` according to your own legal obligations. Include Flusterduck in your data-processing records where required and honor deletion requests. This page describes the product's technical boundaries; it is not legal advice.

## What you put into track() and signal()

The `track()` and `signal()` methods accept arbitrary metadata. Don't put PII in them.

Safe for `track()`: `plan_id`, `amount_cents`, `billing`, `currency`, `quantity`, `product_id`.

Safe for `signal()`: CSS selectors, element labels, page section names, plan IDs, product IDs.

Never pass: names, emails, phone numbers, addresses, order notes, or any text the user typed.

## Subprocessors and service providers

- **Supabase** runs the database and edge functions that receive structured behavioral data, pseudonymous session IDs, IP-derived hashes, and redacted element labels.
- **Anthropic** processes bounded evidence and context for AI detection, diagnosis, Guide, and Autofix when those features run. For AI detection, a separate read-only model call may receive up to four fresh public-page screenshot images in one investigation. When a site opts into code evidence, the detection model may also receive short source excerpts from that site's mapped repository.
- **context.dev** retrieves rendered text and screenshots from publicly reachable pages used to ground AI analysis and page heatmaps. It receives public page URLs, not a visitor replay.
- **Cloudflare** serves CDN and MCP infrastructure and runs the isolated logged-out browser used for public-page checks.
- **Resend** sends transactional email to Flusterduck account members.
- **Stripe** processes Flusterduck customer billing.

The data sent depends on the feature in use. Form values and user-typed text are excluded from the behavioral pipeline by design.

## Data retention

Raw events: 90 days. Aggregated scores and issue history: life of your account. On cancellation, all data deleted within 30 days.

## What to tell your users

This is accurate for most privacy policies:

> We use Flusterduck to detect usability issues on our site. Flusterduck collects pseudonymous behavioral signals such as clicks, scroll patterns, navigation, short PII-redacted control labels, and, when enabled, bounded state evidence around detected friction. It does not record session replay, form values, or user-typed text.

If you're in a jurisdiction with specific disclosure requirements, check with your legal team.
