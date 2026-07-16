# Security

Flusterduck collects behavioral signals, not private content. This page covers the security model: what leaves the browser, how authentication and authorization work, how keys are protected, where rate limits sit, and what the database enforces. If you're running a security review, this is your reference, and if you have a question it doesn't answer, ask us at support@flusterduck.com.

## What leaves the browser, and what never does

The SDK captures behavioral patterns: clicks, scroll depth, cursor movement, timing between interactions, and the **visible label** of an element a user interacted with (button text, a heading, an aria-label) so traces read in plain language: "Rage click on *Apply coupon*" instead of a bare CSS selector. Labels are PII-redacted **in the browser before anything is sent**: email addresses, phone numbers, card-length digit runs, and web addresses are replaced with `[redacted]` in place. A second, server-side redaction pass runs at ingest as a backstop.

What is never collected, by architecture rather than configuration:

- No session replay or screen recording
- No DOM recording or page-content capture from visitors' sessions
- No form values, no user-entered text, no keystrokes
- No passwords, tokens, or credentials
- No raw IP addresses: hashed at the edge before storage, originals never written

## Authentication

Four distinct authentication paths exist, each scoped to the surface it protects.

### Publishable keys (`fd_pub_`)

The browser SDK authenticates with a publishable key included in each event batch. The ingest endpoint validates the key against a salted HMAC-SHA256 hash (raw keys are never stored) and rejects batches for inactive sites, lapsed billing, or expired trials. Publishable keys can only write events. They can never read scores, issues, or any other data.

### User JWTs

The dashboard authenticates via Supabase Auth JWTs, validated server-side against the auth server on every request, so a stolen session cookie with an expired token is caught rather than trusted. After validation, every route independently checks org membership and role; admin-only operations require the `owner` or `admin` role.

### Secret keys (`fd_sec_`)

Server-side integrations use secret keys with the read and write APIs. Each key carries explicit scopes (`query:read`, `manage:write`, `webhook:write`), a per-key rate limit, and an optional IP allowlist with CIDR support. Revoked, expired, mis-scoped, or wrong-IP requests fail closed.

### MCP keys (`fd_mcp_`)

MCP keys work like secret keys but carry only the `mcp:read` scope. They give AI assistants read access to scores and issues through the hosted MCP server, never write access.

## Authorization

Authentication proves identity; authorization proves access. Every read route verifies that the caller's organization owns the requested site before returning a byte. Every write route checks org membership with the appropriate role: viewers can't modify anything; members can update issues but can't create sites or keys. Users can't change their own role or remove themselves from an org. Org creation and admin counts are capped. Management actions are recorded in an audit log with hashed IP and user-agent for forensics.

## Rate limits

Every endpoint enforces rate limits through an atomic, database-backed counter. When a limit is hit, the API returns HTTP 429 with a `reset_at` timestamp.

| Surface | Limit | Window |
|---------|-------|--------|
| Event ingestion | 10,000 requests | 60 seconds, per environment |
| Read API | 100 requests (or per-key custom RPM) | 60 seconds, per org + caller |
| Write API | 30 requests | 60 seconds, per actor |
| Waitlist | 5 requests | 15 minutes, per IP |
| Webhook delivery | 60 requests | 60 seconds |

API keys can carry a custom per-key limit, which replaces the default on the read API.

## Cryptography

- **API keys** are never stored in plaintext. Keys are hashed with HMAC-SHA256 using a server-side salt that exists only as a deployment secret. There is no hardcoded fallback; if the secret is absent the system refuses to operate rather than degrade.
- **IP addresses** are hashed with a separate salted HMAC and truncated before storage. The original IP is never written to any table.
- **All key and signature comparisons are constant-time.** No comparison short-circuits on the first mismatched byte, so timing attacks yield nothing.
- **Outbound webhooks** are signed with HMAC-SHA256 over `{timestamp}.{payload}`, giving you both integrity and replay protection: verification rejects signatures older than 300 seconds. Your endpoint secret is shown once at creation and stored encrypted.
- **Slack requests** are verified with Slack's `v0` signing scheme and a 5-minute timestamp tolerance.
- **Integration credentials and webhook secrets** are encrypted at rest with AES-256-GCM, each value under its own random IV, with key material held only as a deployment secret.

## Database isolation

Browser clients cannot query the database. This is enforced in layers, so no single mistake can expose data:

1. **No grants.** Browser-facing database roles hold no privileges on any product table, sequence, or function. The only thing a browser can do against the database layer is authenticate.
2. **Deny-all row-level security.** RLS is enabled on every table, and product tables carry explicit deny-all policies for client roles, so even if a grant were ever mistakenly issued, the policy layer still returns nothing.
3. **Future-proofing.** Default privileges are configured so tables and functions added later inherit the same lockdown automatically, rather than depending on someone remembering.
4. **All access flows through the API.** Reads and writes happen exclusively in server-side edge functions that authenticate the caller, verify org ownership, and query with service credentials. Where the dashboard needs key metadata, it reads through views that structurally exclude secret material (key hashes are not selectable, even by authenticated org members).
5. **Authorization helpers are hardened.** The functions that back org-scoping run with pinned search paths to prevent search-path injection.

## Input validation

- Request bodies are capped at 64KB on every endpoint, enforced on the compressed size *and* re-checked after decompression, so compressed payloads can't smuggle larger bodies in.
- User-controlled strings are sanitized (HTML/script metacharacters stripped) and length-capped before storage.
- Every UUID parameter is validated against a strict format before any query runs.
- Enumerated fields (trigger types, notification channels, member roles, issue statuses, key scopes) are validated against fixed allowlists; unknown values are rejected, not stored.
- Numeric parameters are clamped to safe ranges with sane defaults.

## Content Security Policy

The web app generates a per-request CSP with a cryptographically random nonce (16 bytes, never reused). Inline scripts without the nonce are blocked; network calls are restricted to same-origin and our auth endpoint with no wildcards; framing is denied entirely (`frame-ancestors 'none'`), and plugin/worker execution is blocked. `upgrade-insecure-requests` is set in production.

## Audit logging

Management actions are recorded with actor identity (user or API key), action and target, hashed IP, hashed user-agent, and operation metadata. Nothing in the audit trail contains raw network identifiers.

## Browser SDK hygiene

Use attributes or stable selectors for signal context. Don't send private user data in metadata.

```ts
signal('dead_click', {
  page: '/checkout',
  selector: '[data-action="continue"]',
})
```

Don't do this:

```ts
track('checkout_note', {
  email: user.email,
  message: form.message,
})
```

Payloads are scrubbed at ingest as a backstop: keys that look like credentials or personal fields are dropped, and element labels are PII-redacted. But the safest data is data you never send.

## Network-level controls

- API keys support IP allowlists with CIDR notation; requests from outside the allowlist get a 403.
- Customer webhook URLs are validated against internal, private, and metadata address ranges before any delivery, redirects are refused outright, and the same checks re-run at delivery time, so a webhook endpoint can never be used to probe internal networks.
- The site scanner's fetches run through the same public-address validation. AI-diagnosis page fetches are made by our rendering provider against the site's own registered public URL, never an address derived from visitor input.
- Unauthenticated browser navigations redirect to login; unauthenticated API fetches receive a 401. The middleware distinguishes the two by fetch metadata, so APIs never leak redirect chains.

## Reporting a vulnerability

We publish a `security.txt` at [flusterduck.com/.well-known/security.txt](https://flusterduck.com/.well-known/security.txt). If you find something, tell us at support@flusterduck.com. We read every report and fix what's real, regardless of severity.
