# Decisions and open questions

Product decisions now **resolved**, one **deferred**, a table of our open
**requests** to Ecobank's core API team (what VA Portal needs, what's provided
today, and what we're asking for), and one item that's a product/roadmap question
rather than an API one. Several places elsewhere in this doc set previously said
"provisional" or "proposed default" pending the resolved items below — those have
been updated to reflect the decision; this page is the record of why.

## Product decisions

Resolved on 2026-10-12.

1. **Should the pool account show a balance?** → **No.** Not shown, regardless of
   whether core ever exposes a balance endpoint. See the
   [persistence decision](../architecture/persistence-decision.md#what-persists-what-doesnt-and-how-it-stays-fresh)
   table and the [MVP scope](../product/overview.md#mvp-scope) pool account card. This
   also takes pool balance off the [core API blocking list](../integration/core-gaps.md#priority)
   — it's a product decision, not something waiting on Ecobank.
2. **When a VA moves outlets, do past transactions stay with the old outlet?** →
   **Yes**, confirmed. The working assumption stands: the outlet snapshot on each
   transaction (`va_outlet_history` and the `outlet_id` column on `va_transaction` —
   see [data model](../architecture/data-model.md#va_outlet_history)) is the permanent
   design, not a placeholder.
3. **Who approves VA requests and domiciliary accounts, and how fast?** → **Bank
   admin**, and the SLA is **as fast as possible** — no multi-day approval queue to
   design for. See [security](../engineering/security.md#authorization). (This does
   not change the general maker-checker rule for merchant approve/activate/deactivate,
   which still needs a separate approver role — this decision is specific to VA
   requests and linked accounts.)
4. **How long are transactions and the activity log kept, and how far back can an
   export go?** → **7 years**, for both `va_transaction` and `activity_log` — see
   [data model](../architecture/data-model.md#va_transaction). (The integration log's
   90-day retention is a separate, shorter-lived operational log and wasn't part of
   this question — see [observability](../engineering/observability-resilience.md).
   Export page-size limits, e.g. the CSV export's 12-month max range, are a separate
   ergonomics question from retention and are unchanged.)

## Deferred (not decided, not blocking MVP)

- **Who receives SMS alerts, and does the bank charge for them?** Explicitly set
  aside for now — not a decision, just not being made yet. `notification_preference`
  ships with its existing email/in-app defaults; SMS opt-in/opt-out behavior is
  undefined until this comes back up.

## Requests for Ecobank

What VA Portal needs from the core API team that it doesn't fully have yet. See
[core API request — MVP](../integration/core-api-requirements-mvp.md) for the short
version meant to be handed over directly — this is the fuller internal version, with
links into the rest of the doc set.

| VA Portal needs | Provided? | What's provided | Request |
|---|---|---|---|
| Create, update, and get a merchant (service provider) | ❌ No | — | Full API — or, if it stays manual, confirm turnaround time and how `serviceProviderId` gets back to us. The biggest item here — see [core gaps](../integration/core-gaps.md), row 1, and its effect on [Sprint 6](delivery-plan.md#reading-this-plan-against-the-gaps) |
| Generate a VA on demand | ❌ No | — | New endpoint, plus clarity on how many VAs are pre-allocated per merchant today (static VAs currently assume a pre-allocated pool via assign-static, core endpoint #6) |
| Reach transaction history and fetch VA on a real host | ⚠️ Partial | Both endpoints exist, but only on an internal IP; transaction history's production path is incomplete | Public UAT host for both, and the full production path for transaction history — blocks our integration testing regardless of code readiness. See [core API mapping](../integration/core-api-mapping.md#endpoint-map), endpoints #9–#11 |
| A webhook payload with amount, currency, date, and status | ⚠️ Partial | Webhook fires on payment, but without those fields | Add the fields, a signature we can verify, and the retry schedule. Affects [webhook security](../engineering/security.md#webhook), which relies on a secret path segment plus mandatory requery as a stand-in today |
| Full list of `transactionStatus` / `transactionRespCode` meanings | ⚠️ Partial | Only `TRANS_IN_PROGRESS` and `TRANS_COMPLETED` documented | Full list including reversals — needed to finish the status mapper in [data model](../architecture/data-model.md#va_transaction) |
| Affiliate codes, base URLs, and credentials for other launch countries | ❌ No | Nigeria (`ENG`) only | Togo, Ghana, Benin, and others — needed before `country_config` rows can exist for them. See [multi-country model](../product/overview.md#multi-country-model) |
| Rate limits, timeouts, and `requestId` uniqueness rules | ❓ Not documented | — | Governs the Resilience4j defaults in [observability](../engineering/observability-resilience.md#resilience), which are conservative guesses today |
| Confirmation of the credential model | ❓ Not documented | — | One platform credential per affiliate, or one per merchant? We're assuming one per affiliate in [security](../engineering/security.md#core-integration) until confirmed |
| Know which endpoints require AES-GCM encrypted bodies | ❓ Unclear | Spec says all payloads are encrypted; nearly every sample we've seen is plain JSON | Confirmation per endpoint — see [core API mapping](../integration/core-api-mapping.md#payload-inconsistencies-the-core-client-must-normalize) |
| Understand fetch VA vs. fetch all VA | ❓ Unclear | Both exist and look similar | What distinguishes them, and whether either already returns `status`, `type`, `createdDate`, or a total count — see endpoint #9 vs. #10 in [core API mapping](../integration/core-api-mapping.md) |

## Separately — not a core API question

**Scope and timing of the bank team view.** The source deck marks this "Next" — see
[product overview](../product/overview.md#who-uses-it). This is a product/roadmap
decision for Ecobank's business side, not something for the API team. This spec
designs the admin APIs now (onboarding approval needs them) but leaves the
bank-facing UI itself for a later release.
