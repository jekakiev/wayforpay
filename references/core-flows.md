# Core Flows

## Contents

- Hosted checkout with `Purchase`
- Callback handling on `serviceUrl`
- Status reconciliation with `CHECK_STATUS`
- Refunds and voids with `REFUND`
- Operational rules for idempotency and repair

## Default flow

Use the standard hosted WayForPay payment page unless the user explicitly needs widget, host2host, token, or regular-payment flows.

Baseline flow:

1. Create the order in the merchant backend.
2. Generate the Purchase payload and `merchantSignature` on the backend.
3. Send the customer to `POST https://secure.wayforpay.com/pay`.
4. Receive the callback on `serviceUrl`.
5. Verify callback signature before changing any order state.
6. Respond with signed acknowledgment payload.
7. If callback is delayed, repeated, or missing, use `CHECK_STATUS`.
8. If the payment must be reversed, use `REFUND` from the backend.

Never treat `returnUrl` as the final payment truth. Final state must come from backend callback or backend reconciliation.

## Purchase hosted checkout

Official doc: https://wiki.wayforpay.com/en/view/852102

Endpoint:

- `POST https://secure.wayforpay.com/pay`

Core required fields for a normal hosted payment:

- `merchantAccount`
- `merchantDomainName`
- `merchantTransactionSecureType`
- `merchantSignature`
- `orderReference`
- `orderDate`
- `amount`
- `currency`
- `productName[]`
- `productPrice[]`
- `productCount[]`

Commonly useful optional fields:

- `merchantAuthType`
- `merchantTransactionType`
- `language`
- `returnUrl`
- `serviceUrl`
- `apiVersion`
- `clientFirstName`
- `clientLastName`
- `clientEmail`
- `clientPhone`
- `paymentSystems`

Practical guidance:

- Generate the request on the backend only.
- Keep `orderReference` unique in your system and reuse it for reconciliation.
- Use backend-owned amount, currency, and cart data, not client-sent values.
- `apiVersion=2` is useful when you need richer callback or status response data.
- `merchantTransactionType` defaults to `AUTO`; use `AUTH` only when you intentionally need delayed capture.

Purchase signature base string:

`merchantAccount;merchantDomainName;orderReference;orderDate;amount;currency;productName[0..n];productCount[0..n];productPrice[0..n]`

## Callback handling on `serviceUrl`

Official doc: https://wiki.wayforpay.com/en/view/852102

WayForPay sends payment result data to `serviceUrl` for authorized and verified orders and for order status changes.

Treat the callback as the authoritative payment event.

Minimum callback rules:

- parse JSON from the request body
- verify `merchantSignature`
- look up order by `orderReference`
- apply idempotent state transition logic
- return signed acknowledgment payload

Callback acknowledgment shape:

```json
{
  "orderReference": "ORDER123",
  "status": "accept",
  "time": 1712059200,
  "signature": "..."
}
```

Practical guidance:

- Return acknowledgment only after signature validation and safe persistence.
- Store raw callback payload for audit/debug work when possible.
- Expect repeated callbacks and make order updates idempotent.
- If the callback handler is temporarily unavailable, reconcile later with `CHECK_STATUS`.

The docs state that WayForPay expects a signed response from the merchant server. If the correct response is not accepted, retries can continue for an extended period, so duplicate-safe handling is required.

## `CHECK_STATUS`

Official doc: https://wiki.wayforpay.com/en/view/852117

Endpoint:

- `POST https://api.wayforpay.com/api`

Use `CHECK_STATUS` when:

- callback delivery is delayed or missing
- payment sits in a non-terminal status
- you need a cron-based repair path
- you need a backend-only verification before changing order state
- you need to investigate support issues or chargeback/dispute claims

Core request fields:

- `transactionType=CHECK_STATUS`
- `merchantAccount`
- `orderReference`
- `merchantSignature`
- `apiVersion`

Signature base string:

`merchantAccount;orderReference`

Practical guidance:

- Prefer `apiVersion=2` when you need expanded response data such as additional fields, delivery, or comments.
- Use the same `orderReference` as your internal reconciliation key.
- Do not poll aggressively in a tight loop; use bounded retry logic or scheduled repair jobs.

## `REFUND`

Official doc: https://wiki.wayforpay.com/en/view/852115

Endpoint:

- `POST https://api.wayforpay.com/api`

Use `REFUND` for cancellation of payment or return of funds from the backend only.

Core request fields:

- `transactionType=REFUND`
- `merchantAccount`
- `orderReference`
- `amount`
- `currency`
- `comment`
- `merchantSignature`
- `apiVersion`

Optional line-item fields:

- `productName[]`
- `productPrice[]`
- `productCount[]`

Signature base string:

`merchantAccount;orderReference;amount;currency`

Practical guidance:

- Validate refund amount against your own order ledger before sending the request.
- Keep refund initiation backend-only; never let the client assemble the gateway request.
- Treat refund response as a separate status transition, not as a replay of the original payment.
- Use the response signature rules from `signatures-and-statuses.md` when validating the gateway response.

## Operational rules

- Payment success is determined from backend callback or status reconciliation, never from frontend redirect.
- Write idempotent handlers for callbacks, `CHECK_STATUS`, and refund updates.
- Persist both gateway status and internal business status; do not collapse them blindly.
- Non-terminal statuses should remain reconcilable and should not unlock the product/service as paid.
- Build at least one repair path, typically a scheduled `CHECK_STATUS` job for stale or missing callbacks.
