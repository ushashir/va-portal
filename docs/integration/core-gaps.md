# Core API gaps

What Ecobank core does not expose today, why the portal needs it, and the interim
workaround the portal uses in the meantime. This is the list to take back to Ecobank —
the corresponding questions are consolidated in
[open questions](../process/open-questions.md#questions-for-ecobank).

| Gap | Why the portal needs it | Interim workaround |
|---|---|---|
| Create / update / deactivate merchant (service provider) API | Self-serve onboarding and admin deactivate | Portal runs the workflow; Ecobank ops provisions manually and admin enters `serviceProviderId` |
| Get merchant (service provider) details | Show settlement account, status, amount rules | Store what was submitted at onboarding |
| Create new VA on demand (static or dynamic) for a merchant | "Generate virtual account" button | Use create-order for dynamic; static only from the pre-allocated pool via assign-static |
| Pool / settlement account details, name enquiry, balance | Pool account card, domiciliary name check | Show stored details; no balance (decided — see [decisions](../process/open-questions.md#product-decisions) — not just a core-API gap) |
| Pool account statement across all VAs | Union of stored VA transactions | Mirror in `virtual_account`, updated on each portal action |
| Single VA detail (status, type, created, expiry, outlet tags) | VA list filters and detail page | Page until empty; count locally |
| Merchant-wide transaction history, paged, filter by status | Pool account transaction table | Webhook + per-VA history into local store |
| Total count in fetch-VA response | Pagination, "22 linked VAs" | Aggregates (sum, count by status and period) |
| Aggregates (sum, count by status and period) | KPI cards, charts | Local `daily_aggregate` |
| Full list of `transactionStatus` values and response codes, including reversal | Status bar (successful, pending, reversed, failed) | Map known codes; treat unknown as `PENDING` and alert |
| Amount and transaction date in the webhook payload | Record the payment without a second call | Call #8 after each webhook to get amount |
| Webhook authentication (signature, IP list, mTLS) and retry policy | Reject forged payment notifications | IP allowlist + mandatory requery before marking `SUCCESSFUL` |
| Domiciliary (USD, GBP, EUR) account link and verification | Linked accounts settings | Portal-only maker-checker; ops verifies offline |
| KYC verification per country (BVN, CAC, TIN, Ghana Card, etc.) | Automated onboarding checks | Document upload + manual review |
| Affiliate codes and base URLs for Togo, Ghana, Benin | Multi-country | Config table, populated when Ecobank confirms |
| Order cancel, list orders | Expire or void a dynamic VA | Track expiry locally from `validityPeriod` |
| Rate limits, SLAs, requestId idempotency rules, timeouts | Retry and circuit-breaker settings | Conservative defaults — see [observability](../engineering/observability-resilience.md#resilience) |
| UAT access to history and fetch VA (currently private IPs) | Testing reconciliation | VPN or public UAT host |
| Core health / status endpoint | Readiness checks, status page | Synthetic token call |
| One platform credential or one per merchant? | Token caching and key management | Assume one platform credential per affiliate |

## Priority

The gaps that most constrain the MVP build order (see
[delivery plan](../process/delivery-plan.md)) are, in rough order of blocking risk:

1. **Full `transactionStatus`/response-code list** — the status mapper
   (see [data model](../architecture/data-model.md#va_transaction)) is built on
   incomplete information until this lands; shipping without it means an unknown
   status defaults to `PENDING` and alerts, which is safe but noisy.
2. **Webhook authentication** — until Ecobank supplies a signature or IP list, the
   portal relies on a secret path segment per affiliate plus mandatory requery before
   marking anything `SUCCESSFUL`. This is a real security gap, not just an
   inconvenience — see [security](../engineering/security.md#webhook).
3. **UAT access to fetch-VA and transaction history** (currently private IPs) —
   blocks integration testing for VA sync and reconciliation regardless of code
   readiness.

Pool account balance is no longer on this list: it's decided (not a core-API
question) that the dashboard won't show a balance regardless of what core exposes —
see [decisions](../process/open-questions.md#product-decisions). The core gap itself
(no balance endpoint) still stands for `virtual-account/api/v2/merchant/settlement-account`
if it's ever needed for internal reconciliation, but it no longer blocks the pool
account card.
