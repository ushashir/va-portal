# Data model (PostgreSQL via JPA)

## Conventions

- Every table extends a shared `BaseEntity` (`id` UUID, `created_at`, `updated_at`,
  `created_by`, `updated_by`, `version` for optimistic locking) through
  `@MappedSuperclass` and Spring Data JPA auditing.
- Money is `NUMERIC(19,4)` mapped to `BigDecimal` with an ISO 4217 currency column —
  see the single `Money` type in [coding standards](../engineering/coding-standards.md).
- Timestamps are `TIMESTAMPTZ` in UTC; the UI converts using the merchant's country
  timezone (from `country_config`).
- **No generic `deleted_at` / soft-delete column.** Rows that leave a lifecycle do so
  through an explicit status value instead (`ARCHIVED`, `REVOKED`, `DEACTIVATED`,
  etc.) — see each table below — with the transition itself recorded in
  `activity_log`. A timestamp-only soft delete would duplicate what the status
  column and the audit trail already capture, and loses the "why."
- **Schema changes go through Flyway only.**

## Tables

### `country_config`
One row per Ecobank affiliate.

| Column | Notes |
|---|---|
| `affiliate_code` (PK) | e.g. `ENG` for Nigeria |
| `country_iso`, `currency`, `timezone` | |
| `kyc_schema` (JSONB) | per-country required KYC document types |
| `core_base_url` | |
| `active` | |

Credentials are referenced **by secret name**, never stored in this table.

### `merchant`
| Column | Notes |
|---|---|
| `affiliate_code` | |
| `legal_name`, `trading_name`, `registration_no`, `tax_id` | |
| `status` | see onboarding status list below |
| `service_provider_id` | unique, nullable until Ecobank provisions the merchant in core |
| `settlement_account_no` | |
| `notification_url` | |

### `merchant_invitation`
`merchant_id`, `email`, `token_hash`, `invited_by`, `expires_at`, `accepted_at`,
`status` (`PENDING`, `ACCEPTED`, `EXPIRED`, `REVOKED`). Token stored as SHA-256 hash;
72-hour expiry (proposed). `status` is what `DELETE /admin/merchants/invitations/{id}`
(revoke) and the resend endpoint actually change — `accepted_at` alone can't
represent "revoked."

### `kyc_document`
`merchant_id`, `doc_type`, `storage_key`, `checksum`, `status`, `reviewed_by`,
`review_note`. **Files live in object storage (S3-compatible), not the DB.**

### `approval_request`
`entity_type`, `entity_id`, `action`, `maker_id`, `checker_id`, `status`, `reason`.
Generic maker-checker record, reused for merchants, linked accounts, and VA requests.

### `app_user`
| Column | Notes |
|---|---|
| `email` (unique) | one login per email |
| `phone`, `password_hash` | |
| `user_type` | `BANK` or `MERCHANT` |
| `merchant_id` | null for bank users |
| `status` | `INVITED`, `ONBOARDING`, `SUBMITTED`, `APPROVED`, `ACTIVE`, `SUSPENDED`, `DEACTIVATED`, `REJECTED` |
| `mfa_enabled`, `mfa_secret_enc` | |
| `last_login_at`, `failed_attempts`, `locked_until` | |

### `role`, `permission`, `user_role`, `role_permission`
Standard RBAC. Roles are seeded by Flyway — see [security](../engineering/security.md)
for the role list and the permissions they carry.

### `outlet_access`
`user_id`, `outlet_id` — scopes outlet admins to the outlets they're allowed to see.

### `user_session`
`user_id`, `refresh_token_hash`, `device`, `ip`, `expires_at`, `revoked_at`. Powers
"active sessions" in settings.

### `outlet`
`merchant_id`, `name`, `address`, `code`, `status` (`ACTIVE`, `ARCHIVED` — archiving
is blocked while the outlet has active VAs, per
[API design](../api/api-design.md)).

### `virtual_account`
Mirror of core VA state.

| Column | Notes |
|---|---|
| `merchant_id` | |
| `va_number` | unique per affiliate |
| `va_name`, `display_name` | |
| `type` | `STATIC`, `DYNAMIC` |
| `status` | `ACTIVE`, `BLOCKED`, `EXPIRED` |
| `outlet_id`, `expires_at` | |
| `core_synced_at` | see [persistence decision](persistence-decision.md) rule 2 |

### `va_outlet_history`
`virtual_account_id`, `outlet_id`, `from_at`, `to_at`, `changed_by`. Lets past
transactions stay attached to the outlet they happened under even after the VA moves —
confirmed (see [decisions](../process/open-questions.md#product-decisions)), via this
snapshot table.

### `order`
Dynamic VA requests. `merchant_id`, `virtual_account_id`, `core_order_id` (unique),
`amount`, `amount_type` (`A`, `E`, `L`, `G`), `validity_minutes`, `description`,
`status`, `request_id`, `core_synced_at` (same meaning as on `virtual_account` and
`va_transaction` — order status is also a core mirror, per
[persistence decision](persistence-decision.md)).

### `va_transaction`
The transaction mirror.

| Column | Notes |
|---|---|
| `merchant_id`, `virtual_account_id` | |
| `outlet_id` | snapshot at time of transaction, not a live lookup |
| `transaction_ref_no`, `core_order_id` | |
| `amount`, `amount_paid`, `currency` | |
| `status` | `SUCCESSFUL`, `PENDING`, `REVERSED`, `FAILED` (the portal's four statuses) |
| `core_status`, `core_resp_code` | the raw core values behind `status` |
| `payer_name` | |
| `payer_account_no` | **encrypted** |
| `payer_narration`, `bank` | |
| `transaction_at`, `source`, `core_synced_at` | `source` is `WEBHOOK`, `REQUERY`, or `RECON` |

Unique constraint on (`va_number`, `transaction_ref_no`) — this is what makes webhook
retries and reconciliation idempotent (upsert, never duplicate). Indexes on
(`merchant_id`, `transaction_at`), (`virtual_account_id`, `transaction_at`), and
(`merchant_id`, `status`).

**Retention: 7 years** (confirmed, same as `activity_log` — see
[decisions](../process/open-questions.md#product-decisions)).

**Status mapping:** the mapping between core transaction statuses
(`TRANS_IN_PROGRESS`, `TRANS_COMPLETED`, and others not yet documented — see
[core gaps](../integration/core-gaps.md)) and the four portal statuses lives in **one**
enum mapper (see [coding standards](../engineering/coding-standards.md)).
`TRANS_COMPLETED` alone is not sufficient to mean success: the history sample shows
`TRANS_COMPLETED` with `transactionRespCode` 94 (duplicate), and the status sample
shows `TRANS_COMPLETED` with code 120 (invalid amount). **Only `TRANS_COMPLETED` +
code `000` maps to `SUCCESSFUL`.**

### `daily_aggregate`
`merchant_id`, `outlet_id`, `virtual_account_id`, `day`, `status`, `txn_count`,
`txn_value`. Feeds all dashboard reads — see [API design](../api/api-design.md#dashboard-analytics).

### `linked_account`
`merchant_id`, `account_no`, `currency` (`NGN`, `USD`, `GBP`, `EUR`), `account_name`,
`name_match`, `status`. Domiciliary accounts.

### `notification`, `notification_preference`
`notification`: `user_id`, `type`, `title`, `body`, `read_at`.
`notification_preference`: `user_id`, `type`, `email`, `sms`, `in_app`.

### `activity_log`
Append-only. `actor_id`, `actor_type`, `merchant_id`, `action`, `entity_type`,
`entity_id`, `before` (JSONB), `after` (JSONB), `ip`, `user_agent`, `request_id`,
`occurred_at`. **No UPDATE or DELETE grants** on this table — see
[observability](../engineering/observability-resilience.md#activity-log).

### `integration_log`
Partitioned by month. `direction` (`OUT`, `IN`), `endpoint`, `request_id`,
`http_status`, `core_resp_code`, `latency_ms`, `request_masked`, `response_masked`,
`error`.

### `webhook_event`
Idempotent inbox. `request_id`, `posting_reference_no` (unique), `payload`, `status`,
`attempts`, `processed_at`.
