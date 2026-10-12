# Decisions and open questions

Three lists: product decisions now **resolved**, one deferred, and the API gaps still
open with Ecobank's integration team. Several places elsewhere in this doc set
previously said "provisional" or "proposed default" pending the resolved items below —
those have been updated to reflect the decision; this page is the record of why.

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

## Questions for Ecobank

Still open — none of these were addressed by the product decisions above.

1. **Will a create / get / update merchant API be exposed, or does onboarding stay
   manual?** The single biggest gap — see
   [core gaps](../integration/core-gaps.md), row 1, and its effect on
   [Sprint 6](delivery-plan.md#reading-this-plan-against-the-gaps).
2. **How does a merchant get a new VA on demand, and how many are pre-allocated per
   merchant?** Affects the "generate virtual account" flow — static VAs currently
   assume a pre-allocated pool via assign-static (core endpoint #6).
3. **Which endpoints actually require AES-GCM encrypted bodies?** The spec text says
   all payloads are encrypted, but every sample except transaction history is plain
   JSON — see
   [core API mapping](../integration/core-api-mapping.md#payload-inconsistencies-the-core-client-must-normalize).
4. **Full list of `transactionStatus` values and `transactionRespCode` meanings,
   including reversals.** Blocks finishing the status mapper in
   [data model](../architecture/data-model.md#va_transaction).
5. **Can the webhook include amount, currency, transaction date, and an HMAC
   signature? What is the retry policy?** Directly affects
   [webhook security](../engineering/security.md#webhook), which currently relies on
   a secret path segment plus mandatory requery as a stand-in.
6. **Correct production path for transaction history, and public UAT hosts for
   history and fetch VA.** Both are currently only reachable via internal IPs — see
   [core API mapping](../integration/core-api-mapping.md#endpoint-map), endpoints
   #9–#11. Blocks integration testing for VA sync and reconciliation regardless of
   code readiness.
7. **Difference between fetch VA and fetch all VA; can they return status, type,
   created date, and total count?** See endpoint #9 vs. #10 in
   [core API mapping](../integration/core-api-mapping.md).
8. **Affiliate codes, base URLs, and credentials for Togo, Ghana, Benin, and other
   launch countries.** Needed before `country_config` rows can exist for anything
   beyond Nigeria (`ENG`) — see
   [multi-country model](../product/overview.md#multi-country-model).
9. **Rate limits, timeouts, and `requestId` uniqueness rules.** Governs the
   Resilience4j defaults in
   [observability](../engineering/observability-resilience.md#resilience), which are
   currently conservative guesses.
10. **One API credential for the platform, or one per merchant?** Affects token
    caching and key management in
    [security](../engineering/security.md#core-integration); the spec currently
    assumes one platform credential per affiliate.
11. **Scope and timing of the bank team view.** The source deck marks this "Next" —
    see [product overview](../product/overview.md#who-uses-it). This spec designs the
    admin APIs now (onboarding approval needs them) but leaves the bank-facing UI for
    a later release.
