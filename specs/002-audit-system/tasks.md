---

description: "Implementation tasks for the Audit System feature"
---

# Tasks: Audit System

**Input**: Design documents from `specs/002-audit-system/`

**Prerequisites**: `spec.md`, `plan.md`, `research.md`, `data-model.md`, `contracts/audit.openapi.json`, and `quickstart.md`

**Tests**: Tests are required by the specification and constitution, and are written before the code they cover. Use the existing xUnit service test project and Angular/Vitest specs. PostgreSQL-backed checks cover migrations, grants, and concurrency because SQLite cannot verify PostgreSQL permissions; they run only when `AUDIT_TEST_POSTGRES` is set, so plain `dotnet test` still passes without PostgreSQL.

**Organization**: Tasks are grouped by the four user stories. Shared persistence, the event contract, and delivery infrastructure are foundational and block all story implementation.

**Release gate**: The initial capability is released only when US1 (recording), US3 (Admin review), and US4 (record protection and reliable recording) are all complete and their gates verified. US1 alone is a demonstration, not a release.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Parallelizable after prerequisites because it uses separate files and has no dependency on incomplete tasks.
- **[Story]**: User-story label matching `spec.md` (`US1`–`US4`); Setup, Foundational, and Polish tasks omit the story label.
- Every task includes concrete paths inside the existing Angular or service submodule. Paths below are relative to the repository root; service paths start with `iit-academic-portal-service/` and Angular paths with `iit-academic-portal/`.

## Phase 1: Setup

**Purpose**: Confirm existing conventions and name the deployment-owned decisions that gate release.

- [X] T001 [P] Verify Angular, ASP.NET Core, EF Core, PostgreSQL, and test commands against `iit-academic-portal/README.md`, `iit-academic-portal/package.json`, and `iit-academic-portal-service/README.md`; record any path or version discrepancy in `specs/002-audit-system/plan.md` without scaffolding a parallel application.
- [ ] T002 Record the deployment owner's decisions in `iit-academic-portal-service/README.md` under an "Audit operations" heading: the production durable operational-event sink, the protected alert destination, the authorized recovery owner for exhausted security events, and confirmation or replacement of the 1,000,000-event SC-006 volume. These decisions gate release and the Production configuration in T019, not the start of Phase 2; development uses the file sink. Do not treat console logging as durable or introduce an unapproved provider.

---

## Phase 2: Foundational

**Purpose**: Establish event invariants, PostgreSQL persistence, the application writing boundary, request context, and durable security-event delivery. Test fixtures and tests come first. These tasks block all user stories.

**Checkpoint**: Do not start user-story work until T003–T009 exist and T010–T021 make them pass.

### Test infrastructure and foundation tests (write first)

- [X] T003 [P] Add the PostgreSQL test fixture in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Infrastructure/PostgresFixture.cs`: read the administrator connection string from `AUDIT_TEST_POSTGRES`; create a disposable database with a separate DDL owner role and a restricted runtime role; apply the real migrations and `database/postgresql/runtime-grants.sql`; drop everything afterward. Mark tests `[Trait("Category", "PostgreSQL")]` and skip them with a clear message when the variable is not set.
- [X] T004 [P] Add a test-only stand-in business entity and event definition in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/Fixtures/StandInBusinessEntity.cs`, registered through the test host's DbContext configuration only (not in the production model), so atomic business-event behavior can be proven before any business feature exists.
- [X] T005 [P] Add recorder tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditEventRecorderTests.cs` for business/security/diagnostic classification, unique IDs, actor-null behavior, UTC timestamps, outcomes, server-generated correlation IDs, source, field and metadata size limits, and rejection of client-submitted event claims.
- [X] T006 [P] Add transaction tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditTransactionTests.cs` using the T004 stand-in: a business change and its staged event commit together or both roll back when either write fails; a rejected validation error records nothing.
- [X] T007 [P] Add outbox worker tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/SecurityAuditOutboxTests.cs` for retry with backoff, process restart, idempotent delivery by event ID, two workers never processing one item at once, an expired lease being claimed again, retry exhaustion retained for recovery, authorized retry and handled transitions, and no silent discard.
- [X] T008 [P] Add fallback-sink tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/SecurityAuditFallbackTests.cs` for unchanged primary outcomes when the outbox write fails, a safe event envelope reaching the sink, alert signaling, and Production startup failing when no durable sink is configured.
- [X] T009 [P] Add PostgreSQL integration tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/PostgresAuditPermissionsTests.cs`: the runtime role can SELECT and INSERT audit events but UPDATE and DELETE fail; the runtime role cannot delete outbox rows but can update delivery state; the runtime role does not own the audit table; the API can sign in and use the existing Identity and session tables under the runtime role; a table created by a later migration is usable without a new grant; event IDs are unique; concurrent inserts with identical timestamps lose nothing; historical actor and entity IDs survive deletion of the source row.

### Implementation

- [X] T010 Create the domain audit types in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Auditing/AuditEvent.cs`, `AuditEventCategory.cs`, and `AuditOutcome.cs`; preserve `data-model.md` constraints verbatim: Event ID “Required, unique, generated by the trusted server before persistence”; Occurred at UTC “Required, UTC; immutable after insert”; Category “Required: business or security”; Outcome “Required: success, failure, or denied”; Actor user ID “Nullable for unauthenticated/system activity and failed sign-ins; no live-user FK that would cascade-delete history; no display name or email is stored”; Entity type “Nullable when no target is known; at most 100 characters”; Entity ID “Nullable when no target is known; at most 200 characters; stored without a live-entity FK”; Correlation ID “Always server-generated for events raised during an HTTP request; nullable only for events with no request; at most 100 characters”; Source “Nullable when no request exists; at most 45 characters; no user-agent or other client-supplied value”; Metadata “Optional structured values; only data approved for this event type; at most 4 KB”; Change summary “Optional entries of field name and safe previous/new values, or a safe changed indicator; never contains non-allowlisted values; at most 16 KB”.
- [X] T011 Create the outbox delivery-state model in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Auditing/SecurityAuditOutboxItem.cs`; preserve `data-model.md` constraints verbatim: Event ID “Required, unique; primary idempotency key”; Safe event envelope “Required; already redacted/allowlisted before enqueue”; Enqueued at UTC “Required, UTC”; Delivery state “Required: Pending, RetryScheduled, Exhausted, Delivered, or Handled”; Attempt count “Non-negative; incremented for every attempt”; Next attempt at UTC “Nullable when not scheduled”; Last attempt at UTC “Nullable before first attempt”; Last failure code “Nullable; no secrets or raw exception payload”; Lease until UTC “Nullable; set when a worker claims the item; an expired lease may be claimed again”; Delivered at UTC “Nullable until Delivered”; Handled at/by/reason “Nullable; required for Handled; reason is safe, actor is a stable historical ID”.
- [X] T012 Map audit and outbox entities, keys, unique constraints, required fields, length limits, and filter indexes in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs`; preserve the `data-model.md` requirements that historical actor/entity IDs have no live-row foreign keys, event ID is unique for idempotency, and audit events are immutable after insert. Index the outbox by delivery state and next-attempt time.
- [X] T013 Add the version-controlled EF Core migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/` for audit events, the security outbox, unique event IDs, and the indexes in `specs/002-audit-system/data-model.md`; confirm the generated PostgreSQL DDL adds no actor/entity cascade deletes.
- [X] T014 Define the application event-writing boundary in `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IAuditEventRecorder.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Auditing/`: an `AuditEventDefinition` (event type, category, approved change fields) declared by the owning feature; a `Stage` method that adds a mandatory business event to the caller's current unit of work without saving; a `RecordSecurityAsync` method that writes to the outbox; and an `IAuditRequestContext` supplying actor, correlation ID, and source. Accept no client-submitted event claims.
- [X] T015 Implement the request context in `iit-academic-portal-service/src/IitAcademicPortal.Api/Auditing/AuditRequestContext.cs` and register trusted forwarded-header handling in `iit-academic-portal-service/src/IitAcademicPortal.Api/PortalApi.cs`: a server-generated correlation ID per request, the client IP taken only from configured trusted proxies, and the actor from the validated session claims. Document the `ForwardedHeaders` settings in `iit-academic-portal-service/README.md` and record the client IP, never a user-agent.
- [X] T016 Implement validation, classification, redaction, size limits, and the allowlisted change-summary builder in `iit-academic-portal-service/src/IitAcademicPortal.Application/Auditing/AuditEventRecorder.cs` and `AuditChangeSummaryBuilder.cs`: omit every secret and unapproved before/after value, and default to omission for any field the owning feature has not approved. Add `AuditMetrics` in the same folder using `System.Diagnostics.Metrics` with counters for anonymous 401 responses, outbox write failures, retry exhaustion, fallback failures, and event-loss risk.
- [X] T017 Implement audit-event and outbox persistence in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/AuditEventRepository.cs` and `SecurityAuditOutboxRepository.cs`: idempotent by event ID, append-only for formal events, and a conditional-update lease claim for outbox items that works on both PostgreSQL and SQLite.
- [X] T018 Implement the persisted outbox worker in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Auditing/SecurityAuditOutboxWorker.cs`: claim items by lease, retry with bounded backoff, retain exhausted envelopes for authorized recovery, acknowledge only after formal event persistence, and safely acknowledge a replay whose event ID already exists.
- [X] T019 Implement the durable fallback for outbox-write failures in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Auditing/DurableSecurityEventSink.cs` and `FileSecurityEventSink.cs`: an `IDurableSecurityEventSink` abstraction, a JSON-lines file sink for development, a critical alert signal, and options validated at startup so that outside Development the service refuses to start unless the deployment owner's durable sink (T002) is configured. Document the settings in `iit-academic-portal-service/README.md`; console logging alone must fail the release gate.
- [X] T020 Set up the separate database accounts: add a `ConnectionStrings:PortalMigrations` setting; select it for `dotnet ef` using `EF.IsDesignTime` in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs`; add `iit-academic-portal-service/database/postgresql/runtime-grants.sql` granting the runtime role what it needs on every existing table and sequence, setting default privileges for tables created by later migrations, and limiting audit events to SELECT and INSERT and outbox rows to no DELETE. Update the setup commands in `iit-academic-portal-service/README.md`, `specs/001-role-based-auth/quickstart.md`, and `iit-academic-portal/README.md` where they run `dotnet ef`.
- [X] T021 Register the recorder, request context, repositories, worker, fallback sink, metrics, and validated delivery options in `iit-academic-portal-service/src/IitAcademicPortal.Application/DependencyInjection.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Api/Program.cs` using existing DI and options conventions.

**Checkpoint**: Domain contract, PostgreSQL schema, transaction-capable recorder, request context, outbox retry path, durable failure fallback, and database account split exist, and the foundation tests pass.

---

## Phase 3: User Story 1 - Record Important Events (Priority: P1)

**Goal**: Provide the reusable server-side event contract and record the initial security events; future business features submit declared events through the same boundary.

**Independent Test**: Exercise sign-in, password-reset, session, role-switch, access-denial, and audit-mutation paths, and submit a stand-in business event; verify event type, category, actor, outcome, UTC time, correlation, source, and secret redaction.

### Tests for User Story 1

- [X] T022 [P] [US1] Add service API tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/SecurityEventRecordingTests.cs` for exactly one `auth.sign-in` event per attempt (a failure records no account and no submitted email), `auth.password-reset.completed` outcomes, `auth.session.revoked` with each reason (logout, idle timeout, password reset), and `auth.role.switched` success and denial with no duplicate `access.denied`; assert no passwords, proofs, cookies, or session handles are recorded.
- [X] T023 [P] [US1] Add access-denial tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AccessDeniedEventTests.cs`: an authenticated 403 records `access.denied` with the matched route template and method (never the raw path or query); an anonymous 401, including `GET /api/auth/sessions/current` while signed out, records no event and increments the anonymous-401 counter; the original 401/403 responses are unchanged.

### Implementation for User Story 1

- [X] T024 [US1] Record sign-in success and failure in `iit-academic-portal-service/src/IitAcademicPortal.Application/Authentication/PortalAuthenticationService.cs` as one `auth.sign-in` event per attempt, preserving generic client responses; a failed attempt records no account, and no submitted email or password appears in any event.
- [X] T025 [US1] Record password-reset completion outcomes in `iit-academic-portal-service/src/IitAcademicPortal.Application/PasswordRecovery/PasswordRecoveryService.cs`; record safe outcome and context only, never the proof.
- [X] T026 [US1] Record session revocation as `auth.session.revoked` with its reason: logout and password-reset revocation in `iit-academic-portal-service/src/IitAcademicPortal.Application/Authentication/PortalAuthenticationService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/SessionsController.cs`, and idle-timeout revocation in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/AuthSessionRepository.cs`, which revokes on the request path. Add no extra database write on ordinary authenticated requests.
- [X] T027 [US1] Record active-role switch outcomes as `auth.role.switched` in `iit-academic-portal-service/src/IitAcademicPortal.Application/Authentication/PortalAuthenticationService.cs`, including denial for an unassigned role.
- [X] T028 [US1] Record authenticated 403 responses as `access.denied` through `HandleForbiddenAsync` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Authentication/PortalSessionAuthenticationHandler.cs` and the request pipeline in `iit-academic-portal-service/src/IitAcademicPortal.Api/PortalApi.cs`, using the non-recursive security-event path. In `HandleChallengeAsync`, record nothing for anonymous 401 responses and only increment the anonymous-401 counter. Preserve the original status codes.
- [X] T029 [US1] Publish the internal event-definition registry in `iit-academic-portal-service/src/IitAcademicPortal.Application/Auditing/AuditEventDefinitions.cs` with the initial security definitions (`auth.sign-in`, `auth.password-reset.completed`, `auth.session.revoked`, `auth.role.switched`, `access.denied`, `audit.mutation.attempted`, `audit.review.accessed`); verify with the T004 stand-in that an owning feature can declare an event type and changed-field allowlist, and that no public client event-write endpoint exists.

**Checkpoint**: The reusable event contract works independently, and the initial security actions are recorded without auditing anonymous traffic.

---

## Phase 4: User Story 2 - Understand Relevant Changes (Priority: P1)

**Goal**: Preserve approved before/after values for audited changes while redacting secrets, personal data, and every field not explicitly allowlisted.

**Independent Test**: Submit allowlisted scalar and structured field changes alongside sensitive and unapproved fields; verify only approved safe values are persisted and returned.

### Tests for User Story 2

- [X] T030 [P] [US2] Add change-summary tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditChangeSummaryTests.cs` for old/new values, changed indicators, unchanged values, structured JSON values, size limits, and omission of unapproved values.
- [X] T031 [P] [US2] Add redaction tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditRedactionTests.cs` asserting passwords, reset proofs, tokens, cookies, session handles, cryptographic secrets, and unapproved personal values are absent from persisted payloads, API responses, and captured ordinary application logs (SC-003).

### Implementation for User Story 2

- [X] T032 [US2] Add a defense-in-depth secret guard in `iit-academic-portal-service/src/IitAcademicPortal.Application/Auditing/AuditSecretGuard.cs`, applied to metadata and change values even when a field is allowlisted: reject or mask values under credential-like field names and values that look like tokens, and keep event payloads out of ordinary application logs.
- [X] T033 [US2] Document how an owning feature declares its event types and approved change fields, stages a business event in its own transaction, and records security events, in `iit-academic-portal-service/README.md` under "Audit integration for feature owners" (FR-006, FR-012).

**Checkpoint**: Change history is understandable for approved fields and exposes no secret or non-allowlisted value in storage, responses, or logs.

---

## Phase 5: User Story 3 - Search and Review Audit History (Priority: P1)

**Goal**: Let only a session currently acting in Admin search, paginate, and inspect safe audit history, with every read itself recorded.

**Independent Test**: Search and retrieve detail as Admin; repeat as Student, Teacher, Coordinator, and a Teacher+Coordinator session in both active roles. Verify denial unless the current session's active role is Admin, including after role switches.

### Tests for User Story 3

- [X] T034 [P] [US3] Add authorization tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditReviewAuthorizationTests.cs` for Admin success, non-Admin 403, unauthenticated 401, a multi-role account denied after switching away from Admin and allowed after switching back, and no client-supplied role bypass.
- [X] T035 [P] [US3] Add search tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditEventSearchTests.cs` for every contract filter, inclusive/exclusive UTC boundaries, invalid ranges, newest-first keyset order, bounded page size, cursor continuation, events with identical timestamps, inserts between pages causing no duplicates or omissions, the resolved `actorDisplay` for active, removed, and absent actors, missing event behavior, and each successful search or detail view recorded as `audit.review.accessed` without the returned contents.
- [X] T036 [P] [US3] Add Angular/Vitest tests in `iit-academic-portal/src/app/features/audit/audit-page.spec.ts` for Admin-only navigation presentation, filter submission with the UTC label, cursor pagination, detail loading, safe change summaries, actor display including "(account removed)", every timestamp labeled UTC, empty/error/loading states, and accessible status announcements.

### Implementation for User Story 3

- [X] T037 [US3] Implement filtered audit queries and stable keyset pagination in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/AuditEventRepository.cs`; support all filters in `specs/002-audit-system/contracts/audit.openapi.json` and order by `OccurredAtUtc DESC, EventId DESC`.
- [X] T038 [US3] Add the audit-review policy requiring the current session's active Admin role in `iit-academic-portal-service/src/IitAcademicPortal.Api/Authorization/PortalPolicies.cs`; do not authorize from other assigned roles, Angular visibility, or client-submitted role values.
- [X] T039 [US3] Add the read-only collection and detail resources in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/AuditEventsController.cs` with DTOs in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/AuditEventResponse.cs`, matching `specs/002-audit-system/contracts/audit.openapi.json`: validate query ranges, resolve `actorDisplay` for a page with one batched account lookup, record `audit.review.accessed` for each successful read through the security-event path, and expose no create/update/delete route. This task also covers the change-summary and safe-metadata fields of the DTO.
- [X] T040 [P] [US3] Add a timestamp display helper in `iit-academic-portal/src/app/shared/utc-timestamp/utc-timestamp.ts` that renders a timestamp with an explicit "UTC" label inside a `time` element, for use by the audit screens (FR-022).
- [X] T041 [US3] Implement the audit search and detail screens in `iit-academic-portal/src/app/features/audit/audit-page.ts` and `audit-page.html` using Tailwind utilities and the shared component classes from `specs/design-system.md`; add no component stylesheet. Filter inputs for the date range are labeled as UTC and sent as UTC.
- [X] T042 [US3] Register the audit screens under `/admin/audit` in `iit-academic-portal/src/app/app.routes.ts` (give the Admin area child routes) and add the Admin-only "Audit" navigation item in `iit-academic-portal/src/app/layout/shell/shell.ts`. The path under `/admin` lets the existing role-switch landing logic leave the page when the active role changes; the API remains authoritative on every request.

**Checkpoint**: Admin can search and inspect events; every non-Admin active session is denied, including multi-role sessions switched away from Admin; each read is itself recorded.

---

## Phase 6: User Story 4 - Protect Records and Ensure Reliable Recording (Priority: P1)

**Goal**: Prevent unauthorized event mutation, keep required business writes atomic, and preserve and recover security events without changing authentication or authorization outcomes.

**Independent Test**: Inject transaction, outbox, audit-store, and operational-sink failures; verify rollback, unchanged auth results, retained retries, alerting, authorized recovery, and database-level append/read-only grants.

### Tests for User Story 4

- [X] T043 [P] [US4] Add mutation-attempt tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditMutationAttemptTests.cs`: POST, PUT, PATCH, and DELETE on `/api/audit-events` and `/api/audit-events/{eventId}` return 405 with `Allow: GET` for every authenticated session (Admin included), with and without a valid `X-CSRF-Token`, and record `audit.mutation.attempted` with outcome `denied` in both cases; an anonymous request returns 401 and records nothing; exactly one event is recorded per attempt (no additional `access.denied`); no stored event changes.
- [X] T044 [P] [US4] Add recovery and alerting tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditRecoveryAndAlertTests.cs`: the operator recovery script (PostgreSQL-backed) retries or marks only exhausted items and requires an operator ID and reason; counters and the health check report outbox write failure, retry exhaustion, fallback-sink failure, and event-loss risk; no alert path records another audit event.

### Implementation for User Story 4

- [X] T045 [US4] Implement the mutation guard in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/AuditEventsController.cs`: actions for the rejected methods on both audit routes that record `audit.mutation.attempted` through the non-recursive security-event path and return `405 Method Not Allowed` with `Allow: GET` for authenticated sessions; anonymous requests return 401 without an audit event, and no request path touches stored events. Exempt these actions from the global `ValidateAntiforgeryTokenFilter` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Security/ValidateAntiforgeryTokenFilter.cs`; they change nothing, and otherwise an attempt without a CSRF token would be rejected with 400 before it could be recorded.
- [X] T046 [US4] Add the operator recovery script `iit-academic-portal-service/database/postgresql/audit-outbox-recovery.sql` and document its authorized invocation in `iit-academic-portal-service/README.md`: require operator ID and reason, allow only exhausted items to be retried or marked handled, retain the event envelope, and never modify formal audit events.
- [X] T047 [US4] Wire monitoring in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Auditing/SecurityAuditOutboxWorker.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Api/PortalApi.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Api/appsettings.json`: emit the T016 counters, write critical-level structured logs for outbox write failure, retry exhaustion, fallback failure, and event-loss risk, and expose a protected (not anonymous) ASP.NET Core health check that reports exhausted outbox items. Document how a deployment wires alerts to these signals in `iit-academic-portal-service/README.md`.
- [X] T048 [US4] Verify there is no automatic audit deletion or archive job in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/` and document the approved-policy gate for any future lifecycle processing in `iit-academic-portal-service/README.md`; state that retained delivered outbox rows are not formal audit retention.

**Checkpoint**: Mandatory business writes roll back with failed audit persistence; security events survive retry and recovery; audit history is read-only for every role; runtime database credentials cannot mutate formal audit events.

---

## Final Phase: Polish and Cross-Cutting Concerns

**Purpose**: Validate the complete design against the spec, runtime prerequisites, and accessibility expectations.

- [X] T049 [P] Run service tests and build, frontend tests and build, and the PostgreSQL-backed suite with `AUDIT_TEST_POSTGRES` set; record the results, including the migration and grant verification, in `specs/002-audit-system/implementation-notes.md`. Do not treat SQLite alone as proof of PostgreSQL permissions.
- [ ] T050 [P] Complete the accessibility and responsive review for the audit search and detail screens in `iit-academic-portal/src/app/features/audit/` against `specs/design-system.md`, including keyboard use, screen reader, supported breakpoints, and 200% zoom; record findings in `specs/002-audit-system/implementation-notes.md`.
- [X] T051 Verify all API routes, filters, response schemas (including `actorDisplay`), and authorization outcomes match `specs/002-audit-system/contracts/audit.openapi.json`; record any approved differences in `specs/002-audit-system/implementation-notes.md`.
- [X] T052 Measure SC-006: add `iit-academic-portal-service/database/postgresql/seed-audit-volume.sql` (development only, clearly marked) to load the confirmed event volume (1,000,000 unless T002 changed it), run first-page searches, and record the 95th-percentile time and the test environment in `specs/002-audit-system/implementation-notes.md`; add indexes only where the measured query plans justify them.
- [X] T053 Record PostgreSQL migration status, runtime and DDL grants, durable outbox and fallback sink evidence, and every unresolved release gate in `specs/002-audit-system/implementation-notes.md`; confirm US1, US3, and US4 are all complete before declaring the release gate met.
- [ ] T054 Run all scenarios in `specs/002-audit-system/quickstart.md` and record pass/fail evidence in `specs/002-audit-system/implementation-notes.md`; do not claim FR-014 compliance until the durable fallback and alerting are verified. Schedule the SC-005 reviewer usability sessions separately and record their result.

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 Setup**: T001 verifies tooling. T002 records deployment decisions; it gates release and the Production configuration in T019, not the start of Phase 2.
- **Phase 2 Foundational**: Write T003–T009 first, then T010–T021. T010 and T011 precede T012; T012 precedes T013; T014 precedes T015–T017; T017 precedes T018; T019 and T020 can proceed in parallel with T018. This phase blocks all user stories.
- **US1**: T022–T029 depend on Phase 2.
- **US2**: T030–T033 depend on T016 and can proceed in parallel with US1.
- **US3**: T034–T042 depend on T017 and T020; T040 can start at any time; US3 can proceed in parallel with US2.
- **US4**: T043–T048 depend on Phase 2 and on the `AuditEventsController` from T039; T045 shares that file, so schedule it after T039.
- **Polish**: T049–T054 depend on the stories and on the PostgreSQL and operational prerequisites. T052 needs T002's confirmed volume.

### User Story Dependencies

- **US1 (P1)**: Event types are emitted by the existing authentication, session, and recovery flows; business-domain events stay with their owning features.
- **US2 (P1)**: Shares the change-summary builder built in T016.
- **US3 (P1)**: Consumes stored event envelopes and enforces current active-Admin authorization.
- **US4 (P1)**: Its tests validate the shared persistence boundary used by US1.

### Within Each User Story

- Write and run the story's tests before its implementation tasks; keep tests in the existing projects and frameworks.
- Keep business event creation inside the owning feature's transaction; do not make frontend routes a security boundary.
- Keep security-event recording independent of the primary decision and durable across worker restarts; retain exhausted events until authorized recovery.
- Formal audit events remain append-only at both the application API and the runtime database-role boundaries.
- Do not implement retention, archival, or deletion jobs without an approved institutional policy.

### Parallel Opportunities

- T003–T009 are separate test files and can be written in parallel once the stand-in fixtures T003 and T004 are agreed.
- After the foundation, US2 tests and guide, US3 search, controller, and UI work, and US4 failure and permission tests proceed on separate files.
- Within US3, backend query and API tests and Angular component tests can be authored in parallel against `contracts/audit.openapi.json`.

## Parallel Example: User Story 3

```text
Task T034: backend Admin-session authorization tests in tests/IitAcademicPortal.Api.Tests/Auditing/AuditReviewAuthorizationTests.cs
Task T035: backend filter/keyset pagination tests in tests/IitAcademicPortal.Api.Tests/Auditing/AuditEventSearchTests.cs
Task T036: Angular audit-page tests in src/app/features/audit/audit-page.spec.ts
Task T040: UTC timestamp display helper in src/app/shared/utc-timestamp/utc-timestamp.ts

After the tests establish expected behavior:
Task T037: persistence query/filter/keyset implementation in src/IitAcademicPortal.Infrastructure/Persistence/AuditEventRepository.cs
Task T038: active-Admin server policy in src/IitAcademicPortal.Api/Authorization/PortalPolicies.cs
Task T039: read-only controller and DTOs in src/IitAcademicPortal.Api/Controllers/AuditEventsController.cs
Task T041: Angular Admin search/detail UI in src/app/features/audit/
```

## Implementation Strategy

### Demonstration slice (User Story 1 only)

1. Complete Setup and Phase 2, including the PostgreSQL fixture and the development file sink.
2. Complete US1 for the shared recorder and the initial security event catalog.
3. Validate with quickstart scenarios 3 and 4. This slice is for demonstration only. It must not be released, because review, the append-only database boundary, and recovery are not yet in place.

### Release (User Stories 1, 3, and 4 together)

1. Complete the foundational phase and the release prerequisites from T002: the production durable sink, the alert destination, the recovery owner, and the confirmed data volume.
2. Complete US1, US3, and US4. Verify the runtime database role, the outbox fallback, and recovery with PostgreSQL.
3. Complete US2 alongside, then the Polish phase. Do not release if the durable fallback sink is not verified.

### Incremental Delivery

1. Complete Setup and Foundational gates.
2. Deliver US1 event recording and the initial security-event catalog.
3. Add US2 allowlisted change summaries, redaction, and the owner guide.
4. Add US3 Admin-only search and detail with accessible Angular presentation.
5. Complete US4 mutation protection, recovery, monitoring, and retention protections.
6. Complete the PostgreSQL, accessibility, performance, contract, and manual quickstart gates.

### Parallel Team Strategy

After Phase 2, assign US1 security-event integrations, US2 change-summary handling, US3 review, and US4 hardening to separate workers against the event contract and database model. The durable fallback sink, the database account split, and shared outbox behavior are blocking foundational dependencies. The US3 and US4 workers must coordinate on `AuditEventsController.cs`.
