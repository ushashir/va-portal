# Delivery order

Build the dashboard first: data in (core client, webhook, sync), then data out
(dashboard, VA, transactions), then onboarding and admin. Proposed two-week sprints;
**durations to be confirmed once Ecobank answers the gaps** in
[open questions](open-questions.md).

| Sprint | Goal | Main deliverables |
|---|---|---|
| 1 | Foundations | Repo, CI/CD, three environments, common package, Flyway baseline, auth + RBAC + MFA, `EcobankClient` with token, cipher, logging; WireMock stubs for all 12 core endpoints |
| 2 | Data in | Webhook inbox, requery job, VA sync, per-VA history backfill, reconciliation, `daily_aggregate`, integration log |
| 3 | Dashboard (MVP target) | Overview KPIs, timeseries, top VAs, recent transactions, pool account page, status summary, transactions list + CSV export |
| 4 | VA management and outlets | VA list/detail/export, generate VA, block/unblock, rename, make static, outlets + reassignment, notifications |
| 5 | Onboarding and team | Invites, self sign-up, KYC upload, submit; team invites and outlet access; activity log UI |
| 6 | Admin and hardening | Approvals (maker-checker), provisioning, lifecycle, linked accounts, admin dashboard; load test, pen test fixes, UAT sign-off |

## Reading this plan against the gaps

Sprint 1 (`EcobankClient` + WireMock stubs for all 12 endpoints) can start immediately
— it does not depend on Ecobank answering anything, since WireMock stands in for core.

Sprint 2 (data in) is where the [core gaps](../integration/core-gaps.md) start to
bite: VA sync and reconciliation need real UAT access to fetch-VA and transaction
history (currently private IPs), and the webhook ingest is only as safe as whatever
authentication Ecobank eventually supplies. Treat Sprint 2's exit criteria as
provisional until those two are resolved.

Sprint 3 (the MVP target) is otherwise unblocked by core gaps — everything it reads
comes from the portal's own `daily_aggregate` and `va_transaction` tables, not live
core calls. The pool account card ships without a balance figure — confirmed, not
just a core-API gap; see [decisions](open-questions.md#product-decisions).

Sprint 6 depends on the merchant/service-provider API gap — if Ecobank doesn't expose
create/update/deactivate-merchant by then, provisioning and lifecycle screens stay
semi-manual (admin enters `serviceProviderId` by hand) rather than fully automated.
