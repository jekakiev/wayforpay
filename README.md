# wayforpay

Minimal `skills.sh`-ready repository for a WayForPay integration skill.

## Install

```bash
npx skills add jekakiev/wayforpay
```

## What it covers

- hosted checkout via WayForPay Purchase
- backend callback signature verification
- payment status reconciliation with `CHECK_STATUS`
- refunds and voids with `REFUND`
- practical review guardrails for production integrations

## Source of truth

This skill is based on the official WayForPay documentation:

- https://wiki.wayforpay.com/en
- https://wiki.wayforpay.com/en/view/852102
- https://wiki.wayforpay.com/en/view/852117
- https://wiki.wayforpay.com/en/view/852115
- https://wiki.wayforpay.com/en/view/852131

## Repo layout

```text
.
├── README.md
└── SKILL.md
```
