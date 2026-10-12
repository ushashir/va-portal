# Ecobank Merchant Virtual Account Portal

A Spring Boot backend and web dashboard that lets Ecobank merchants see their pool
account, the virtual accounts (VAs) attached to it, and every transfer received into
those VAs. It sits between the Ecobank Virtual Account System (VAS) core APIs and a
separate dashboard SPA. **It does not move money** — no payouts, refunds or transfers
out.

This repository currently holds **specification and planning documents only**. No
application code has been written yet; the documents below are the product and
engineering spec that implementation will follow.

> Source: product/engineering spec dated 2026-10-12 (Ushahemba Shir). This doc set
> reorganizes that spec into living, per-topic documents so each concern can be updated
> independently as decisions firm up (especially the [open questions](docs/process/open-questions.md),
> most of which need an answer from Ecobank before build starts).

## Documentation map

### Product
- [Product overview](docs/product/overview.md) — who uses the portal, MVP scope, what's
  explicitly out of scope, and the multi-country model.

### Architecture
- [System architecture](docs/architecture/system-architecture.md) — portal-between-core-and-UI
  topology, why a modular monolith, deployment shape.
- [Persistence decision](docs/architecture/persistence-decision.md) — why the portal
  cannot be a pure pass-through to core, and the resulting source-of-truth table.
- [Data model](docs/architecture/data-model.md) — every PostgreSQL table, its key columns,
  and the rules that follow from the persistence decision.

### Core integration
- [Core API mapping](docs/integration/core-api-mapping.md) — the 12 Ecobank VAS core
  endpoints the portal depends on, and the payload inconsistencies the core client must
  normalize.
- [Core gaps](docs/integration/core-gaps.md) — what core does not expose today, the
  interim workaround for each, and the full list of questions for Ecobank.
- [Core API requirements — MVP only](docs/integration/core-api-requirements-mvp.md) —
  the short version of the above to hand to Ecobank's core API team: the 13 P1 items
  that block MVP, nothing else.

### Portal API
- [API design](docs/api/api-design.md) — REST conventions, the response envelope, and
  every portal endpoint grouped by area (auth, admin, onboarding, merchant, dashboard,
  webhook, health).

### Engineering
- [Security](docs/engineering/security.md) — authentication, authorization, core
  integration security, webhook trust, data protection, web/API hardening.
- [Observability and resilience](docs/engineering/observability-resilience.md) — the
  three log records, metrics and tracing, alerts, resilience patterns, scheduled jobs,
  health checks.
- [Coding standards](docs/engineering/coding-standards.md) — stack, package layout, the
  non-negotiable DRY rules, code style, definition of done.
- [Git and environments](docs/engineering/git-and-environments.md) — branch-to-environment
  mapping, commit conventions, CI/CD pipeline, per-environment configuration.

### Process
- [Delivery plan](docs/process/delivery-plan.md) — build order and the six proposed
  two-week sprints.
- [Decisions and open questions](docs/process/open-questions.md) — the resolved
  product decisions, one deferred item, and the API gaps still open with Ecobank.

## How to use this doc set

- Start with the [product overview](docs/product/overview.md), then
  [system architecture](docs/architecture/system-architecture.md) — everything else
  builds on those two.
- Anyone touching the core integration should read
  [core API mapping](docs/integration/core-api-mapping.md) and
  [core gaps](docs/integration/core-gaps.md) first; most of the portal's defensive
  design (reconciliation, requery, status mapping) exists because of what's in those
  two documents.
- [Decisions and open questions](docs/process/open-questions.md) carries the
  resolved product decisions (pool balance, outlet history, approval SLA, retention)
  and the 11 questions still open with Ecobank — most of the core-integration
  defensive design exists because those 11 aren't answered yet.
