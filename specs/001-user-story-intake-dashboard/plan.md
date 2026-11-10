# Implementation Plan: User Story Intake and Admin Dashboard

**Branch**: `001-user-story-intake-dashboard` | **Date**: 2026-11-10 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-user-story-intake-dashboard/spec.md`

## Summary

Build a responsive, accessible user-story workflow with separate React/Vite
frontend and Express/Node.js REST backend. Persist stories, comments, and
status history in MongoDB through Mongoose. Enforce contributor ownership,
administrator permissions, deterministic state transitions, and request
idempotency on the server. Keep authentication-provider specifics behind a
trusted adapter; do not introduce an external AI/LLM dependency.

## Technical Context

**Language/Version**: JavaScript; Node.js supported LTS (pin exact runtime in
implementation manifests and CI)

**Primary Dependencies**: React, Vite, Node.js, Express, Mongoose; Vitest and
React Testing Library (frontend tests), Node `node:test` and Supertest (backend
tests), Testcontainers MongoDB module (persistence tests), Playwright
(end-to-end and accessibility)

**Storage**: MongoDB through Mongoose; deployment MUST support transactions
(replica set or sharded cluster)

**Testing**: Frontend component/unit tests, backend domain tests, Express API
integration tests, disposable MongoDB persistence tests, Playwright
end-to-end/accessibility tests; at least 80% line coverage for backend
business logic and core frontend business logic

**Target Platform**: Modern evergreen desktop and mobile browsers; Node.js
server environment; minimum viewport width 320 CSS pixels

**Project Type**: Full-stack web application with separate frontend and
backend projects

**Performance Goals**: With at least 50 concurrent clients, API p95 latency
MUST be at most 500 ms and HTTP 5xx rate below 1%; in a 5,000-story dataset,
at least 95% of administrators MUST locate a target story within 10 seconds

**Constraints**: React/Vite, Node.js/Express, REST/JSON, MongoDB/Mongoose;
WCAG 2.2 Level AA; 99.5% monthly availability excluding scheduled maintenance;
restore core workflows within one hour without losing any acknowledged
submission; zero unresolved critical/high dependency vulnerabilities and zero
committed secrets; no external AI/LLM dependency

**Scale/Scope**: Two roles; four prioritized user journeys; 5,000-story
search usability target; at least 50 concurrent clients; one responsive
frontend, one REST backend, and MongoDB persistence

## Constitution Check

**Pre-design gate: PASS**

- **Approved stack and separation**: React with Vite, Node.js with Express,
  MongoDB with Mongoose; the browser consumes REST APIs and never accesses
  MongoDB. Frontend and backend deployable responsibilities remain separate.
- **Modular architecture**: Frontend grouped by contributor/admin workflows
  with shared UI and API modules; backend separates routes/controllers,
  validation, services, repositories, and persistence models.
- **REST and data contracts**: Resource-oriented JSON endpoints and the
  approved request/response semantics are specified in
  [`contracts/openapi.yaml`](contracts/openapi.yaml).
- **Security**: Every protected operation uses a trusted normalized principal;
  role choice in the UI is never authority. Boundary validation, ownership
  scoping, safe errors, and secret/dependency scans are planned.
- **Verification**: Acceptance behaviors map to unit, API integration,
  persistence, and E2E tests. Coverage, lint, build, accessibility, load, and
  failure gates remain required.
- **Governance**: The approved specification is treated as authoritative. No
  test, validation, lint, or security gate is to be bypassed; generated
  implementation remains untrusted until gates pass and human acceptance is
  recorded.

No constitutional exception is required.

## Architecture and Key Decisions

1. **Trust boundary**: A deployment-configured authentication adapter
   validates the existing identity mechanism and supplies a stable `userId`
   and `role`. Application services receive this normalized principal, not
   provider-specific tokens or claims. The exact provider remains a
   deployment dependency; absence or invalidity of trusted identity fails
   closed. Development/test identities are isolated to test configuration.
2. **Authorization**: Express middleware authenticates first; role and
   ownership checks are enforced again in the relevant service/query. A
   contributor's attempt to open another owner's story is indistinguishable
   from an unknown story (`404`). The client-side role chooser only changes
   view mode.
3. **Story lifecycle**: One transition policy defines the allowed transition
   table. Services apply it atomically; the browser cannot assign status,
   actor, ownership, or timestamps. `APPROVED` and `REJECTED` are terminal.
4. **Idempotency**: `POST /stories` requires an `Idempotency-Key`. Store a
   normalized-content fingerprint and a unique `(ownerId, key)` index.
   Matching retries return the original story; key reuse with different
   content returns the spec-required `409`. A new key may create a distinct
   story with identical content.
5. **Concurrency**: Story representations expose a version/ETag. Resubmission
   and status changes require `If-Match`; compare-and-update failure is
   `409 Conflict`. This prevents a stale administrator decision or
   contributor edit from overwriting newer state.
6. **Persistence**: Store current story content/status separately from
   append-only comments and status events. MongoDB transactions couple a
   status change, its status event, and any required comment. Initial story
   creation and its initial event are also committed atomically. The MongoDB
   deployment must support transactions.
7. **Search and paging**: Admin search uses escaped, case-insensitive literal
   substring matching over the specified text fields. Apply filters before
   pagination; order by latest submission descending then ID ascending.
   Validate all query parameters and enforce page size 1-100 (default 20).
8. **Errors and observability**: A request-ID middleware assigns/propagates a
   safe request ID. Central error middleware maps expected domain errors to
   the approved `API-001` JSON envelope and status codes. Structured server
   logs record request ID, operation, status, duration, and safe error class;
   never log credentials, full sensitive request bodies, or stack traces to
   client responses.
9. **Dashboard counts**: Aggregate persisted stories by current status and
   priority; initialize every known category to zero so absent groups remain
   explicit. Do not cache until measurements demonstrate the need; any future
   cache must preserve the spec's current-count behavior.
10. **Accessibility**: Use semantic controls, visible labels, keyboard
    operability, clear focus indicators, and accessible names. Automated
    scans supplement (not replace) manual keyboard-only evaluation.

## API Endpoints

Base path: `/api/v1`. The complete schema, validation constraints, parameters,
and response shapes are in [`contracts/openapi.yaml`](contracts/openapi.yaml).

| Method and path | Access | Purpose |
|---|---|---|
| `GET /stories` | CONTRIBUTOR (own records); ADMIN (all) | List; admin may filter by `q`, `status`, `priority`; paginated |
| `POST /stories` | CONTRIBUTOR | Create story; requires `Idempotency-Key` |
| `GET /stories/{storyId}` | CONTRIBUTOR (owner); ADMIN | Complete details, comments, and status history |
| `PATCH /stories/{storyId}` | CONTRIBUTOR (owner, `NEEDS_CHANGES`) | Edit content and resubmit; requires `If-Match` |
| `POST /stories/{storyId}/comments` | ADMIN | Add review comment; requires current `If-Match` |
| `PATCH /stories/{storyId}/status` | ADMIN | Transition status; optional comment, required for `NEEDS_CHANGES`/`REJECTED`; requires `If-Match` |
| `GET /dashboard/summary` | ADMIN | Counts by every status and priority, including zeroes |

### Error Handling

- `400`: malformed JSON or request shape.
- `401`: missing or invalid trusted authentication.
- `403`: authenticated role is insufficient.
- `404`: missing resource; also used to conceal another contributor's story.
- `409`: stale ETag, disallowed/no-op transition, or idempotency key reused
  with different content.
- `422`: well-formed request with invalid fields.
- `500`: unexpected failure, with safe client message and internal diagnostics
  logged using request ID.
- `503`: dependency outage prevents processing.

Every error uses the spec's `{ "error": { "code", "message", "fieldErrors?",
"requestId" } }` envelope. Coupled writes fail as a unit and never return
success-shaped fallback data.

## Project Structure

### Documentation (this feature)

```text
specs/001-user-story-intake-dashboard/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── openapi.yaml
├── checklists/
│   └── requirements.md
└── tasks.md
```

### Source Code (repository root)

```text
frontend/
├── src/
│   ├── app/                 # App shell, routing, auth context integration
│   ├── features/
│   │   ├── stories/         # Contributor list, form, detail, revision
│   │   └── admin/           # Dashboard, search/filter, review screens
│   ├── components/          # Reusable accessible controls and layout
│   ├── services/            # Typed API client and error mapping
│   └── styles/
└── tests/
    ├── unit/
    └── e2e/
backend/
├── src/
│   ├── app.js               # Express composition, no listen side effect
│   ├── server.js            # Process startup and graceful shutdown
│   ├── config/              # Environment parsing, database connection
│   ├── api/
│   │   ├── routes/
│   │   ├── controllers/     # HTTP translation only
│   │   └── validators/      # Request schemas and boundary validation
│   ├── middleware/          # Auth adapter, role guards, request ID, errors
│   ├── services/            # Workflow, ownership, idempotency, summaries
│   ├── repositories/        # Mongoose queries and atomic operations
│   ├── models/              # Story, comment, status event schemas/indexes
│   └── errors/              # Typed domain/API error mapping
└── tests/
    ├── unit/
    ├── integration/
    └── fixtures/
```

**Structure Decision**: Separate `frontend/` and `backend/` projects under
one repository. Within each, organize by feature and responsibility. The API
service owns validation, access decisions, workflow transitions, and
persistence; React owns presentation and user interaction. No database or
security decision is delegated to the browser.

## Testing Strategy

- **Backend unit**: Pure validation rules, normalization/fingerprints,
  transition table, role policy, error-to-status mapping, and summary
  zero-filling; Node `node:test`.
- **API integration**: Supertest against a constructed Express app with
  controlled trusted-principal fixtures. Verify every protected route, all
  error envelope codes, malformed/invalid payloads, ownership privacy, and
  role chooser non-authority.
- **MongoDB integration**: Testcontainers MongoDB replica set; verify schema
  constraints, unique idempotency index under concurrency, transactions,
  version preconditions, comment/event ordering, filters, and count accuracy.
- **Frontend component**: Vitest and React Testing Library for accessible form
  validation, server errors, role-view selection, list/filter/pagination, and
  review interactions.
- **End-to-end**: Playwright covers submit/list/detail, administrator search
  and review, change request and resubmission, terminal states, dashboard
  counts, and mobile/keyboard flows.
- **Quality gates**: At least 80% business-logic line coverage on both sides;
  lint with zero warnings/errors; tests and production build pass; zero
  unresolved critical/high dependency findings and zero committed secrets.
  Accessibility checks include automated scans and manual WCAG 2.2 AA
  keyboard review.
- **Operational gates**: At least 50 concurrent clients with p95 at most
  500 ms and 5xx below 1%; availability at least 99.5% monthly excluding
  scheduled maintenance; demonstrate one-hour service recovery with no loss
  of acknowledged submissions.

## Dependencies and Risks

| Dependency / risk | Plan |
|---|---|
| Existing authentication provider is not named | Keep a provider-neutral adapter; resolve provider, credential transport, and stable subject mapping before deployment. Never accept browser-selected role. |
| MongoDB multi-document transaction support | Require replica set or sharded cluster in dev/test/prod; integration-test transactions against a disposable replica set. |
| Search uses literal substring semantics | Escape input before case-insensitive matching; benchmark with 5,000 stories and 50 concurrent clients against the required p95 target. |
| Idempotency and concurrent duplicate requests | Unique owner/key index plus fingerprint; test duplicate-key races and distinct-key identical submissions. |
| Availability/recovery targets need operational support | Document monitoring and backup/restore procedures during implementation; verify restoration and acknowledged-write durability in a controlled test. |
| Detailed observability signals are not specified | At minimum record request ID, route/operation, response code, duration, and dependency failure class without sensitive values; refine dashboards during implementation without changing acceptance criteria. |

## Post-Design Constitution Check

**Gate: PASS**

- The selected design retains the required React/Vite + Node/Express +
  MongoDB/Mongoose stack and strict client/server responsibility boundary.
- REST API behavior matches the approved spec, including its error envelope,
  status codes, idempotency conflict behavior, and auth/ownership rules.
- MongoDB transaction support is an explicit deployment constraint required
  by atomic status/comment/event updates; it does not alter application scope.
- Reusable frontend components and modular backend boundaries are represented
  in the source layout.
- Tests cover acceptance criteria and constitution quality gates, including
  security, accessibility, coverage, reliability, and maintainability.
- No external AI/LLM dependency, unapproved specification change, or bypass of
  quality gates is introduced.

No constitutional exception is required.

## Complexity Tracking

No constitution violations; no complexity exception requested.
