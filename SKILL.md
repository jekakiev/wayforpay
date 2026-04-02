---
name: wayforpay-integration
description: Guides WayForPay integration decisions for hosted checkout, callback verification, payment status checks, refunds, signature validation, and operational safety. Use this skill when building, modifying, debugging, or reviewing any WayForPay integration.
---

# WayForPay Integration

Use this skill for WayForPay purchase flows, backend callback handling, status reconciliation, and refunds.

## Source of truth

Use official WayForPay docs as the authority and prefer them over memory:

- Main docs: https://wiki.wayforpay.com/en
- Purchase: https://wiki.wayforpay.com/en/view/852102
- Check Status: https://wiki.wayforpay.com/en/view/852117
- Refund: https://wiki.wayforpay.com/en/view/852115
- Response codes and statuses: https://wiki.wayforpay.com/en/view/852131

If the implementation and memory disagree, follow the docs and update the code accordingly.

## Default integration path

Prefer the standard hosted payment page flow unless the user explicitly needs another WayForPay product:

1. Create the order in the merchant backend.
2. Build the Purchase payload and generate `merchantSignature` on the backend.
3. Submit the user to `https://secure.wayforpay.com/pay`.
4. Receive the server-to-server callback on `serviceUrl`.
5. Verify the callback signature before changing order state.
6. Mark the order by `transactionStatus`, not by the frontend redirect.
7. Use `CHECK_STATUS` for reconciliation, retries, cron repair jobs, or dispute investigation.
8. Use `REFUND` only from the backend after validating business rules and refund amount.

## Core endpoints

- Purchase hosted checkout: `POST https://secure.wayforpay.com/pay`
- Check payment status: `POST https://api.wayforpay.com/api` with `transactionType=CHECK_STATUS`
- Refund or void: `POST https://api.wayforpay.com/api` with `transactionType=REFUND`

## Purchase checklist

For hosted checkout, expect these core fields at minimum:

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

- `merchantTransactionType`
- `returnUrl`
- `serviceUrl`
- `language`
- `apiVersion`
- `clientFirstName`
- `clientLastName`
- `clientEmail`
- `clientPhone`

Generate the Purchase signature as HMAC-MD5 over this semicolon-separated UTF-8 string:

`merchantAccount;merchantDomainName;orderReference;orderDate;amount;currency;productName[0..n];productCount[0..n];productPrice[0..n]`

## Callback handling

WayForPay sends transaction updates to `serviceUrl` with JSON payload. Treat this callback as the authoritative payment event.

Verify callback `merchantSignature` as HMAC-MD5 over:

`merchantAccount;orderReference;amount;currency;authCode;cardPan;transactionStatus;reasonCode`

WayForPay retries callbacks for up to 4 days if your server does not return the expected valid response.

Return a confirmation payload signed as HMAC-MD5 over:

`orderReference;status;time`

Expected response shape:

```json
{
  "orderReference": "ORDER123",
  "status": "accept",
  "time": 1712059200,
  "signature": "..."
}
```

## Status checks

Use `CHECK_STATUS` when:

- the callback was missed or delayed
- you need idempotent reconciliation
- an order remains pending too long
- you need a backend-only verification step

Generate the `CHECK_STATUS` request signature as HMAC-MD5 over:

`merchantAccount;orderReference`

## Refunds

Use `REFUND` for payment cancellation or funds return through the backend only.

Core request fields:

- `transactionType=REFUND`
- `merchantAccount`
- `orderReference`
- `amount`
- `currency`
- `comment`
- `merchantSignature`

Generate the `REFUND` request signature as HMAC-MD5 over:

`merchantAccount;orderReference;amount;currency`

Verify the refund response signature as HMAC-MD5 over:

`merchantAccount;orderReference;transactionStatus;reasonCode`

## Transaction statuses

Use official response codes docs for the full list. The statuses most often relevant in application logic are:

- `Approved`: payment succeeded
- `Pending`: under antifraud review
- `inProcessing`: payment still processing
- `WaitingAuthComplete`: hold succeeded, waiting for settle or void
- `Declined`: payment failed
- `Expired`: payment window expired
- `Refunded/Voided`: refund or void completed
- `RefundInProcessing`: refund accepted but not finished yet

## Guardrails

- Never mark an order as paid from `returnUrl` alone.
- Never trust frontend state for final payment status.
- Always verify every WayForPay signature on the backend.
- Always make `orderReference` unique and idempotent in your system.
- Always use HTTPS for callback and backend API handling.
- Prefer backend-generated amounts, currency, and line items over client-provided values.
- Treat duplicate callbacks and repeated status checks as normal; write idempotent handlers.
- When status is not terminal, persist the event and reconcile later instead of guessing.

## When reviewing code

Check for these failure modes first:

- signature base string assembled in the wrong field order
- arrays joined incorrectly when generating Purchase signature
- order state updated from redirect instead of callback
- missing idempotency around repeated callbacks
- refund route exposed to the client
- missing `CHECK_STATUS` repair path for operational recovery
- stale or incomplete status mapping
