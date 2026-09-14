---
name: authologic-screen-aml-lists
description: >-
  Screen a person against AML lists (sanctions, PEP, adverse media) with Authologic — with or without an
  identity verification attached — read the hits, and manage ongoing monitoring. Use for KYC/AML
  onboarding and for continuous screening of an existing customer base.
api: Authologic Customer API
version: '1.1'
generated: '2026-09-14'
method: generated
source: >-
  Grounded in openapi/authologic-customer-api-openapi.yml (operationIds verified against the spec) and
  https://developer.authologic.com/docs/products/aml.md,
  /docs/products/aml/api-integration-with-identity.md,
  /docs/products/aml/api-integration-without-identity.md,
  /docs/products/aml/api-integration-aml-monitoring.md, /docs/integration/callbacks.
operations:
  - createConversation
  - getConversation
  - getAMLList
  - cancelAMLSubscription
---

# Screen against AML lists

Authologic offers AML screening in three shapes, all started by `createConversation` with an `aml`
section in `query`:

| Shape | What it does | User interaction |
|---|---|---|
| AML **with** identity | Verify who the person is, then screen the verified identity | Yes — user completes a verification |
| AML **without** identity | Screen the data *you* supply | No — server-to-server only |
| AML **monitoring** | Keep screening over time and notify on list changes | No, after setup |

Which one you need is a compliance decision, not a technical one: screening data you were *given* is
weaker evidence than screening an identity that was *verified*.

## Step 1 — start the screening (`createConversation`)

`POST /api/conversations`, with the same auth, versioning headers and callback rules as any other
conversation (see `authologic-run-identity-verification`).

- **Without identity**, no redirect happens — you supply the person's data and the process runs
  server-side. The response gives you a conversation id and status; the result arrives by callback.
- **With identity**, you get a `url` and redirect the user as normal.
- A single conversation can combine products — you do not need separate conversations for identity and
  AML. See `/docs/integration/combining-products`.

Note the deprecation: creating a conversation for AML **without** KYC no longer requires a
`query.identity` section. If your integration still sends one out of habit, drop it.

**No idempotency.** A retried create screens again and bills again.

## Step 2 — read the hits (`getAMLList`)

`GET /api/conversations/{conversationId}/aml/{list}?page=&pageSize=`

`{list}` selects which AML list's findings to read. This is one of only two paginated operations in the
API — page-number style, `page` and `pageSize` query parameters, no cursor and no `Link` header. Walk
pages until you get a short one.

Read the overall outcome from `getConversation` first (`result.aml.status`); use `getAMLList` to pull the
detail behind a hit.

Expect PEP, sanctions and adverse-media categories. Treat every hit as a *candidate* requiring human
adjudication — a name match is not a determination.

## Step 3 — ongoing monitoring and cancelling it (`cancelAMLSubscription`)

When AML monitoring is enabled, the conversation carries a subscription. As lists change, Authologic
sends a callback:

```json
{ "target": "SUBSCRIPTION", "event": "NEW_DATA", "payload": { "conversation": { ... } } }
```

`payload.conversation` carries the updated findings. Re-read with `getAMLList` for the detail.

To stop monitoring:

`DELETE /api/subscriptions/{conversationId}/aml` → **204 No Content**

Note the path shape: the subscription has **no id of its own** — it is addressed by the conversation it
belongs to. Cancellation is the reversal path for monitoring; the provider states no window, so treat it
as effective from the call.

`404` here means either the conversation or the AML subscription does not exist — the response
description covers both, so check which before assuming the subscription was already cancelled.

## Retention

AML findings live on the conversation and disappear with it: once the customer's retention policy expires
the conversation, `getAMLList` returns **410 Gone**. If you have a regulatory obligation to keep screening
records, copy them into your own system when the callback arrives. Do not treat Authologic as your
archive.

## Compliance handling

- `FRAUD_SUSPICION` and `PROVIDER_BLACKLIST`, if they appear, are internal-only by provider instruction —
  never surface them to the subject.
- AML findings are special-category personal data under GDPR. Authologic operates under a published
  eIDAS Trust Service Policy (see `conformance/authologic-conformance.yml`); your own lawful basis and
  retention obligations are separate and yours.
