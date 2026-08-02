---
name: Place a customer and debt, then track it
description: Onboard a consumer with their contact info and debt to TrueAccord for recovery, then look the customer up to monitor progress.
api: openapi/trueaccord-recover-openapi.yml
operations: [addCustomers, listCustomers, getCustomer, addContactInformation]
---

# Place a customer and debt, then track it (TrueAccord Recover API)

Use the TrueAccord Recover API to place a consumer and their debt for
digital-first collection, then monitor the account.

## Authentication
- HTTP Basic over HTTPS. Send your API key as the **username**; leave the
  password empty but keep the trailing colon: `-u $YOUR_KEY:`.
- For multi-creditor accounts, send `X-TA-CREDITOR: $CREDITOR_ID` on every
  request or you will get a `403`.

## Steps
1. **Place the customer and debt** — `addCustomers` (`POST /customers/`).
   - Body: `{ "customers": [ { name, addresses[], emails[], phones[], debts[] } ] }`.
   - Each debt needs a unique `transactionId`. Money is `{ amount, currency }`
     in minor units; dates are `{ day, month, year }`.
   - Set `?addDebtsIfPossible=true` to attach a debt to a customer that already
     exists instead of failing on a duplicate `reference`.
   - For large loads use `addCustomersBatch` (`POST /customersBatch/`) which
     returns a per-record result so one bad row does not fail the batch.
2. **Confirm placement** — `listCustomers` (`GET /customers/?reference={ref}`)
   or `getCustomer` (`GET /customers/{customerId}`) to read back the created
   customer and its `metadata`.
3. **Keep contact info fresh** — `addContactInformation` (`PATCH /customers/{customerId}`)
   appends new addresses/phones/emails without replacing existing ones.

## Rules an agent must follow
- **Idempotency:** re-sending a debt with an existing `transactionId` returns
  `409 Conflict` and does not double-place it. Reuse the same `transactionId`
  when retrying, never a new one.
- **Pagination:** `listCustomers` returns at most 100 per page; page with
  `count` + `offset` and read `totalResults`.
- **Errors** come back as `{ code, msg, errorInfo }` (not RFC 9457). On `400`,
  read `errorInfo` for the invalid fields.
- Placing a debt starts real collection activity against a consumer — treat
  `addCustomers` as a high-consequence write and require explicit purpose.
