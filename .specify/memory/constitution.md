<!--
Sync Impact Report
Version change: uninitialized → 1.0.0
Modified principles: none; five principles established for the first time.
Added sections: Quality Standards; Harness and Agent Governance.
Removed sections: none.
Follow-up TODOs: Confirm the original ratification date and replace TODO(RATIFICATION_DATE).
-->

# User Story Intake and Admin Dashboard Constitution

## Core Principles

### I. Approved Architecture and Separation of Concerns
The frontend MUST use React with Vite. The backend MUST use Node.js with Express,
and MongoDB with Mongoose MUST be the persistence layer. Frontend and backend
responsibilities MUST remain separate, with the frontend consuming backend APIs
rather than accessing the database directly. Frontend features MUST use reusable
React components, and backend features MUST follow a modular architecture. This
keeps the approved stack consistent and the application maintainable.

### II. REST and Data Contracts
Backend APIs MUST follow REST resource conventions and use appropriate HTTP
methods, status codes, and JSON representations. API request data MUST be
validated at the boundary, and persisted data MUST be constrained by Mongoose
schemas. The API contract and approved acceptance criteria MUST remain
authoritative for client-server behavior.

### III. Security by Default
Every administrative API operation MUST enforce authentication and
authorization on the server; hiding a frontend control is not authorization.
All untrusted input MUST be validated before processing or persistence.
Credentials and secrets MUST be supplied through protected configuration and
MUST NOT be committed to source control or included in client bundles.

### IV. Automated Verification and Quality Gates
Each implemented user story MUST have verification for every approved
acceptance criterion. Business logic MUST have automated unit tests, and
client-server or persistence behavior MUST have integration tests. Changes
MUST pass the repository's tests and lint checks before approval; tests,
validations, lint rules, and security checks MUST NOT be bypassed or weakened.

### V. Maintainable and Reliable Delivery
Features MUST keep presentation, API handling, business logic, and persistence
in their appropriate frontend or backend modules. Shared behavior MUST be
implemented through reusable components or focused shared modules rather than
duplicated. Failures MUST be surfaced with appropriate HTTP responses and
diagnostic logging, without exposing secrets or sensitive data. Release
readiness MUST be demonstrated against the measurable standards below.

## Quality Standards

- **Security:** Before release, dependency and secret scans MUST report zero
  unresolved critical or high-severity dependency vulnerabilities and zero
  committed secrets. All administrative endpoints MUST have automated
  authentication and authorization tests. All write endpoints MUST reject
  invalid request data with a client error response.
- **Reliability:** A recorded load test for the agreed release profile of at
  least 50 concurrent clients MUST show API p95 response latency of 500 ms or
  less and an HTTP 5xx rate below 1%. Integration tests MUST verify explicit
  failure handling for invalid input and unavailable persistence dependencies
  on critical user-story workflows.
- **Testability:** CI MUST enforce at least 80% line coverage for backend
  business logic and core frontend business logic. Every approved acceptance
  criterion MUST link to an automated test or a documented human verification.
- **Maintainability:** CI MUST complete the configured lint checks with zero
  errors and zero warnings. Every change MUST pass the repository's build and
  test commands, and new business logic MUST have corresponding automated
  tests.

## Harness and Agent Governance

- Approved specifications and acceptance criteria are authoritative. Changes
  to them MUST be made only through approved Spec Kit artifacts, preserving
  their version history.
- Coding agents MUST NOT weaken acceptance criteria, skip tests, remove
  validations, disable linting rules, bypass security checks, alter harness
  results, or expose secrets.
- Before any repair attempt, the first generated implementation MUST be
  preserved as an immutable baseline. No more than three repair attempts may
  be made for any one failure; each attempt and its result MUST be recorded.
- Generated code remains untrusted until all applicable quality gates pass.
- Final approval MUST include a recorded human `ACCEPT` or `REJECT` decision.
  An agent MUST NOT represent unapproved work as accepted.

## Governance

This constitution governs feature specifications, implementation, review, and
release decisions for the application. Amendments MUST be made through the
approved Spec Kit workflow, reviewed for impact on existing specifications and
acceptance criteria, and recorded with version history. The constitution uses
semantic versioning: MAJOR for incompatible governance changes, MINOR for new
or materially expanded principles or sections, and PATCH for clarifications
that do not change obligations.

Each change MUST be checked against this constitution during specification,
implementation, and review. Reviews MUST verify applicable architecture,
security, reliability, testability, and maintainability standards, and MUST
confirm that the human acceptance decision is recorded. Any exception requires
an approved, versioned specification amendment; agents MUST NOT grant
themselves exceptions.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): original adoption date not provided | **Last Amended**: 2026-11-10
