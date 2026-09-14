---
name: authologic-retrieve-bank-transactions
description: >-
  Retrieve a user's bank accounts, transactions and transaction statistics through Authologic's
  PSD2-backed Bank Transactions product, and read the derived affordability (source of funds/wealth)
  assessment. Use for income verification, affordability checks and source-of-funds review.
api: Authologic Customer API
version: '1.1'
generated: '2026-09-14'
method: generated
source: >-
  Grounded in openapi/authologic-customer-api-openapi.yml (operationIds verified against the spec) and
  https://developer.authologic.com/docs/products/bank-transactions.md,
  /docs/products/bank-transactions/api-integration.md, /docs/products/affordability.md,
  /docs/integration/testing.md.
operations:
  - createConversation
  - getConversation
  - getBankTransactionsAccounts
  - getBankTransactions
  - getBankTransactionsStats
  - getAffordability
---

# Retrieve bank transactions and affordability

## What this product is

The user authenticates with their bank (a PSD2 account-information flow through an upstream aggregator
such as Kontomatik) and consents to share account data. Authologic normalises the result onto the
conversation. This is consented access to a person's financial history — treat it accordingly.

## Step 1 — create the conversation with the bank product

`POST /api/conversations` with a bank-transactions section in `query`, plus your `callbackUrl` and
`returnUrl`. Redirect the user to the returned `url`; they pick their bank and authorise sharing.

Standard rules apply: Basic auth, `application/vnd.authologic.v1.1+json` on Content-Type and Accept, no
idempotency key, HTTP 402 if the plan is exhausted.

A conversation that combines identity and bank transactions may emit **two** callbacks — one when the
identity data lands and another when transaction data is ready. Wait for `status: FINISHED` before
treating the dataset as complete.

The common failure here is `BANK_NO_ACCOUNTS_SHARED` — the user did not select an account, or the bank
did not release one. That is a re-invite, not a technical retry.

## Step 2 — list the accounts (`getBankTransactionsAccounts`)

`GET /api/conversations/{conversationId}/bankTransactions/accounts`

Start here. `accountId` is the join key for everything below.

Note the deprecation: `accounts.iban` was removed. The account identifier is `accounts.accountId`
(and, for data verification, `verify.user.ids.accounts.accountId`). If your integration still reads an
IBAN field, it is broken.

## Step 3 — page the transactions (`getBankTransactions`)

`GET /api/conversations/{conversationId}/bankTransactions?page=&pageSize=`

Page-number pagination, no cursor, no `Link` header. Each `Transaction` carries `id`, `accountId`,
`senderAccountId` and `recipientAccountId`, so you can reconstruct counterparties across accounts.

For a large history this is the only bulk read in the API — walk it once, persist it, and do not re-page
it on every request. There is no server-side filter beyond pagination.

## Step 4 — statistics instead of raw rows (`getBankTransactionsStats`)

`GET /api/conversations/{conversationId}/bankTransactions/stats?dateFrom=&dateTo=`

Aggregates per account over a date range. Dates are RFC 3339 (`2026-04-02` for a bare date,
`2026-04-02T09:20:50Z` with a time). Prefer this over paging thousands of rows when you only need
inflow/outflow shape.

## Step 5 — affordability (`getAffordability`)

`GET /api/conversations/{conversationId}/affordability/info`

Returns the source-of-funds / source-of-wealth assessment derived from the collected data. Request the
affordability product in `query` at creation time — it is a separate product, not a free side effect of
the bank product.

## Testing

Use the sandbox (`https://sandbox.authologic.com`) with `strategy: "public:sandbox"` for the flow itself.
For a realistic PSD2 run, Kontomatik publishes test accounts — select 'Konto Bank' and use
<https://developer.kontomatik.com/coverage?resource=test-accounts>. Authologic publishes no bank test
credentials of its own.

## Data handling

- This is financial personal data under GDPR. Pull it, use it for the decision, and do not park it in a
  model context or a log.
- Retention is the customer's policy and is not published. Once the conversation expires,
  every one of these reads returns **410 Gone**. If you need the transaction record for a regulatory
  file, copy it into your own system when the callback arrives.
- `deleteConversation?mode=DELETE_DATA` erases it immediately and irreversibly — that is your data-subject
  erasure path, and the pending-callback caveat applies.
