# Core API requirements — MVP only

What the core API team needs to deliver so the MVP dashboard can ship. Everything
here is **P1 — blocks MVP**. P2 items (needed before go-live, not before MVP) are
listed at the bottom with no detail; full detail on both is in
[core gaps](core-gaps.md) and [core API mapping](core-api-mapping.md).

13 P1 items: 2 already work, 5 need a fix, 6 don't exist yet.

## Already works — no action needed

| # | Capability | Endpoint |
|---|---|---|
| P-1 | Access token | `POST /auth-service/oauth/token` |
| P-2 | Create order (dynamic VA) | `POST /virtual-account/api/v2/create-order` |

## Needs a fix

| # | Capability | What's broken | What to fix |
|---|---|---|---|
| E-1 | Fetch VAs for a merchant | Returns only number + name, no count, UAT on a private IP | Add `status`, `type`, `displayName`, `createdDate`, `expiryDate`, `currency`, `totalRecords`; put it on a public host |
| E-2 | Transaction history per VA | Prod path incomplete, UAT on a private IP, not paginated | Publish the real path + public UAT host; add paging |
| E-3 | Payment webhook | No amount/currency/date/status in the payload, unsigned, no retry policy | Add `amount`, `currency`, `transactionDate`, `transactionStatus`; sign with HMAC; document retries |
| E-4 | Transaction status by order | Only 2 of N status values documented, no currency | Publish every `transactionStatus` value (incl. expired/failed/reversed); add `currency` |
| E-5 | VA transaction status by reference | No `headerInfo`, status alone is ambiguous (e.g. `TRANS_COMPLETED` + code 120 = invalid amount) | Standard `headerInfo`; one unambiguous final status; amounts as numbers |

## Doesn't exist yet

| # | Capability | Why MVP needs it |
|---|---|---|
| M-1 | Create/update merchant | No self-serve onboarding without it |
| M-2 | Get merchant details | Confirm provisioning, show settlement account |
| M-3 | Generate VA on demand | The "generate virtual account" button has nothing to call |
| M-4 | Get single VA details | VA detail page / single refresh without paging the whole list |
| M-5 | Pool account details | The pool account card has nothing to show without it |
| M-6 | Merchant-wide transaction list | One call across all VAs instead of 22 calls per merchant per date range |

## The 3 questions that matter most right now

1. **M-1**: will merchant creation become an API, or does it stay manual? If manual,
   what's the turnaround and how do we get `serviceProviderId` back?
2. **M-5**: can the pool balance be exposed, and is the bank willing to show it?
3. Which of E-1 through M-6 ship first, and by what date?

(Full question list, including VA pre-allocation, retention, and UAT contacts: see
[open questions](../process/open-questions.md).)

## Cross-cutting, still needed for these 13 to be usable

- **Encryption scope** — confirm which endpoints actually need AES-GCM bodies (the
  spec says all; the samples say otherwise).
- **One header object** — `headerInfo` everywhere, not `hostHeaderInfo` or none.
- **Full status/response code list** — `responseCode`, `transactionRespCode`,
  `transactionStatus`, including reversal/expiry/duplicate.
- **Consistent formats** — ISO 8601 dates, numeric amounts with currency, one
  response wrapper shape.
- **Public UAT hosts** — fetch-VA and transaction history are only reachable on a
  private IP today; this blocks testing regardless of what else ships.

---

## Deferred to go-live (P2 — not blocking MVP)

Block/unblock VA (P-3, already works), update VA name (E-6), assign static VA
details (E-7), change merchant status (M-7), account name enquiry (M-8), transaction
summary (M-9, optional if M-6 ships), health check (M-10) — plus the cross-cutting
asks on multi-country credentials, rate limits/SLAs, and versioning notice. Full
detail in [core gaps](core-gaps.md).
