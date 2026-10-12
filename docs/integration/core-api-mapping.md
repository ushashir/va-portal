# Ecobank core API mapping

The core APIs cover VA operations and per-VA transaction lookups. They do **not**
cover merchant onboarding, pool account data, analytics, VA metadata, or webhook
security. These gaps drive the [persistence decision](../architecture/persistence-decision.md)
and are listed in full in [core gaps](core-gaps.md).

**Sources:** VAS Ecobank Virtual Account Integration Specification (test/UAT, v1) and
VAS Production Integration (prod, v1).
**Base hosts:** UAT `https://eidev.ecobank.com:7505`, Prod
`https://virtualacctservice.ecobank.com`.

## Endpoint map

| # | Core capability | Method and path | UAT | Prod | Used by portal for |
|---|---|---|---|---|---|
| 1 | Get access token (password grant) | `POST /auth-service/oauth/token` | `artxuat.ecobank.com`, `expires_in` 11999s | `digitalservice.ecobank.com`, `expires_in` 1199s | Every core call |
| 2 | Create merchant | Not exposed; payload sample only | "Done by Ecobank" | "Done by Ecobank" | Onboarding (manual step) |
| 3 | Create order (dynamic VA) | `POST /virtual-account/api/v2/create-order` | Yes | Yes | Generate dynamic VA / collection order |
| 4 | Transaction status by order | `POST /virtual-transaction-status/api/v2/trans-status` | Yes | Yes | Order status, requery |
| 5 | Block / unblock VA | `POST /virtual-account-modify/api/v2/block-unblock-account` | Yes | Yes | VA block, unblock |
| 6 | Assign static VA details | `POST /virtual-account-static/api/v2/assign-details` | Yes | Yes | Convert VA to static, set display name |
| 7 | Update VA name | `POST /virtual-account/api/v2/update/virtual-account-name` | Path blank in doc | Yes | Rename VA (only if no transactions yet) |
| 8 | VA transaction status by reference | `POST /virtual-account/api/v2/get/virtual-account-transaction-status` | Path partial | Yes | Requery pending transaction, detail refresh |
| 9 | Fetch VAs (paged) | `POST /virtual-account/api/v2/fetch/virtual-account` | Internal IP `192.168.31.89:7005` | Yes | VA sync |
| 10 | Fetch all VAs (paged) | `POST /virtual-account/api/v2/fetch/all-virtual-account` | Not documented | Yes | VA sync (difference from #9 unclear — see [core gaps](core-gaps.md)) |
| 11 | Transaction history per VA | `POST`, encrypted `text/plain` body | Internal IP `192.168.31.89:7003/.../trans-status/transaction-history` | Path given only as `/virtual-account/api/v2` | Nightly reconciliation, backfill |
| 12 | Payment notification (inbound webhook) | `POST` to merchant `serviceProviderNotificationUrl` | Not documented | Yes | Real-time transaction ingest |

Both UAT and prod environments currently expose #9, #10, and #11 only via internal IPs
or partial paths — see the UAT-hosts question in
[open questions for Ecobank](../process/open-questions.md#questions-for-ecobank).

**Confirmed: no GET merchant(s) endpoint.** Checked against both source specs
(test/UAT v1 and prod v1) on 2026-10-12 — there is no endpoint to fetch a merchant /
service provider record. #2 ("Create merchant") exists only as a payload sample and
is explicitly "done by Ecobank" manually; there is no corresponding get/fetch call.
Tracked as the missing **M-2** capability in
[core API requirements — MVP](core-api-requirements-mvp.md) and as a core gap in
[core gaps](core-gaps.md).

## Payload inconsistencies the core client must normalize

These are quirks observed directly in the Ecobank sample payloads. The client hides
all of them behind one internal model — see `EcobankClient` in
[coding standards](../engineering/coding-standards.md) — so no other layer in the
portal ever sees a core quirk.

- **Header object name differs by endpoint**: `headerInfo` on most calls,
  `hostHeaderInfo` on assign-static (#6), and absent entirely on get-VA-transaction-status
  (#8) and fetch-VAs (#9, #10).
- **Create order response key is spelled `oder`, not `order`.**
- **Response wrappers differ**: `details`, `data` (object), `data` (array),
  `modelResponse`, or fields at the top level — depending on endpoint.
- **Date formats differ** across responses: `Sep 21, 2023 12:56:16 PM`,
  `2023-08-10T08:54:21.000+0000`, `2025-09-17 12:08:47.0`.
- **Amounts arrive as numbers in some responses and strings (`"5000"`) in others.**
- **`orderId` is `0` (number) in one history sample and a string elsewhere.**
- **The block-VA response sample returns a different `serviceProviderId` and VA number
  from the request** — needs verification before the portal trusts the response body
  over its own request state.
- **Encryption is inconsistent with the spec text**: the spec states all payloads are
  AES-GCM encrypted, but every sample *except* transaction history (#11) is plain
  JSON. See the encryption-scope question in
  [open questions for Ecobank](../process/open-questions.md#questions-for-ecobank) —
  which endpoints actually require encryption is not yet confirmed; it's configured per-endpoint in the client
  until Ecobank confirms.

## Token handling

- Fetched via the password grant (#1), cached in memory/Redis, refreshed 60 seconds
  before `expires_in`.
- Single-flight lock so 50 concurrent requests trigger exactly one refresh, not 50.
- See [security](../engineering/security.md#core-integration) for where credentials
  and the AES key are stored.
