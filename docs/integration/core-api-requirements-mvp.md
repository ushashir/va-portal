# Core API request — MVP

What VA Portal needs from the core API to ship the MVP dashboard, what's already
provided, and what we'd ask for on top of that. Everything below is needed for MVP;
P2 items (needed before go-live, not before MVP) are listed at the bottom with no
detail.

Full background: [core gaps](core-gaps.md) and [core API mapping](core-api-mapping.md).

| # | VA Portal needs | Provided? | What's provided | Request |
|---|---|---|---|---|
| 1 | Authenticate every core call | ✅ Yes | `POST /auth-service/oauth/token` | — |
| 2 | Generate a dynamic VA / collection order | ✅ Yes | `POST /virtual-account/api/v2/create-order` | — |
| 3 | List one merchant's VAs with status, type, outlet, expiry, and a total count | ❓ Unconfirmed | `POST /virtual-account/api/v2/fetch/virtual-account` — documented as returning only VA number and name, no count. We can't confirm more: both UAT (private IP) and prod require a valid bearer token before any request body is validated, so we can't see what filters it actually accepts without real credentials | Share the request schema — specifically, does it take `serviceProviderId` to scope results to one merchant? Add `status`, `type`, `displayName`, `createdDate`, `expiryDate`, `currency`, `totalRecords` to the response; expose UAT on a public host |
| 4 | Paginated transaction history per VA | ⚠️ Partial | Endpoint exists; prod path incomplete, UAT on a private IP, no paging | Publish the full prod path and a public UAT host; add paging |
| 5 | Real-time payment notification with amount, currency, date, and status | ⚠️ Partial | Webhook fires on payment but carries none of those fields | Add `amount`, `currency`, `transactionDate`, `transactionStatus`; sign the call; document the retry schedule |
| 6 | Full set of transaction status values, with currency | ⚠️ Partial | `POST /virtual-transaction-status/api/v2/trans-status` — only `TRANS_IN_PROGRESS` / `TRANS_COMPLETED` documented, no currency | Publish every `transactionStatus` value (expired, failed, reversed, etc.); add `currency` |
| 7 | One unambiguous final status per transaction reference | ⚠️ Partial | `POST /virtual-account/api/v2/get/virtual-account-transaction-status` — status + code need interpretation together (e.g. `TRANS_COMPLETED` + code 120 = invalid amount) | A single final-outcome field (e.g. `SUCCESSFUL`/`FAILED`/`REVERSED`); standard `headerInfo`; amounts as numbers |
| 8 | Create / update a merchant (service provider) | ❌ No | — | New API — or confirm this stays manual, with expected turnaround and how we receive `serviceProviderId` |
| 9 | Get a merchant's details | ❌ No | — | New API — confirm provisioning, show settlement account |
| 10 | Generate a VA on demand | ❌ No | — | New API for the "generate virtual account" button |
| 11 | Get a single VA's details | ❌ No | — | New API — refresh one VA without paging the whole list |
| 12 | Pool account details | ❌ No | — | New API — the pool account card needs something to read |
| 13 | Merchant-wide transaction list (all VAs, one call) | ❌ No | — | New API — one call per merchant per date range, instead of one call per VA |

2 fully provided, 1 whose request-side behavior we can't confirm without real
credentials, 4 provided but incomplete, 6 not provided at all.

## A couple of observations (not requests)

Noticed while building against what's already there — not asking for anything to
change, just flagging in case it's useful:

- The spec describes all payloads as AES-GCM encrypted; most samples we've worked
  from are plain JSON. We're building the client to handle either — a confirmation
  of which endpoints expect encrypted bodies would save us some guesswork.
- Row 3's endpoint has a sibling, "fetch all VA," that looks similar — we're not
  sure what distinguishes the two, or whether either already returns the fields
  we're asking for above.

## What would help us plan

1. Row 8 (merchant API) — will it be exposed, or does onboarding stay manual?
2. Which rows above you'd take on first, and roughly when.

(Fuller list, including multi-country credentials, rate limits, and retention:
[open questions](../process/open-questions.md).)

---

## For later, not needed for MVP

Update VA name, assign static VA details, change merchant status, account name
enquiry, a transaction summary endpoint (optional if row 13 covers it), and a
health-check endpoint. Full detail in [core gaps](core-gaps.md).
