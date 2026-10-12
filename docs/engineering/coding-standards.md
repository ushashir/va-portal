# Coding standards and DRY rules

Java 21, Spring Boot 3.x, Maven, a **modular monolith packaged by feature**. DRY is
enforced by design: one core client, one envelope, one base entity, one audit aspect,
one query builder. Code review rejects copy-paste of any of these.

## Stack

- Java 21 (records, sealed types, virtual threads for the core client), Spring Boot
  3.x, Spring Web, Spring Security, Spring Data JPA (Hibernate 6), Bean Validation.
- PostgreSQL 16 (pgAdmin is the team's admin client, not part of the runtime). Flyway
  for migrations.
- MapStruct for DTO mapping, Lombok limited to `@Getter`, `@Builder`,
  `@RequiredArgsConstructor` (no `@Data` on entities).
- Resilience4j, Bucket4j, ShedLock, Caffeine (Redis when scaled out),
  springdoc-openapi, Apache POI (Excel), OpenCSV.
- Tests: JUnit 5, AssertJ, Mockito, Testcontainers (PostgreSQL), WireMock (core),
  Spring REST Docs or OpenAPI contract tests.

## Package layout

```
com.ecobank.vaportal
  common/          envelope, exceptions, BaseEntity, paging, money, audit, security utils
  config/          Spring config, @ConfigurationProperties
  integration/
    ecobank/       EcobankClient, token, cipher, request/response models, mappers
  auth/            login, MFA, sessions, JWT
  admin/           bank users, approvals, country config
  merchant/        merchant profile, onboarding, team
  outlet/
  virtualaccount/
  order/
  transaction/     ingest (webhook), requery, reconciliation
  dashboard/       aggregates and analytics queries
  notification/
  activitylog/
```

Each feature package holds `controller`, `service`, `repository`, `domain`
(entities), `dto`, `mapper`. **Features talk through services, never through another
feature's repository.**

## DRY rules (non-negotiable)

1. **One core client.** `EcobankClient.execute(EcobankEndpoint endpoint, Object body,
   Class<T> type)` handles base URL per affiliate, token, `headerInfo`/
   `hostHeaderInfo` construction, encryption, timeouts, retries, circuit breaker,
   integration logging, and response-code mapping. Endpoint-specific classes only
   declare path, header style, encryption flag, and the response type. See
   [core API mapping](../integration/core-api-mapping.md) for everything this client
   has to normalize.
2. **One response envelope** `ApiResponse<T>` built by a `ResponseFactory`;
   controllers never assemble maps. See the envelope in
   [API design](../api/api-design.md#envelope).
3. **One exception model.** Services throw `BusinessException(ErrorCode)`; one
   `@RestControllerAdvice` maps every exception to the envelope. No
   try/catch-and-return-error in controllers.
4. **One base entity** with auditing fields and optimistic locking (see
   [data model](../architecture/data-model.md#conventions)).
5. **One audit mechanism**: `@Auditable` + aspect (see
   [observability](observability-resilience.md#activity-log)).
6. **One filter builder.** Dynamic list filters use a shared `SpecificationBuilder`
   with tenant and outlet scoping built in. No hand-written JPQL per filter
   combination.
7. **One paging model** `PageResponse<T>` and one `PageRequestFactory` that enforces
   max size and allowed sort fields.
8. **One money type** `Money(BigDecimal amount, Currency currency)` with a JPA
   embeddable; no raw `double`.
9. **One status mapper** from core codes to portal statuses (see
   [data model](../architecture/data-model.md#va_transaction)).
10. **Config, not constants.** URLs, timeouts, limits, and per-country values come
    from `@ConfigurationProperties` or `country_config`.
11. **Shared test fixtures**: one `TestDataFactory` and one WireMock stub set per
    core endpoint, reused by all tests.

## Code style

- Google Java Style, enforced by Spotless (format) and Checkstyle; SonarQube quality
  gate: no new bugs, no new vulnerabilities, duplication under 3%, coverage on new
  code at least 80%.
- Constructor injection only; no field `@Autowired`.
- Controllers are thin: validate, call one service method, return the envelope.
- `@Transactional` on service methods, never on controllers; **no core HTTP call
  inside a DB transaction** (call core, then open a short transaction to persist).
- DTOs are Java records; entities never leave the service layer.
- Naming: classes PascalCase, methods and fields camelCase, constants UPPER_SNAKE,
  tables and columns snake_case, REST paths kebab-case, JSON camelCase.
- Methods under 30 lines, classes under 300 lines as a guideline; one public
  responsibility per service.
- Every public API has OpenAPI annotations with examples; every Flyway migration is
  forward-only and reviewed.
- No `System.out`, no swallowed exceptions, no TODO merged without a ticket
  reference.

## Definition of done

- Unit and integration tests pass, coverage gate met
- OpenAPI updated and reviewed by frontend
- Activity log entries added for new state changes
- No secrets or PII in code, logs, or test data
- Flyway migration tested against a copy of staging data
- Reviewed and approved by at least one other engineer
