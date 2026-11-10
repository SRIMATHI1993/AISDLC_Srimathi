# Research: User Story Intake and Admin Dashboard

## Architecture and stack decisions

### React with Vite

- **Decision**: Organize the client by product area (`stories`, `admin`) and
  keep routing, shared UI, and API access in shared modules. Build components
  around meaningful user responsibilities.
- **Rationale**: Intake and administration are distinct workflows, while
  shared client modules keep API behavior and common controls consistent.
  React describes component decomposition around UI and behavior; Vite provides
  development and production tooling without imposing an application layout.
- **Alternatives considered**: A global `pages/components/services` layout is
  acceptable for a small app, but feature grouping keeps the two workflows
  easier to evolve independently.
- **References**: [React: Thinking in React](https://react.dev/learn/thinking-in-react);
  [Vite: Getting Started](https://vite.dev/guide/).

### Accessibility verification

- **Decision**: Target WCAG 2.2 Level AA as required by the approved
  specification. Use accessible-name and keyboard interaction tests,
  automated page scans, and manual keyboard-only evaluation.
- **Rationale**: Automated scans do not identify every accessibility problem;
  manual review is necessary for the specified core workflows.
- **Alternatives considered**: Automated-only checks are insufficient for
  complete conformance.
- **References**: [WCAG 2.2](https://www.w3.org/TR/WCAG22/);
  [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/);
  [Playwright accessibility testing](https://playwright.dev/docs/accessibility-testing).

### Express REST architecture and errors

- **Decision**: Separate resource routers and HTTP handlers from application
  services and persistence. Validate requests at the boundary, enforce
  authorization in middleware/services, and centralize error translation.
- **Rationale**: Express supports composable routing and middleware; this
  separation makes business rules independently testable and avoids divergent
  per-route behavior.
- **Alternatives considered**: A framework-prescribed architecture is
  unnecessary; Express does not prescribe one folder structure.
- **Error contract**: The approved specification's `API-001` JSON envelope,
  stable error codes, field errors, and request ID are authoritative. RFC 9457
  Problem Details was considered as a standard alternative but is not adopted
  because it would change the approved response shape. Internal diagnostics
  remain in server logs; responses do not disclose stack traces or persistence
  details.
- **References**: [Express routing](https://expressjs.com/en/guide/routing.html);
  [Express error handling](https://expressjs.com/en/guide/error-handling/);
  [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html).

### MongoDB and Mongoose data design

- **Decision**: Keep current story content and status on a story document;
  store comments and status events in separate collections with story
  references. Add indexes for owner access, stable lists, filters, and
  owner-scoped idempotency.
- **Rationale**: Comments and status history grow over time and are
  independently appended/read. Separate collections avoid unbounded story
  documents. Mongoose schema validation complements API-boundary validation.
- **Atomicity**: Use MongoDB transactions for a status transition, event,
  and required explanatory comment so they commit together. The deployment
  must support transactions through a replica set or sharded cluster. Initial
  story creation and its initial status event likewise commit together.
- **Alternatives considered**: Embedding history is simpler if it is bounded
  and always read with the story; no bound is present in the approved spec.
  MongoDB single-document atomicity is insufficient for the chosen separate
  collections, hence the replica-set-capable deployment requirement.
- **References**: [MongoDB data modeling](https://www.mongodb.com/docs/manual/data-modeling/);
  [MongoDB modeling best practices](https://www.mongodb.com/docs/manual/data-modeling/best-practices/);
  [Mongoose schemas](https://mongoosejs.com/docs/guide.html);
  [Mongoose validation](https://mongoosejs.com/docs/validation.html);
  [MongoDB atomicity](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/);
  [Mongoose transactions](https://mongoosejs.com/docs/transactions.html).

### Idempotent story creation

- **Decision**: Require an `Idempotency-Key` per authenticated contributor and
  store it on the story under a unique `(ownerId, idempotencyKey)` index with
  a fingerprint of normalized submitted content. Same key and same content
  returns the existing result; same key with changed content returns the
  specification-required `409 Conflict`. Keys remain reserved for the story's
  lifetime.
- **Rationale**: A unique database index protects against concurrent
  duplicate requests, and the payload fingerprint distinguishes safe retries
  from key misuse. A new key may submit identical content as a separate story.
- **Alternatives considered**: Content-based de-duplication would reject
  legitimate distinct stories and is explicitly not required. The IETF
  Idempotency-Key document is an Internet-Draft, not a finalized RFC; its
  differing suggested conflict status does not supersede the approved
  specification.
- **References**: [IETF Idempotency-Key draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/);
  [MongoDB unique indexes](https://www.mongodb.com/docs/manual/core/index-unique/).

### Authentication integration boundary

- **Decision**: Keep provider-specific login, credential/session validation,
  and claim mapping behind a deployment-provided authentication adapter. The
  application consumes only a normalized stable `userId` and role. Fail
  closed if no trusted principal is available. The demo/testing role chooser
  changes only client presentation and never API authority.
- **Rationale**: The approved spec relies on an existing trusted identity
  provider but does not name its protocol or vendor. Provider-neutral
  authorization avoids coupling data and business logic to mutable claims or
  one identity SDK.
- **Alternatives considered**: Adding local accounts or selecting an external
  identity provider expands scope and contradicts the current assumptions.
- **Dependency to resolve before deployment**: Identify the existing
  authentication provider and configure its trusted adapter. Do not infer the
  provider or accept role information from an untrusted browser.
- **References**: [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html);
  [OAuth 2.0 Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html).

## Automated test strategy

- **Frontend unit/component tests**: Vitest and React Testing Library, querying
  by accessible names and exercising real user interactions.
- **Backend domain tests**: Node's built-in `node:test` runner for
  validation, transition tables, access policies, and error mapping.
- **API integration tests**: Supertest against the Express application for
  request validation, roles, status codes, and response envelopes.
- **Persistence integration tests**: Testcontainers with a disposable MongoDB
  replica set for Mongoose schema/index behavior, transactions, ownership
  scoping, and idempotency races. Do not treat mocked persistence as proof of
  these behaviors.
- **End-to-end tests**: Playwright for contributor intake/revision and admin
  review/dashboard workflows; include automated accessibility scans and
  manual keyboard-only checks.
- **Alternatives considered**: One JavaScript test runner could host multiple
  suites. Keep test layers and real persistence checks distinct regardless of
  runner.
- **References**: [Vitest](https://vitest.dev/guide/);
  [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/);
  [Node test runner](https://nodejs.org/api/test.html);
  [Supertest](https://github.com/forwardemail/supertest);
  [Testcontainers MongoDB module](https://node.testcontainers.org/modules/mongodb/);
  [Playwright accessibility testing](https://playwright.dev/docs/accessibility-testing).
