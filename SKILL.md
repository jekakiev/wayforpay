---
name: wayforpay-integration
description: Guides WayForPay integration decisions for hosted checkout, callback verification, payment status checks, refunds, signature validation, two-stage payments, token-based flows, and review/debug work. Use this skill when building, modifying, debugging, or reviewing any WayForPay integration.
---

# WayForPay Integration

Use this skill for merchant-side WayForPay payment integrations and code reviews.

## Source of truth

Always use official WayForPay documentation as the authority:

- Main docs: https://wiki.wayforpay.com/en/
- Payments overview: https://wiki.wayforpay.com/en/view/852100
- Purchase: https://wiki.wayforpay.com/en/view/852102
- Settle: https://wiki.wayforpay.com/en/view/852113
- Refund: https://wiki.wayforpay.com/en/view/852115
- Check Status: https://wiki.wayforpay.com/en/view/852117
- Response codes and statuses: https://wiki.wayforpay.com/en/view/852131
- Tokenization: https://wiki.wayforpay.com/en/view/852175
- Verify: https://wiki.wayforpay.com/en/view/852189
- Charge host2host: https://wiki.wayforpay.com/en/view/852194
- Payment Widget: https://wiki.wayforpay.com/en/view/852091
- Regular payments: https://wiki.wayforpay.com/en/view/852496

If code, memory, and docs disagree, follow the docs.

## When to open each reference

- Open [references/core-flows.md](references/core-flows.md) for the default merchant flow: hosted checkout, callback handling, `CHECK_STATUS`, `REFUND`, and operational reconciliation.
- Open [references/signatures-and-statuses.md](references/signatures-and-statuses.md) when validating HMAC inputs, debugging signature mismatches, or mapping gateway statuses to merchant-side order states.
- Open [references/adjacent-features.md](references/adjacent-features.md) only when the user needs `AUTH`/`SALE`, `Settle`, widget integration, `Verify`, tokenization, `Charge`, or regular payments.

## Routing

| User need | Start here | Primary WayForPay method |
| --- | --- | --- |
| Standard hosted checkout | `references/core-flows.md` | `POST https://secure.wayforpay.com/pay` |
| Payment status verification | `references/core-flows.md` | `POST https://api.wayforpay.com/api` with `CHECK_STATUS` |
| Refund or void | `references/core-flows.md` | `POST https://api.wayforpay.com/api` with `REFUND` |
| Debug signature mismatch | `references/signatures-and-statuses.md` | HMAC-MD5 validation |
| Map gateway status to order state | `references/signatures-and-statuses.md` | status handling |
| Hold then capture later | `references/adjacent-features.md` | `AUTH` + `SETTLE` |
| Token-based or one-click payment | `references/adjacent-features.md` | tokenization + `Charge` |
| Popup checkout instead of redirect | `references/adjacent-features.md` | Payment Widget |
| Card verification without a normal purchase | `references/adjacent-features.md` | `Verify` |
| Scheduled recurring billing | `references/adjacent-features.md` | regular payments API |

## Default implementation path

Prefer the standard hosted payment page flow unless the user explicitly needs another WayForPay product:

1. Create the order in the merchant backend.
2. Build the Purchase payload and generate `merchantSignature` on the backend.
3. Submit the customer to `https://secure.wayforpay.com/pay`.
4. Receive the server-to-server callback on `serviceUrl`.
5. Verify the callback signature before changing order state.
6. Use `transactionStatus` and backend reconciliation, not `returnUrl`, as the payment truth.
7. Use `CHECK_STATUS` for retries, repair jobs, delayed statuses, and dispute investigation.
8. Use `REFUND` only from the backend after validating business rules and refund amount.

## Review checklist

Check these failure modes first:

- purchase signature fields are concatenated in the wrong order
- arrays in the purchase signature are joined incorrectly
- the app marks the order paid from frontend redirect instead of callback or status check
- callback handling is not idempotent
- duplicate callbacks can create duplicate side effects
- `CHECK_STATUS` is missing as an operational repair path
- refund logic is exposed to the client
- `AUTH`/`SALE`/`SETTLE` semantics are mixed up
- status mapping treats non-terminal statuses as success

## Working style

Keep the root skill short. Load only the reference files relevant to the task at hand.
