---
name: authologic-run-identity-verification
description: >-
  Run an end-to-end identity (KYC) verification with Authologic — create a conversation declaring the
  attributes you need, redirect the user, receive the signed callback, and read the result. Use when you
  need to verify a natural person's identity via eID, bank ID, document scan or liveness.
api: Authologic Customer API
version: '1.1'
generated: '2026-09-14'
method: generated
source: >-
  Grounded in openapi/authologic-customer-api-openapi.yml (operationIds verified against the spec) and
  https://developer.authologic.com/docs/products/5-minutes-tutorial,
  /docs/integration/overview, /docs/integration/callbacks, /docs/technical/errors.
operations:
  - createConversation
  - getConversation
  - deleteConversation
---

# Run an identity verification

## Before you start

- **Base URL.** `https://sandbox.authologic.com` for testing. Production is `https://api.authologic.com`
  and is **IP-allowlisted** — an unlisted caller gets an nginx `403 IP not allowed` HTML page, not a JSON
  error. Ask Authologic to allowlist your egress addresses before you go live.
- **Auth.** HTTP Basic: account name as username, environment-specific API key as password. The API key
  is generated in OmniPanel and is *not* the OmniPanel account password. OAuth 2.0 client credentials is
  also available at `/api/oauth2/token` if you prefer bearer tokens.
- **Versioning.** Send **both** headers on every call:
  `Content-Type: application/vnd.authologic.v1.1+json` and `Accept: application/vnd.authologic.v1.1+json`.
- **You need a working callback receiver.** Results arrive asynchronously. Polling works but the provider
  explicitly discourages it — verification can take minutes or days.

## Step 1 — create the conversation (`createConversation`)

`POST /api/conversations`

```bash
curl -X POST -u my_login "https://sandbox.authologic.com/api/conversations" \
  -H "Accept: application/vnd.authologic.v1.1+json" \
  -H "Content-Type: application/vnd.authologic.v1.1+json" \
  -d '{
    "userKey": "7dfb9ded-c38f-49ae-95e2-307283a0b1f6",
    "returnUrl": "https://your-app.example/verify/{conversationId}/done",
    "callbackUrl": "https://your-app.example/hooks/authologic?conversation={conversationId}&event={event}",
    "strategy": "public:sandbox",
    "query": {
      "identity": {
        "requireOneOf": [["PERSON_NAME_FIRSTNAME", "PERSON_NAME_LASTNAME"]]
      }
    }
  }'
```

- `userKey` is **your** identifier for the user. Authologic echoes it back so you can correlate.
- `query.identity.requireOneOf` declares the attributes you need from the `PERSON_*` / `COMPANY_*`
  vocabulary. See `/docs/integration/user-info-fields` for the full list and
  `/docs/technical/mandatory-and-optional-queries` for the requireOneOf-vs-optional syntax.
- `strategy` is optional. `public:sandbox` simulates the whole flow without a real verification — use it
  for every integration test.
- `{conversationId}`, `{target}` and `{event}` in either URL are substituted at delivery time.

**This is a billable, non-idempotent write.** There is no `Idempotency-Key` on this API. A retried POST
creates a **second** conversation with a new id. If a call times out, do not blind-retry — you have no way
to look up what you created (there is no list or search operation), so record the response id first and
treat a timeout as a manual reconciliation case.

The 200 returns `id`, `userKey`, `url` and `status: CREATED`. Redirect the user's browser to `url`.

## Step 2 — receive the callback

Authologic POSTs to your `callbackUrl`:

```json
{
  "id": "02eb1705-fe8f-4d3d-b768-f48b06d26a7e",
  "created": "2020-09-17T11:18:21.999Z",
  "target": "CONVERSATION",
  "event": "FINISHED",
  "payload": { "conversation": { "id": "e0c0b3cc-...", "status": "FINISHED", "result": { "identity": { ... } } } }
}
```

Four rules, all of which the provider states explicitly:

1. **Verify the signature first.** Compute `HMAC_SHA_256("<X-Signature-Timestamp>:<raw body>")` with the
   signing key Authologic issued (a *different* key from your API key) and compare to `X-Signature`.
   Reject if `X-Signature-Timestamp` is more than 5 minutes from now. Verify against the **raw** body — a
   framework that re-serialises the JSON will break the digest.
2. **Use `payload.conversation.id`, not the top-level `id`.** The top-level `id` is the *callback*
   identifier. Calling `getConversation` with it returns 404. This is the most common integration bug here.
3. **Answer 200, 201, 202 or 204** — anything else is a failed delivery and triggers retries (at least 20
   attempts, the last no sooner than 4 days). Dedupe on the top-level `id`, which is stable across
   redeliveries.
4. **Ignore unknown fields and unknown `target`/`event` values.** Accept them, respond success, move on.
   Authologic's integration verification deliberately sends a payload with random extra fields.

You may receive more than one callback for a conversation (for example, one when user data lands and a
second when bank transactions are ready). The final one has `status: FINISHED`.

## Step 3 — read the result (`getConversation`)

`GET /api/conversations/{conversationId}` — use this to poll if you must, or to re-read after a callback.

**`status: FINISHED` does not mean the verification succeeded.** It means the process ended. Success lives
in `result.identity.status`:

- `FINISHED` — verified; read attributes from `result.identity.user`.
- `FAILED` — read `result.identity.errors[]` for the reason codes.

Also check `result.identity.checks[]` for non-fatal validations such as `CONSISTENCY_CHECK`.

### Handling failure reasons

See `errors/authologic-error-codes.yml` for all 52 codes. The decision you actually need to make:

- **Retry with a better capture:** `SCAN_DOCUMENT_TOO_BLURRY`, `SCAN_DOCUMENT_NOT_DETECTED`,
  `SCAN_DOCUMENT_NOT_FULLY_VISIBLE`, `SCAN_DOCUMENT_LOW_QUALITY`, `SCAN_FACE_TOO_BLURRY`,
  `SCAN_FACE_NOT_DETECTED`, `SCAN_FACE_TOO_MANY_PEOPLE`.
- **Do not retry the same document:** `SCAN_DOCUMENT_EXPIRED`, `SCAN_DOCUMENT_PROBABLY_FAKE`,
  `SCAN_DOCUMENT_NOT_SUPPORTED`, `SCAN_FACE_MISMATCH`, `SCAN_DOCUMENT_DATA_MISMATCH`.
- **Retry later or fall back to another method:** `PROVIDER_UNAVAILABLE`, `PROVIDER_SERVER_ERROR`,
  `SOURCE_UNAVAILABLE`.
- **Re-invite the user:** `ABANDONED`, `NO_DATA_SHARED`.
- **Never show the user:** `FRAUD_SUSPICION` and `PROVIDER_BLACKLIST` are internal-only by provider
  instruction. Log them; do not put them in user-facing messaging.

The list is explicitly **not closed** — handle unknown codes as a generic failure.

## Step 4 — ending a conversation early (`deleteConversation`)

`DELETE /api/conversations/{conversationId}?mode=EXPIRE|DELETE_DATA`

- `EXPIRE` (default) ends an unfinished conversation. Status becomes `EXPIRED`; a user who opens the
  process URL is bounced to your `returnUrl`.
- `DELETE_DATA` permanently deletes the data of a **finished** conversation. Irreversible, and pending
  callbacks for that conversation may never arrive. Returns 409 if the state does not allow it.

After either, reads return **410 Gone**. Authologic publishes no retention window — it is a per-contract
setting — so do not assume data will still be there tomorrow.

## HTTP errors

`400` validation (read `violations[]`) · `402` plan exhausted or account restricted for non-payment ·
`403` not entitled (or, on production, not IP-allowlisted) · `404` unknown conversation ·
`409` state conflict on delete · `410` data gone · `500` retry with backoff.

No `429` and no rate-limit headers exist on this API. `402` is the only exhaustion signal.
