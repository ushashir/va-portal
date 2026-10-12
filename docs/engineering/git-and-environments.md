# Git management and environments

Three long-lived branches map one-to-one to three environments: `dev` → Dev,
`staging` → Staging, `main` → Production. Code only moves forward through pull
requests; nobody pushes directly to any of them.

## Branches

| Branch | Environment | Ecobank core | Deploys | Protection |
|---|---|---|---|---|
| `main` | Production | Prod (`virtualacctservice.ecobank.com`) | On tag `vX.Y.Z`, manual approval | 2 approvals incl. tech lead, CI green, no direct push, no force push, signed tags |
| `staging` | Staging / UAT | UAT (`eidev.ecobank.com:7505`) | Automatic on merge | 1 approval, CI green, no direct push |
| `dev` | Dev | UAT or WireMock | Automatic on merge | 1 approval, CI green |
| `feature/ECO-123-va-block`, `bugfix/ECO-140-csv-escape` | Local / preview | WireMock | Optional preview | None |
| `hotfix/ECO-150-webhook-dup` | Cut from `main` | n/a | Through `main` | Same as `main` |

## Flow

1. Branch `feature/<ticket>-<short-desc>` from `dev`.
2. Open a PR to `dev`; squash-merge after review and green CI.
3. When a release is ready, open a release PR `dev` → `staging` (merge commit). QA and
   Ecobank UAT sign-off happen on staging.
4. Open PR `staging` → `main` (merge commit), tag `vX.Y.Z`, approve the production
   deploy.
5. **Hotfix**: branch from `main`, PR to `main`, tag a patch version, then merge
   `main` back into `staging` and `dev` the same day.

## Commits and versions

- Conventional Commits: `feat(va): block and unblock virtual accounts`,
  `fix(webhook): ignore duplicate postingReferenceNo`, `chore`, `refactor`, `test`,
  `docs`.
- Every commit and PR title carries the ticket id.
- Semantic versioning; `CHANGELOG.md` generated from commits on release.
- PR template: what and why, screenshots or sample requests, migration notes,
  rollback plan, checklist from the
  [definition of done](coding-standards.md#definition-of-done).
- `.gitignore` covers `application-local.yml`, `.env`, keys; Gitleaks runs in CI and
  as a pre-commit hook.

## CI/CD pipeline (every PR and merge)

1. Build and Spotless/Checkstyle
2. Unit tests, then integration tests (Testcontainers + WireMock)
3. SonarQube quality gate
4. OWASP Dependency-Check and Gitleaks
5. Build Docker image, tag with commit SHA, scan image
6. Deploy to the branch's environment, run Flyway, run smoke tests against
   `/actuator/health/readiness`

## Configuration per environment

- Spring profiles `local`, `dev`, `staging`, `prod`; only non-secret values in
  `application-<profile>.yml`.
- Secrets injected from the secrets manager per environment; **production keys are
  never available to dev or staging.**
- **Production data is never copied to lower environments**; staging uses masked or
  synthetic data.
