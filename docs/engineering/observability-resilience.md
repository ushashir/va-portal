# Logging, activity log, observability, and resilience

Three separate records, each with one job: application logs for engineers, the
activity log for auditors and merchants, and the integration log for disputes with
core. **One `requestId` ties all three together.**

| Record | Where | Contents | Retention | Who reads it |
|---|---|---|---|---|
| Application log | stdout as JSON (Logback + logstash encoder) to ELK / Loki (proposed) | Level, logger, message, `requestId`, `userId`, `merchantId`, latency; PII masked | 30 days hot, 1 year archive | Engineers, support |
| Activity log | `activity_log` table, append-only | Who did what to which entity, before/after, IP, user agent | 7 years or bank policy | Merchant admins (own), bank users (all) |
| Integration log | `integration_log` table, partitioned monthly | Every core call and webhook: endpoint, `requestId`, status, core code, latency, masked bodies | 90 days, then archive | Bank super admin, support |

## Activity log

- Written by an `@Auditable(action = "VA_BLOCK", entity = "VIRTUAL_ACCOUNT")`
  annotation and **one** Spring AOP aspect, so no service writes audit rows by hand —
  see [coding standards](coding-standards.md#dry-rules-non-negotiable).
- Covers: logins (success and failure), MFA changes, password changes, invites, role
  and outlet changes, VA create/block/unblock/rename/reassign, order create, exports,
  onboarding steps, every approval decision, country config changes.
- The DB role used by the app has **INSERT and SELECT only** on this table.
- API: `GET /merchant/activity-logs` and `GET /admin/activity-logs` with filters
  `actorId`, `action`, `entityType`, `from`, `to` — see
  [API design](../api/api-design.md).

## Logging rules

- INFO for business events, WARN for handled core rejections, ERROR only for things
  someone must act on.
- **Never log** passwords, tokens, OTPs, AES keys, full account numbers, or raw
  encrypted payloads. A Logback masking converter enforces this as a backstop.
- `requestId` is in MDC for every log line, returned as `X-Request-Id`, and sent to
  core as `headerInfo.requestId`.

## Metrics and tracing

Micrometer to Prometheus; Grafana dashboards.

Key metrics: core call latency and error rate per endpoint, token refresh count,
webhook received vs. processed, reconciliation drift count, pending transactions
older than 10 minutes, HTTP p95 per endpoint, DB pool usage.

OpenTelemetry tracing across the portal and the core client.

**Alerts**: core error rate above 5% for 5 minutes, webhook backlog above 100,
reconciliation drift above threshold, any 5xx spike, certificate expiry within 21
days.

## Resilience

- Resilience4j on the core client: timeout 10s (connect 3s), 2 retries with backoff
  only for idempotent reads and network errors, circuit breaker per endpoint,
  bulkhead to stop core slowness exhausting threads.
- **When core is down**, dashboards still work from stored data and show "Last synced
  at"; write actions return `CORE_UNAVAILABLE`.
- Scheduled jobs (ShedLock so only one instance runs them): pending requery every 5
  minutes, VA sync nightly, reconciliation nightly, invitation expiry hourly,
  aggregate rebuild on demand.

## Health

- **Liveness**: app process only.
- **Readiness**: PostgreSQL, Redis (if used), secrets loaded, and a custom
  `EcobankCoreHealthIndicator` that reports UP if a token was obtained in the last 5
  minutes (it does **not** call core on every probe).
- `/actuator/health` details, `prometheus`, and `loggers` are exposed only on the
  management port, not the public ingress.
- `/actuator/info` returns version, commit, and build time so support can confirm
  what is deployed.
