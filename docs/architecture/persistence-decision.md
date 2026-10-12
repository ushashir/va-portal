# Persistence decision: what the portal stores

## Decision

The portal persists its own data **plus a synced read model of VAs and transactions**.
Ecobank core stays the source of truth for money; the portal is the source of truth
for everything core does not know about. A pure pass-through portal cannot render the
dashboard the product spec asks for.

## Why pass-through does not work

- **No aggregates in core.** KPI cards, the value/volume chart, top VAs, and the
  status bar all need sums and counts over a period. Core has no endpoint for this.
- **History is per-VA, per-date-range.** A merchant with 22 VAs and a 90-day view
  means 22 encrypted history calls per page load, then summing in memory. At hundreds
  of merchants this will hit core rate limits and make the dashboard slow.
- **Fetch VA returns only number and name.** Status, type (static/dynamic), outlet,
  created date, and expiry — everything the VA list filters on — are not returned by
  that call. See [core API mapping](../integration/core-api-mapping.md).
- **Core has no concept of** outlets, users, teams, invitations, KYC, notifications,
  or audit trails. These must live in the portal.
- **Merchant creation is manual at Ecobank.** The onboarding workflow and its state
  have nowhere else to live but the portal.

## What persists, what doesn't, and how it stays fresh

| Data | Persist? | Source of truth | How it stays fresh |
|---|---|---|---|
| Users, roles, sessions, MFA secrets, invitations | Yes | Portal | Portal writes |
| Merchant profile, KYC documents, onboarding state, approvals | Yes | Portal (core holds the service provider record) | Portal writes; `serviceProviderId` stored once Ecobank provisions it |
| Outlets and VA-to-outlet assignment history | Yes | Portal | Portal writes |
| Virtual account registry (number, name, type, status, outlet, expiry) | Yes, as a mirror | Core | Written on create/block/unblock/rename/assign-static; nightly sync from fetch VA |
| Orders (dynamic VA requests) | Yes | Core | Written on create-order; status from webhook or requery |
| Transactions (amount, payer, status, reference, dates) | Yes, as a mirror | Core | Webhook on arrival; requery for pending; nightly reconciliation from transaction history |
| Daily aggregates per merchant, outlet, VA | Yes, derived | Portal | Updated on each transaction write; rebuilt by reconciliation |
| Notifications and preferences | Yes | Portal | Portal writes |
| Activity log (business audit) | Yes, append-only | Portal | Written by an aspect on every state-changing call |
| Integration log (each core call and webhook, masked) | Yes, 90-day retention (proposed) | Portal | Written by the core client |
| Linked domiciliary accounts and their approval | Yes | Portal until a core API exists | Portal writes |
| Pool account balance | **No** | Core | Not exposed by any core endpoint — [open question 1](../process/open-questions.md) |
| Core OAuth token | No (memory or Redis only) | Core | Refreshed before expiry |
| AES key, client secret, core passwords | No (secrets manager) | Ecobank | Rotated per Ecobank's process |

## Rules that follow from this

1. **Money fields shown on screen always come from a stored transaction that came
   from core.** The portal never computes a balance it did not receive.
2. **Every mirrored row keeps `core_synced_at` and `source`** (`WEBHOOK`, `REQUERY`,
   `RECON`), so a mismatch can be traced.
3. **A transaction is identified by `transactionRefNo` + `virtualAccountNumber`**
   (unique constraint). Webhook retries and reconciliation upsert — never duplicate.
4. **Pending transactions older than 10 minutes** (proposed) are requeried via the "get
   VA transaction status" endpoint.
5. **Nightly reconciliation** pulls the previous day's history per active VA, fixes
   drift, and raises an alert if more than 0.5% of rows (proposed) differ.
6. **On any read where freshness matters** (VA detail, single transaction), the portal
   can offer a "refresh from bank" action that calls core and updates the mirror.

For the tables these rules apply to, see [data model](data-model.md). For the
endpoints behind "webhook," "requery," and "nightly sync," see
[core API mapping](../integration/core-api-mapping.md).
