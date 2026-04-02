# Adjacent Features

## Contents

- `AUTH` vs `SALE`
- `Settle` for delayed capture
- `apiVersion` guidance
- Payment Widget
- `Verify`
- tokenization and one-click
- `Charge` host2host
- regular payments overview

These are useful extensions of the main flow, not the default path.

## `AUTH` vs `SALE`

Docs:

- Payments overview: https://wiki.wayforpay.com/en/view/852100
- Purchase: https://wiki.wayforpay.com/en/view/852102

Use `SALE` when you want one-step payment: authorization plus withdrawal in one flow.

Use `AUTH` when you want to hold funds first and capture later with `SETTLE`.

Practical cautions:

- Do not choose `AUTH` unless you have a real delayed-capture business need.
- `AUTH` requires explicit downstream handling for capture or release.
- A codebase that assumes all successful payments are immediately settled will break on `AUTH`.

## `Settle`

Doc: https://wiki.wayforpay.com/en/view/852113

Use `SETTLE` when the original `Purchase` or `Charge` was created with `merchantTransactionType=AUTH` and you now want to withdraw the blocked amount.

Endpoint:

- `POST https://api.wayforpay.com/api`

Core cautions:

- `SETTLE` is only relevant for two-stage payment flows.
- Track authorization state separately from paid state.
- Validate amount before capture, especially if partial capture behavior matters in your domain.
- If capture is no longer needed, use `REFUND`/void path according to the current transaction state.

## `apiVersion`

Docs:

- Purchase: https://wiki.wayforpay.com/en/view/852102
- Check Status: https://wiki.wayforpay.com/en/view/852117

`apiVersion=2` is useful when you need expanded data returned on callbacks or status responses, such as additional fields, delivery, or comments.

Practical cautions:

- Do not declare version-specific fields in code unless you actually request the matching API version.
- Keep your parser tolerant to extra fields from WayForPay.
- If a project does not use advanced fields, version `1` is simpler.

## Payment Widget

Doc: https://wiki.wayforpay.com/en/view/852091

Use the widget when the merchant wants a popup payment form instead of redirecting the user to a separate hosted page.

Practical cautions:

- Widget is still WayForPay-handled payment entry; do not move sensitive card handling into your own frontend.
- Keep backend signature generation and callback verification exactly as strict as with hosted checkout.
- Use widget only when the UX gain is worth the extra frontend integration surface.

## `Verify`

Doc: https://wiki.wayforpay.com/en/view/852189

Use `Verify` when the goal is card verification or token acquisition, not a normal customer checkout.

Practical cautions:

- Verification can involve card hold or lookup/3DS-like confirmation depending on verify mode.
- Token acquisition via `Verify` should be treated as a payment-method setup flow, not as a paid order.
- Make sure downstream business logic does not confuse successful verification with successful purchase.

## Tokenization and one-click

Doc: https://wiki.wayforpay.com/en/view/852175

WayForPay can return `recToken` after successful `Purchase` or `Verify`. That token can later be used for token-based charging.

Use this flow when:

- the merchant needs one-click repeat payments
- the product has saved-card behavior
- regular billing starts from an initial customer-approved payment

Practical cautions:

- Guard token usage as a sensitive payment credential even if it is not raw PAN data.
- Separate customer consent and business authorization from mere token possession.
- Token-based charging changes your risk model; require stronger audit trails and idempotency.

## `Charge` host2host

Doc: https://wiki.wayforpay.com/en/view/852194

Use `Charge` when the merchant intentionally needs host2host card charging or token-based charging instead of hosted payment page flow.

Endpoint:

- `POST https://api.wayforpay.com/api`

Practical cautions:

- This is not the default integration path; hosted checkout is usually safer and simpler.
- `Charge` supports both direct card fields and `recToken`, which increases implementation sensitivity.
- If using `AUTH` with `Charge`, ensure your settlement path is fully implemented.

## Regular payments

Docs:

- Overview: https://wiki.wayforpay.com/en/view/852496
- Status: https://wiki.wayforpay.com/en/view/852526
- Suspend: https://wiki.wayforpay.com/en/view/852506
- Remove: https://wiki.wayforpay.com/en/view/852521

Regular payments use the `https://api.wayforpay.com/regularApi` family and rely on merchant-side configuration such as `merchantPassword`, not the standard `merchantSignature` request pattern used by Purchase/Check Status/Refund.

Use this path when the merchant needs scheduled billing rather than one-off repeat charges.

Practical cautions:

- Do not mix regular-payment API authentication rules with the standard payment API signature rules.
- Treat lifecycle states like `Active`, `Suspended`, `Created`, `Removed`, `Confirmed`, and `Completed` separately from one-off payment statuses.
- Keep support tooling for suspend/resume/remove actions if the product exposes subscription-like behavior.
