# Open questions

Two lists: product decisions the portal team or Ecobank's business side needs to
make, and API gaps to take back to Ecobank's integration team. Several architecture
choices elsewhere in this doc set are provisional pending answers here — each is
cross-linked below.

## Product questions

1. **Should the pool account show a balance?** No core endpoint exposes it today.
   Affects the pool account card in the
   [MVP scope](../product/overview.md#mvp-scope) and the
   [persistence decision](../architecture/persistence-decision.md#what-persists-what-doesnt-and-how-it-stays-fresh)
   table (pool account balance is explicitly marked "not persisted, not exposed").
2. **When a VA moves outlets, do past transactions stay with the old outlet?** This
   spec assumes **yes**, via the outlet snapshot on each transaction — see
   `va_outlet_history` and the `outlet_id` snapshot column on `va_transaction` in
   [data model](../architecture/data-model.md). Treat this as the working assumption,
   not a confirmed answer.
3. **Who approves VA requests and domiciliary accounts, and how fast?** Affects the
   maker-checker SLA and the `approval_request` workflow in
   [security](../engineering/security.md#authorization).
4. **Who receives SMS alerts, and does the bank charge for them?** Affects
   `notification_preference` defaults and whether SMS is opt-in or opt-out by role.
5. **How long are transactions and the activity log kept, and how far back can an
   export go?** The retention figures in
   [data model](../architecture/data-model.md#activity_log) (7 years) and
   [observability](../engineering/observability-resilience.md) (90 days for the
   integration log) are proposed defaults, not confirmed bank policy.

## Questions for Ecobank

6. **Will a create / get / update merchant API be exposed, or does onboarding stay
   manual?** The single biggest gap — see
   [core gaps](../integration/core-gaps.md), row 1, and its effect on
   [Sprint 6](delivery-plan.md#reading-this-plan-against-the-gaps).
7. **How does a merchant get a new VA on demand, and how many are pre-allocated per
   merchant?** Affects the "generate virtual account" flow — static VAs currently
   assume a pre-allocated pool via assign-static (core endpoint #6).
8. **Which endpoints actually require AES-GCM encrypted bodies?** The spec text says
   all payloads are encrypted, but every sample except transaction history is plain
   JSON — see
   [core API mapping](../integration/core-api-mapping.md#payload-inconsistencies-the-core-client-must-normalize).
9. **Full list of `transactionStatus` values and `transactionRespCode` meanings,
   including reversals.** Blocks finishing the status mapper in
   [data model](../architecture/data-model.md#va_transaction).
10. **Can the webhook include amount, currency, transaction date, and an HMAC
    signature? What is the retry policy?** Directly affects
    [webhook security](../engineering/security.md#webhook), which currently relies on
    a secret path segment plus mandatory requery as a stand-in.
11. **Correct production path for transaction history, and public UAT hosts for
    history and fetch VA.** Both are currently only reachable via internal IPs — see
    [core API mapping](../integration/core-api-mapping.md#endpoint-map), endpoints
    #9–#11. Blocks integration testing for VA sync and reconciliation regardless of
    code readiness.
12. **Difference between fetch VA and fetch all VA; can they return status, type,
    created date, and total count?** See endpoint #9 vs. #10 in
    [core API mapping](../integration/core-api-mapping.md).
13. **Affiliate codes, base URLs, and credentials for Togo, Ghana, Benin, and other
    launch countries.** Needed before `country_config` rows can exist for anything
    beyond Nigeria (`ENG`) — see
    [multi-country model](../product/overview.md#multi-country-model).
14. **Rate limits, timeouts, and `requestId` uniqueness rules.** Governs the
    Resilience4j defaults in
    [observability](../engineering/observability-resilience.md#resilience), which are
    currently conservative guesses.
15. **One API credential for the platform, or one per merchant?** Affects token
    caching and key management in
    [security](../engineering/security.md#core-integration); the spec currently
    assumes one platform credential per affiliate.
16. **Scope and timing of the bank team view.** The source deck marks this "Next" —
    see [product overview](../product/overview.md#who-uses-it). This spec designs the
    admin APIs now (onboarding approval needs them) but leaves the bank-facing UI for
    a later release.
