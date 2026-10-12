# Product overview

## What it is

The Merchant Virtual Account Portal is a Spring Boot backend and web dashboard that
lets Ecobank merchants see their pool account, the virtual accounts (VAs) attached to
it, and every transfer received into those VAs. It sits between the Ecobank Virtual
Account System (VAS) core APIs and the frontend.

**It does not move money.** No payouts, refunds, or transfers out — read and
account-management operations only (viewing balances/transactions, generating VAs,
blocking/unblocking, renaming, reassigning outlets).

The MVP goal is the merchant VA dashboard. Onboarding, admin, and settings ship in the
same codebase, but the dashboard is the first thing that must be production-ready —
see the [delivery plan](../process/delivery-plan.md).

## Who uses it

| Role | Who | What they can do |
|---|---|---|
| Bank super admin | Ecobank product owner | Configure countries, manage bank users, see every merchant |
| Bank admin (maker) | Ecobank onboarding / ops staff | Invite merchants, review KYC, request activation or deactivation |
| Bank approver (checker) | Ecobank supervisor | Approve or reject merchants, domiciliary accounts, and VA requests |
| Merchant admin | Business owner or finance lead | Full merchant view: pool account, all VAs, outlets, team, linked accounts, settings |
| Outlet admin | Staff at one or more outlets | View-only access to the VAs and transactions of assigned outlets |

The source deck marks the **bank team view** as "Next," with scope to be confirmed.
This spec designs the admin APIs now (onboarding approval needs them) but leaves the
bank-facing UI for a later release. See [open questions](../process/open-questions.md)
for the scope/timing question still outstanding.

## MVP scope

- **Overview** — pool account card, four KPI cards (total received, transactions,
  virtual accounts, average transfer), recent transactions, top VAs by value, a value
  and volume chart (7 days, month, year).
- **Pool account and transactions** — account details, status bar (successful,
  pending, reversed, failed), search, filters, CSV export.
- **Virtual accounts** — list with counts (total, active, static, dynamic), filters
  (status, type, outlet, created date), Excel export, generate VA (merchant admin
  only), detail page, block/unblock, rename, reassign outlet.
- **Outlets** — group VAs by location, totals per outlet, alert for unassigned VAs.
- **Notifications** — in-app feed plus email and SMS preferences.
- **Settings** — profile, password, 2FA, sessions, team and access, linked
  (domiciliary) accounts.
- **Onboarding** — admin sends invite link, merchant signs up, completes KYC, invites
  team; admin approves, onboards, or deactivates.
- **Activity log** — every user and admin action recorded.

## Out of scope for MVP

- Payouts, refunds, reversals initiated from the portal
- Bank team UI (APIs only)
- Native mobile apps
- Multi-country rollout (see below — the *model* supports it from day one; actually
  operating in more than one country is a later release)

## Multi-country model

Ecobank operates in Nigeria, Togo, Ghana, Benin, and other markets, and the product
must work in all of them. Every merchant, account, and core call carries an
`affiliateCode` (this spec uses `ENG` for Nigeria as the running example).

Country specifics — currency, timezone, KYC document types, phone format, core
credentials, and base URL — live in a `country_config` table (see
[data model](../architecture/data-model.md)), not in code. **Adding a country is a
configuration change plus KYC rules, not a release.**

Affiliate codes, base URLs, and credentials for Togo, Ghana, and Benin are not yet
known — tracked in [open questions](../process/open-questions.md).
