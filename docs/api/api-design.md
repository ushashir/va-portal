# Portal API design

One REST API, versioned under `/api/v1`, documented with OpenAPI 3 (springdoc), JSON
only, one response envelope.

**Merchant endpoints never take `merchantId` in the path** — the tenant comes from the
JWT, which removes a whole class of cross-tenant bugs. **Admin endpoints take
`merchantId` explicitly**, since a bank user can act on any merchant.

## Conventions

- Resource nouns, plural, kebab-case: `/virtual-accounts`, `/linked-accounts`.
- Actions that are not CRUD use a sub-resource verb:
  `POST /virtual-accounts/{vaNumber}/block`.
- Pagination: `?page=0&size=20&sort=transactionAt,desc`; max size 100.
- Filters as query params; date ranges as `from` and `to` (ISO 8601) or
  `period=TODAY|LAST_7_DAYS|LAST_30_DAYS|LAST_90_DAYS|CUSTOM`.
- Every request may send `Idempotency-Key`; repeats within 24 hours return the first
  result. The same id is sent to core as `X-Request-Id` / `headerInfo.requestId`; the
  server generates one if absent and returns it.
- Money in responses is always a string, never a float:
  `{ "amount": "8745550.00", "currency": "NGN" }`.

## Envelope

```json
{
  "success": true,
  "code": "OK",
  "message": "Virtual accounts retrieved",
  "data": {},
  "meta": {
    "page": 0,
    "size": 20,
    "requestId": "7c1f...",
    "timestamp": "2026-10-12T20:39:00Z",
    "totalElements": 22,
    "totalPages": 2
  }
}
```

Errors use the same envelope with `success: false`, a stable `code`
(`VALIDATION_FAILED`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`,
`CORE_UNAVAILABLE`, `CORE_REJECTED`, `RATE_LIMITED`) and
`errors: [{ field, message }]` for validation failures.

Both the envelope and error mapping are built once — see
[coding standards](../engineering/coding-standards.md#dry-rules-non-negotiable).

---

## Auth and account

| Method | Path | Who | Purpose |
|---|---|---|---|
| POST | `/auth/login` | Public | Email + password; returns MFA challenge or tokens |
| POST | `/auth/mfa/verify` | Public (challenge token) | TOTP or OTP code; returns access token, sets refresh cookie |
| POST | `/auth/refresh` | Refresh cookie | Rotate refresh token, new access token |
| POST | `/auth/logout` | Any user | Revoke current session |
| POST | `/auth/password/forgot` | Public | Send reset link (same response whether email exists or not) |
| POST | `/auth/password/reset` | Reset token | Set new password |
| GET, PATCH | `/me` | Any user | Profile |
| PUT | `/me/password` | Any user | Change password |
| POST | `/me/mfa/setup`, `/me/mfa/enable`, `/me/mfa/disable` | Any user | 2FA |
| GET | `/me/sessions` | Any user | Active sessions |
| DELETE | `/me/sessions/{sessionId}` | Any user | Sign out a device |
| GET, PUT | `/me/notification-preferences` | Any user | Email, SMS, in-app per type |
| POST | `/me/switch-context` | Merchant admin | Switch merchant view and outlet view |

## Admin: merchant onboarding and management

| Method | Path | Role | Purpose |
|---|---|---|---|
| POST | `/admin/merchants/invitations` | Bank admin | Create merchant shell + send invite link (`email`, `businessName`, `affiliateCode`) |
| POST | `/admin/merchants/invitations/{id}/resend` | Bank admin | Resend or extend invite |
| DELETE | `/admin/merchants/invitations/{id}` | Bank admin | Revoke invite |
| GET | `/admin/merchants` | Bank users | List; filter by status, `affiliateCode`, `q`, created date |
| GET | `/admin/merchants/{merchantId}` | Bank users | Profile, KYC, users, VA count, approval history |
| GET | `/admin/merchants/{merchantId}/kyc-documents/{docId}` | Bank users | Short-lived signed download URL |
| POST | `/admin/merchants/{merchantId}/kyc-documents/{docId}/review` | Bank admin | Accept or reject a document with note |
| POST | `/admin/merchants/{merchantId}/approve` | Bank approver | Checker approves submission |
| POST | `/admin/merchants/{merchantId}/reject` | Bank approver | Reject with reason; merchant can resubmit |
| PUT | `/admin/merchants/{merchantId}/provisioning` | Bank admin | Record `serviceProviderId`, settlement account, notification URL after Ecobank creates the merchant in core |
| POST | `/admin/merchants/{merchantId}/activate` | Bank approver | Go live (requires provisioning) |
| POST | `/admin/merchants/{merchantId}/suspend`, `/reactivate`, `/deactivate` | Bank approver | Lifecycle; deactivate also blocks all VAs in core |
| GET | `/admin/approvals` | Bank approver | Pending maker-checker queue |
| POST | `/admin/approvals/{id}/approve`, `/reject` | Bank approver | Decide any queued request |
| GET, POST, PATCH | `/admin/users` | Bank super admin | Bank staff and roles |
| GET | `/admin/dashboard/summary` | Bank users | Merchants by status, total value and volume across merchants, per country |
| GET | `/admin/merchants/{merchantId}/dashboard/*` | Bank users | Same dashboard endpoints as the merchant, scoped to one merchant |
| GET | `/admin/activity-logs` | Bank users | Platform audit trail |
| GET | `/admin/integration-logs` | Bank super admin | Core call log (masked) |
| GET, PUT | `/admin/countries/{affiliateCode}` | Bank super admin | Country config |

## Merchant onboarding (self-serve after invite)

| Method | Path | Purpose |
|---|---|---|
| GET | `/onboarding/invitations/{token}` | Validate link; return business name and email |
| POST | `/onboarding/signup` | Create the first merchant admin (`token`, `name`, `phone`, `password`); sends email OTP |
| POST | `/onboarding/verify-email` | Confirm OTP |
| GET | `/onboarding` | Current step and what is missing |
| PUT | `/onboarding/business-profile` | Legal name, registration no, tax id, address, contact |
| PUT | `/onboarding/directors` | Directors and signatories |
| POST | `/onboarding/kyc-documents` | Multipart upload; type validated against the country's `kyc_schema` |
| DELETE | `/onboarding/kyc-documents/{docId}` | Remove before submit |
| PUT | `/onboarding/settlement-account` | Pool account number (name check once core exposes it) |
| POST | `/onboarding/submit` | Lock and send for review |

Team invites (`/merchant/team`) are available as soon as email is verified, so the
owner can invite staff while waiting for approval.

## Merchant: team and access

| Method | Path | Purpose |
|---|---|---|
| GET | `/merchant/team/members` | List members, role, outlets, last login |
| POST | `/merchant/team/invitations` | Invite as `MERCHANT_ADMIN` or `OUTLET_ADMIN` with `outletIds` |
| POST | `/merchant/team/invitations/{id}/resend` | Resend |
| PATCH | `/merchant/team/members/{userId}` | Change role or outlets |
| POST | `/merchant/team/members/{userId}/disable` | Remove access (keeps history) |

## Merchant: pool account, virtual accounts, orders

| Method | Path | Core call | Purpose |
|---|---|---|---|
| GET | `/merchant/pool-account` | None (stored) | Account details, linked VA count, totals for period |
| GET | `/merchant/pool-account/transactions` | None (stored) | Same as `/merchant/transactions`, all VAs |
| GET | `/merchant/virtual-accounts` | None (stored) | Filters: status, type, `outletId`, `createdFrom`, `createdTo`, `q` |
| GET | `/merchant/virtual-accounts/summary` | None | Counts: total, active, static, dynamic |
| GET | `/merchant/virtual-accounts/export?format=xlsx` | None | Excel export of filtered list |
| GET | `/merchant/virtual-accounts/{vaNumber}` | None; `?refresh=true` calls #8 for latest txn | Detail: total received, count, type, outlet, status, expiry |
| POST | `/merchant/virtual-accounts` | #3 (dynamic) or #6 (static) | Generate VA; merchant admin only |
| PATCH | `/merchant/virtual-accounts/{vaNumber}` | #7 | Rename (rejected with `CONFLICT` if it has transactions) |
| POST | `/merchant/virtual-accounts/{vaNumber}/make-static` | #6 | Assign static details, display name |
| POST | `/merchant/virtual-accounts/{vaNumber}/block`, `/unblock` | #5 | Block or unblock |
| PUT | `/merchant/virtual-accounts/{vaNumber}/outlet` | None | Reassign outlet; past transactions keep old outlet |
| GET | `/merchant/virtual-accounts/{vaNumber}/transactions` | None | Paged history |
| POST | `/merchant/orders` | #3 | Create a collection order (`amount`, `amountType` A/E/L/G, `validityMinutes`) |
| GET | `/merchant/orders/{orderId}` | #4 if not final | Order and payment status |

(`#3`–`#8` refer to the core endpoints in
[core API mapping](../integration/core-api-mapping.md).)

## Merchant: transactions, outlets, linked accounts, notifications

| Method | Path | Purpose |
|---|---|---|
| GET | `/merchant/transactions` | Filters: status, `outletId`, `vaNumber`, `minAmount`, `maxAmount`, `from`, `to`, `q` (reference, account, payer) |
| GET | `/merchant/transactions/status-summary` | Count, value, and share per status for the period (status bar) |
| GET | `/merchant/transactions/{id}` | Detail |
| POST | `/merchant/transactions/{id}/requery` | Calls #8 and updates the mirror |
| GET | `/merchant/transactions/export?format=csv` | Streams CSV; max range 12 months (proposed) |
| GET, POST | `/merchant/outlets` | List with accounts, transfers, and value per outlet; create |
| GET, PATCH, DELETE | `/merchant/outlets/{outletId}` | Detail, edit, archive (only if no active VAs) |
| GET | `/merchant/outlets/unassigned-accounts` | VAs without an outlet (alert) |
| GET, POST | `/merchant/linked-accounts` | List, add USD/GBP/EUR domiciliary account (goes to approval) |
| GET | `/notifications` | `?unread=true` |
| POST | `/notifications/{id}/read`, `/notifications/read-all` | Mark read |
| GET | `/merchant/activity-logs` | Merchant audit trail (merchant admin only) |

## Dashboard analytics

All read from `daily_aggregate` and `va_transaction` — **none call core.** All accept
`period` or `from`/`to`; outlet admins are automatically scoped to their outlets.

| Method | Path | Returns |
|---|---|---|
| GET | `/merchant/dashboard/summary` | Total received, successful transaction count, VA count (active / inactive-or-expired), average transfer — each with % change vs. previous equal period |
| GET | `/merchant/dashboard/timeseries?granularity=DAY\|WEEK\|MONTH` | Value and volume per bucket for the chart |
| GET | `/merchant/dashboard/top-virtual-accounts?limit=6` | VA, outlet, value received |
| GET | `/merchant/dashboard/recent-transactions?limit=6` | Latest transactions |
| GET | `/merchant/dashboard/outlets` | Value and count per outlet |

Summary responses are **cached per merchant + period for 60 seconds** (Caffeine, or
Redis when running more than one instance) and evicted when a new transaction is
written.

## Inbound webhook and health

| Method | Path | Purpose |
|---|---|---|
| POST | `/webhooks/ecobank/{affiliateCode}/va-notification` | Receives core notification; stores in `webhook_event`, returns `{"responseCode":"000","responseMessage":"SUCCESS"}` within 2s, processes async |
| GET | `/actuator/health/liveness` | Process up (public to load balancer) |
| GET | `/actuator/health/readiness` | DB, Redis, core token reachable |
| GET | `/actuator/health` | Detailed components (internal network only) |
| GET | `/actuator/info` | Build version, git commit |
| GET | `/actuator/prometheus` | Metrics (internal network only) |

See [observability](../engineering/observability-resilience.md#health) for what each
health indicator actually checks, and
[security](../engineering/security.md#webhook) for how the webhook endpoint defends
itself before Ecobank supplies signing.
