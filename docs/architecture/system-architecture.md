# System architecture

## Topology

The portal is a single Spring Boot backend (modular monolith) sitting between the
Ecobank VAS core and the dashboard UI:

- The **UI talks only to the portal** — never directly to Ecobank core.
- The **portal talks to core only through one `EcobankClient`** (see
  [coding standards](../engineering/coding-standards.md) for why there is exactly one).
- A browser must never hold core credentials or the AES encryption key, and the
  dashboard always reads from the portal's own store, never live from core on every
  page load.

> The original hand-drawn architecture sketch showed a direct line from core to the VA
> dashboard UI. This spec deliberately drops that line for the reasons above.

Request flow: requests flow down from the UI, through the gateway, to a feature
module, which either reads PostgreSQL or calls core through `EcobankClient`. Payments
flow the other way: core sends webhooks into the ingest module, which writes the
transaction mirror and updates the daily aggregates the dashboard reads. See
[persistence decision](persistence-decision.md) for what gets written where and when.

## Why a modular monolith, not microservices

One team, one bounded domain, and an MVP deadline. Feature packages with clear
boundaries (see the package layout in
[coding standards](../engineering/coding-standards.md)) let ingest or analytics split
out into separate services later if load demands it — the module boundaries are
designed to make that split cheap, not to avoid it.

## Deployment

- One Docker image, two or more instances behind the gateway per environment.
- App nodes are **stateless** so they scale horizontally — session state lives in
  PostgreSQL/Redis, not in-process (this also governs the token-caching and
  single-flight design in [core API mapping](../integration/core-api-mapping.md)).
- See [git and environments](../engineering/git-and-environments.md) for how this maps
  to dev/staging/production.

## Frontend

A separate SPA (framework chosen by the frontend team) consuming the portal's OpenAPI
contract — see [API design](../api/api-design.md). The portal has no opinion on the
frontend stack; the only contract is the REST API and its envelope.
