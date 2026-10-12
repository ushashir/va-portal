# Security

Spring Security 6, stateless JWT access tokens, rotating refresh tokens, mandatory
2FA, RBAC with tenant and outlet scoping, maker-checker for every bank decision, and
AES-GCM for core payloads. The portal holds bank customer data across several
countries, so it is built to the bank's security bar, not a startup's.

## Authentication

- **Passwords**: Argon2id (Spring `Argon2PasswordEncoder`), min 10 characters, checked
  against a breached-password list, no reuse of last 5.
- **Access token**: JWT, RS256, 15 minutes, claims `sub`, `userType`, `merchantId`,
  `roles`, `outletIds`, `sessionId`. Signing keys rotated via JWKS with `kid`.
- **Refresh token**: opaque, 7 days, stored hashed in `user_session`, sent as
  `HttpOnly; Secure; SameSite=Strict` cookie, rotated on every use. **Reuse of an old
  token revokes the whole session family.**
- **2FA**: TOTP (RFC 6238) mandatory for bank users and merchant admins; optional with
  email/SMS OTP fallback for outlet admins.
- **Lockout**: 5 failed attempts locks for 15 minutes; login and OTP endpoints rate
  limited per IP and per account (Bucket4j).
- **Invite and reset links**: single-use, random 32 bytes, stored as SHA-256 hash,
  72-hour (invite) and 30-minute (reset) expiry.

## Authorization

- **Roles**: `BANK_SUPER_ADMIN`, `BANK_ADMIN`, `BANK_APPROVER`, `MERCHANT_ADMIN`,
  `OUTLET_ADMIN`. Permissions are fine-grained (`VA_BLOCK`, `TEAM_INVITE`,
  `MERCHANT_APPROVE`), enforced with `@PreAuthorize`.
- **Tenant isolation**: merchant endpoints read `merchantId` only from the token. A
  Hibernate `@Filter` on `merchantId` is enabled per request as a second guard.
- **Outlet scoping**: outlet admins get a JPA `Specification` that restricts to
  `outletIds` from the token, applied in **one** shared query builder (not per
  endpoint) — see [coding standards](coding-standards.md#dry-rules-non-negotiable).
- **Maker-checker**: approve, activate, deactivate, linked accounts, and VA requests
  need two different bank users. The approver cannot be the maker (enforced in service
  and by DB check). Backed by the `approval_request` table — see
  [data model](../architecture/data-model.md#approval_request).
- **Merchant deactivation** blocks all VAs in core and revokes all merchant sessions.

## Core integration

- Core credentials, client secret, and the AES key per affiliate live in a secrets
  manager (HashiCorp Vault or the cloud equivalent), loaded at start-up, **never** in
  `application.yml` or Git.
- **Token**: fetched with the password grant, cached in memory/Redis, refreshed 60s
  before `expires_in`, single-flight lock so 50 concurrent requests trigger one
  refresh.
- **Payload encryption**: one `AesGcmCipher` component (AES/GCM/NoPadding, 12-byte
  random IV prefixed, 128-bit tag), as in the Ecobank Java sample. Which endpoints
  need it is configured per endpoint until Ecobank confirms — see
  [core API mapping](../integration/core-api-mapping.md#payload-inconsistencies-the-core-client-must-normalize).
- Outbound TLS 1.2+ with hostname verification; **no trust-all clients in any
  profile.**

## Webhook

- Accept only from Ecobank IP ranges (to be supplied) at the gateway.
- Ask Ecobank for an HMAC signature header; **until then**, a secret path segment per
  affiliate plus mandatory requery (core endpoint #8) before any transaction is marked
  `SUCCESSFUL`.
- Idempotent on `postingReferenceNo`; replay of the same payload is a no-op (see
  `webhook_event` in [data model](../architecture/data-model.md#webhook_event)).

This is a provisional defense pending answers in
[open questions](../process/open-questions.md) about signature, IP list, and retry
policy — treat it as a gap to close, not a finished design.

## Data protection

- Encrypt sensitive columns at rest (payer account number, MFA secret, director ID
  numbers) with a JPA `AttributeConverter` using AES-GCM and a key from the secrets
  manager. PostgreSQL storage encryption on top.
- Mask PII in logs and API responses where not needed: account numbers show last 4,
  emails partially.
- KYC files in private object storage, served only via 5-minute signed URLs,
  virus-scanned on upload, type and size checked (PDF, JPG, PNG; max 10 MB).
- Data protection laws apply per country (for example Nigeria's Data Protection Act
  2023). Retention periods follow the bank's policy —
  [open question 5](../process/open-questions.md).

## Web and API hardening

- CORS restricted to the portal origins per environment.
- Headers: HSTS, Content-Security-Policy, X-Content-Type-Options, Referrer-Policy,
  `frame-ancestors 'none'`.
- Bean Validation on every request DTO; no entity binding from request bodies.
- CSV/Excel export escapes cells starting with `=`, `+`, `-`, `@` (formula injection).
- OWASP Dependency-Check and container image scan in CI; build fails on high or
  critical CVEs.
- Annual penetration test before go-live in each country (bank requirement to
  confirm).
