# Quickstart: User Story Intake and Admin Dashboard

This guide validates the planned feature after implementation. It does not
prescribe application setup beyond the required React/Vite, Node/Express, and
MongoDB/Mongoose stack.

## Prerequisites

- Supported Node.js LTS and npm versions recorded by the implementation.
- MongoDB configured as a replica set (or sharded cluster) so multi-record
  status/comment transactions are available.
- A development or test authentication adapter that issues trusted identities
  with `CONTRIBUTOR` or `ADMIN` roles. Do not treat the demo role chooser as
  authentication or authorization.
- Dependencies installed from the lockfile; no external AI or LLM service is
  needed.

## Install and start the application

From the repository root in PowerShell, install each project from its lockfile:

```powershell
npm --prefix backend ci
npm --prefix frontend ci
```

Set the connection string for a configured replica set and start the processes
in separate terminals:

```powershell
$env:MONGODB_URI = "mongodb://localhost:27017/story_intake?replicaSet=rs0"
npm --prefix backend run dev
```

```powershell
npm --prefix frontend run dev
```

Configure the trusted authentication adapter using the implementation's
documented server-side environment. Never place secrets in source control or
the frontend environment. Sign in with a test CONTRIBUTOR identity and a
separate test ADMIN identity supplied by the test adapter.

## Automated checks

Run the project scripts:

```powershell
npm --prefix backend test
npm --prefix frontend test
npm --prefix backend run lint
npm --prefix frontend run lint
npm --prefix frontend run build
npm --prefix frontend run test:e2e
```

The implementation MUST define these scripts in the corresponding package
manifests. The acceptance gates are:

- Unit and integration suites cover validation, ownership/RBAC, retries,
  status transitions, comments, search/filter, counts, and persistence failure.
- Contract checks validate the API responses against
  [`contracts/openapi.yaml`](contracts/openapi.yaml).
- Backend and core frontend business-logic line coverage is at least 80%.
- Lint completes with zero warnings and zero errors; build and tests pass.
- Automated accessibility checks and manual keyboard-only core workflow checks
  conform to WCAG 2.2 Level AA.
- A recorded test with at least 50 concurrent clients meets API p95 latency
  of 500 ms or less and HTTP 5xx below 1%.

## End-to-end acceptance walkthrough

1. As a CONTRIBUTOR, submit a story with all required fields. Confirm it has a
   server-assigned ID, owner, UTC timestamps, and `SUBMITTED` status.
2. Repeat the same request using its original `Idempotency-Key`. Confirm the
   same story is returned. Reuse that key with changed content and confirm
   `409 Conflict`.
3. As that contributor, list and open the story. Confirm another contributor
   cannot list or open it. Try selecting `ADMIN` in the demo/testing role
   chooser and confirm API calls remain contributor-authorized.
4. As an ADMIN, search/filter the story, open complete details, add a comment,
   and move it to `NEEDS_CHANGES` with an explanatory comment.
5. As the owner, edit and resubmit it with the current `ETag` in `If-Match`.
   Confirm it returns to `SUBMITTED`, increments its version, and retains
   comments and status history. Retry using a stale ETag and confirm
   `409 Conflict` without data loss.
6. As an ADMIN, review the story through an allowed transition. Confirm
   `APPROVED` and `REJECTED` are terminal, and a rejection without a comment
   is rejected.
7. Check dashboard status and priority totals against a fixture dataset,
   including categories with zero records.
8. Interrupt MongoDB in a controlled test. Confirm requests fail explicitly
   without success-shaped responses; restore service and verify all
   acknowledged submissions remain.
9. At 320 CSS pixels and desktop width, complete the core workflows using
   keyboard only and verify WCAG 2.2 AA checks.

## Reference

- API details and error responses: [`contracts/openapi.yaml`](contracts/openapi.yaml)
- Entity constraints and lifecycle: [`data-model.md`](data-model.md)
- Approved behaviors and metrics:
  [`spec.md`](spec.md)
