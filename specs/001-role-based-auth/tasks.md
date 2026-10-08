---

description: "Implementation tasks for role-based authentication and access control"
---

# Tasks: Role-Based Authentication and Access Control

**Input**: Design documents from `specs/001-role-based-auth/`

**Prerequisites**: `spec.md`, `plan.md`, `research.md`, `data-model.md`, `contracts/authentication.openapi.json`, and `quickstart.md`

**Tests**: Automated tests are included because the constitution requires authentication and authorization behavior to be tested. Place tests in the existing test projects discovered in the corresponding submodules; do not add a new test framework without approval.

**Organization**: Tasks are grouped by user story. Setup tasks first confirm the real submodule source paths; the current planning checkout contains README-only submodules.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Parallelizable after listed prerequisites because it uses a separate repository or file set.
- **[Story]**: User-story label matching `spec.md` (`US1`–`US4`).
- Every implementation task identifies the existing repository root or design artifact. Internal source/test paths MUST be discovered from the populated submodule in Setup; do not invent a new folder layout.

## Phase 1: Setup

**Purpose**: Confirm the existing application repositories, exact source/test paths, and pinned toolchain before implementation.

- [X] T001 [P] Inspect `iit-academic-portal/` for the Angular 21 workspace, Node.js 24 configuration, authentication/route entry points, styling tokens, and existing test projects; record the exact discovered paths in the task execution notes, and if source is absent request the approved source revision instead of scaffolding a replacement.
- [X] T002 [P] Inspect `iit-academic-portal-service/` for the ASP.NET Core 10 solution/projects, Identity configuration, EF Core 10 DbContext/migrations, email delivery service, and existing test projects; record exact discovered paths and request the approved source revision if implementation files are absent.
- [X] T003 [P] Verify the submodule toolchain and dependency declarations match Angular 21/Node.js 24 LTS and .NET 10/C# 14/Identity 10/EF Core 10/Npgsql EF provider 10.x; record any mismatch without changing a major version in `iit-academic-portal/` or `iit-academic-portal-service/`.

### Phase 1 Execution Notes (2026-10-08)

- **Starting state**: Both submodules contained only `README.md`. `iit-academic-portal-service` pointed at the frontend repository URL. The project owner approved (a) pointing it at `tayabur-masud/iit-academic-portal-service.git` and (b) scaffolding new projects in place of the "do not scaffold" instruction above.
- **T001 — frontend**: Angular 21.2 CLI workspace in `iit-academic-portal/`. Sources are in `src/app` (`core/auth`, `features/auth`, `layout/shell`, `pages`), tokens in `src/styles/tokens.css`, and Vitest specs sit beside the sources (`npm test`).
- **T002 — service**: `iit-academic-portal-service/IitAcademicPortal.slnx` with `src/IitAcademicPortal.{Domain,Application,Infrastructure,Api}` and the xUnit test project `tests/IitAcademicPortal.Api.Tests` (`dotnet test`). Identity, the DbContext, and migrations are in Infrastructure.
- **T003 — toolchain**: Node.js 24.19, Angular 21.2, .NET SDK 10.0.401, ASP.NET Core / Identity / EF Core 10.0.12, Npgsql EF provider 10.0.3. No major-version mismatch. `psql` and a Docker daemon are not available locally.
- Results for T026–T030 and implementation decisions: [implementation-notes.md](implementation-notes.md).

---

## Phase 2: Foundational

**Purpose**: Establish shared server-side identity/session support and safe request behavior required by all user stories.

**Checkpoint**: Do not start user-story implementation until source paths are confirmed, current Identity data is inspected, and the per-session authorization boundary is agreed with the design artifacts.

- [X] T004 Inspect the existing Identity schema and migration history in `iit-academic-portal-service/`; preserve “Normalized email: Unique across accounts so sign-in resolves to one account” and “Each user-role pair MUST be unique”; check for duplicate normalized email values before adding a constraint, and reuse existing Identity tables rather than recreating them.
- [X] T005 Add or adapt the minimal per-session persistence model in the existing service data layer in `iit-academic-portal-service/`; preserve these `data-model.md` constraints verbatim: “Session ID: Unique; raw bearer material is not stored in the database”; “Session handle digest: Unique; compare in constant time where applicable”; “User ID: Required; references one user account”; “Active role: Nullable only while a multi-role user is choosing a role; when set, it MUST be assigned to the user”; “Created at: Required for security diagnosis; not used as an automatic expiry cutoff”; “Revoked at: Null while active”; and “Revocation reason: Logout, password reset of this session, or another approved security action; Must not contain credentials or reset proofs”.
- [X] T006 Create a version-controlled EF Core 10 migration in the existing migrations path under `iit-academic-portal-service/` only if T004 finds that the per-session state is missing; include required user/role references, unique constraints, and revocation fields without duplicating Identity schema.
- [X] T007 Integrate protected session-cookie validation with the existing Identity/service authentication boundary in `iit-academic-portal-service/`; keep active role per session, validate that it remains assigned on protected requests, implement explicit logout/revocation, and do not add idle or maximum-age expiration.
- [X] T008 Configure the existing service request protections and error/logging conventions in `iit-academic-portal-service/`; require anti-forgery protection for cookie-authenticated state changes, return consistent user-safe errors, and exclude passwords, cookie values, session handles, and reset proofs from logs.

---

## Phase 3: User Story 1 - Sign In and Sign Out (Priority: P1) - MVP

**Goal**: A user signs in with email/password, reaches an assigned role context, and can explicitly end the current session.

**Independent Test**: Valid email/password creates a protected session; invalid and unknown-email attempts have the same generic outcome; logout revokes the current session and protected requests then require authentication.

### Tests for User Story 1

- [X] T009 [P] [US1] Add service API tests in the existing test project under `iit-academic-portal-service/` for valid login, wrong password, unknown email, generic failure equivalence, cookie/session creation, and no automatic session timeout using disposable accounts.
- [X] T010 [P] [US1] Add Angular tests in the existing test project under `iit-academic-portal/` for required email/password fields, generic login failure, loading feedback, duplicate-submit prevention, and explicit logout visibility.

### Implementation for User Story 1

- [X] T011 [US1] Implement email/password sign-in and current-session retrieval in the existing authentication/application/API files under `iit-academic-portal-service/`, matching `contracts/authentication.openapi.json` and returning no account-existence distinction for invalid credentials.
- [X] T012 [US1] Implement the login form and sign-in/loading/validation/error states in the existing Angular source files under `iit-academic-portal/`, using `specs/design-system.md` tokens and preserving the email but not the password after a safe validation failure.
- [X] T013 [US1] Implement current-session logout and the explicit sign-out action in the existing service and Angular source files under `iit-academic-portal-service/` and `iit-academic-portal/`; confirm logout revokes only the current session and protected navigation/API access is rejected afterward.

**Checkpoint**: US1 works independently for valid/invalid credentials and explicit logout; the session remains valid until explicit logout/revocation.

---

## Phase 4: User Story 2 - Access Only Authorized Areas (Priority: P1)

**Goal**: Each protected action is authorized according to the approved role and record boundaries, even when requested directly rather than through visible navigation.

**Independent Test**: Run allowed and denied access checks for Admin Module scope, Student-owned records, Teacher-assigned courses/assessment components, and Coordinator-assigned batches/results.

### Tests for User Story 2

- [X] T014 [P] [US2] Add service authorization tests in the existing test project under `iit-academic-portal-service/` for Admin, Student-own-record, Teacher-assigned-course/component, Coordinator-assigned-batch/result boundaries, removed role assignments, direct unauthorized requests, and no protected data in denial responses.

### Implementation for User Story 2

- [X] T015 [US2] Implement server-side role and record-level authorization in existing application/API/data-access files under `iit-academic-portal-service/`, matching the feature-list boundaries in `specs/001-role-based-auth/spec.md`; do not treat hidden Angular navigation or a client-submitted role as authorization.
- [X] T016 [US2] Implement the existing Angular route/navigation presentation for unauthorized and not-found states in `iit-academic-portal/`; filter visible destinations by the active role while leaving the service authoritative for access decisions.

**Checkpoint**: US2 passes direct service/API checks for each role boundary, including denial after a role assignment is removed.

---

## Phase 5: User Story 4 - Select and Switch Assigned Roles (Priority: P1)

**Goal**: A multi-role user chooses and switches between currently assigned roles without combining their permissions or changing another session's active role.

**Independent Test**: A single-role account enters its sole role automatically; a multi-role account can select and switch only assigned roles, sees the active role, and receives an authorized landing page when the current route is unavailable.

### Tests for User Story 4

- [X] T017 [P] [US4] Add service/API tests in the existing test project under `iit-academic-portal-service/` for assigned-role selection, unassigned-role rejection, active-role isolation across two sessions, removed-role denial, and route-context response.
- [X] T018 [P] [US4] Add Angular tests in the existing test project under `iit-academic-portal/` for single-role bypass, multi-role choices limited to assigned roles, visible active-role state, switching feedback, and unavailable-route handling.

### Implementation for User Story 4

- [X] T019 [US4] Implement session-scoped role selection/switching in the existing service/application/API files under `iit-academic-portal-service/`, matching `PUT /api/auth/sessions/current/active-role` in `specs/001-role-based-auth/contracts/authentication.openapi.json` and validating the role against current assignments.
- [X] T020 [US4] Implement role selection and role-switch controls in the existing Angular shell/navigation files under `iit-academic-portal/`, using `specs/design-system.md` patterns and refreshing role-specific state after a server-validated switch.
- [X] T021 [US4] Implement post-switch landing behavior in the existing Angular routing files under `iit-academic-portal/`; navigate to a target-role-authorized landing page when the current location is unavailable and never retain protected content from the prior role.

**Checkpoint**: US4 can be demonstrated with a Teacher+Coordinator account and two independent sessions without role or session context leakage.

---

## Phase 6: User Story 3 - Recover and Reset a Password (Priority: P2)

**Goal**: A user recovers access through the registered email without account enumeration, then resets a policy-compliant password using a valid one-use proof.

**Independent Test**: Known and unknown email requests have the same public response; valid proof accepts a password meeting the exact 8-character/letter/number rule; invalid/expired/replayed proofs fail; only the current reset session is revoked.

### Tests for User Story 3

- [X] T022 [P] [US3] Add service tests in the existing test project under `iit-academic-portal-service/` for identical known/unknown email responses, registered-email delivery, minimum eight characters plus at least one letter and one number, expired/replayed recovery proof, and current-session-only revocation with another session remaining active.
- [X] T023 [P] [US3] Add Angular tests in the existing test project under `iit-academic-portal/` for recovery/reset required fields, password-rule feedback, loading/success/error states, and no account-existence disclosure.

### Implementation for User Story 3

- [X] T024 [US3] Implement Identity-backed recovery request and reset in existing service authentication/email files under `iit-academic-portal-service/`; send proof to the email recorded for the account, return the same public response for known/unknown email, enforce the stated password rule, revoke only the session used for reset, and preserve the data-model rule that recovery proof is short-lived and single-use.
- [X] T025 [US3] Implement forgot-password and password-reset views in the existing Angular source files under `iit-academic-portal/`, following `specs/design-system.md` and never displaying raw recovery proofs or account-existence distinctions.

**Checkpoint**: US3 passes recovery enumeration, proof validation, password policy, and session-revocation cases independently.

---

## Final Phase: Polish and Cross-Cutting Concerns

**Purpose**: Verify the complete feature against its approved spec, contract, design system, and accepted security risk.

- [X] T026 [P] Run the existing Angular 21/Node.js 24 and .NET 10 test/build commands discovered in T001-T003 and record results against `specs/001-role-based-auth/quickstart.md`.
- [X] T027 [P] Review login, recovery, reset, role selection/switching, unauthorized, and not-found states against WCAG 2.2 AA and `specs/design-system.md` in `iit-academic-portal/`.
- [X] T028 Review session lifetime, persistent-cookie/CSRF protections, current-role checks, reset revocation, and secret-free logging in `iit-academic-portal-service/`; explicitly document the accepted no-timeout residual risk before release.
- [X] T029 Compare implemented routes, request/response schemas, and error behavior with `specs/001-role-based-auth/contracts/authentication.openapi.json`; update the contract or implementation owner’s documentation for approved differences.
- [X] T030 Confirm PostgreSQL 18/Npgsql EF Core 10 compatibility, migration behavior, and normalized-email uniqueness using the actual service schema in `iit-academic-portal-service/`; document whether the session-state migration was applied.
  - **Done (2026-10-08)**: The migration is applied to the local PostgreSQL 18.3 database. A unique `EmailIndex` is on `NormalizedEmail`. Sign-in has created session rows. See [implementation-notes.md](implementation-notes.md#database-t030).

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 Setup**: T001-T003 can run in parallel; source path discovery gates tasks that edit submodules.
- **Phase 2 Foundational**: Depends on source readiness from Phase 1. T004 precedes T005; T005 precedes T006-T007. T008 can proceed after T002 and the existing service request pipeline are identified. This phase blocks all user stories.
- **User Stories**: US1, US2, and US3 can proceed after Phase 2; US4 depends on US1 session creation and US2 authorization boundaries. Execute stories in listed priority order when capacity is limited.
- **Final Polish**: Depends on the desired user stories and their acceptance checkpoints.

### User Story Dependencies

- **US1 (P1)**: Starts after Phase 2; no dependency on another story.
- **US2 (P1)**: Starts after Phase 2; depends on the authenticated session foundation but is independently testable for role boundaries.
- **US4 (P1)**: Starts after Phase 2 and the core session behavior in US1 and role boundaries in US2; it must not union roles.
- **US3 (P2)**: Starts after Phase 2; can proceed independently of US2/US4, while sharing Identity and session primitives.

### Within Each User Story

- Write the constitution-required security tests before completing the corresponding implementation tasks; use the existing test projects found in Setup.
- Service authorization/session state is authoritative; Angular routes and hidden controls are presentation only.
- Keep implementation inside the existing submodules and actual paths discovered by T001/T002; do not scaffold a parallel application structure.
- Re-run each story's independent test criteria before its checkpoint.

### Parallel Opportunities

- T001/T002/T003 are independent setup investigations across the frontend, service, and pinned toolchain.
- Within US1, frontend form tests and service API tests (T009/T010) can proceed in parallel; server and frontend implementation work (T011/T012) can proceed in parallel once the contract is agreed.
- Within US2, service authorization tests and Angular unauthorized-state tests can proceed in parallel before their implementations.
- Within US4, session API tests and frontend role-picker tests can proceed in parallel; role switching implementation requires the validated session foundation.
- Within US3, backend recovery tests and frontend recovery/reset tests can proceed in parallel; implementation must preserve identical public response behavior.
- US3 can run in parallel with US2/US4 after Phase 2 if staffing permits; US4 should wait for US1/US2 role/session contracts.

## Parallel Example: User Story 1

```text
Task T009: service sign-in/session API tests in the existing test project under iit-academic-portal-service/
Task T010: Angular login/loading-state tests in the existing test project under iit-academic-portal/

After both test tasks establish expected behavior:
Task T011: email/password sign-in and current-session API in the existing service source under iit-academic-portal-service/
Task T012: login form and validation/loading states in the existing Angular source under iit-academic-portal/
```

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 source/version discovery and obtain the approved source contents if the current submodule checkouts remain README-only.
2. Complete Phase 2 foundational Identity/session/error protections.
3. Complete Phase 3 (US1) sign-in and logout.
4. Validate US1 independently using `specs/001-role-based-auth/quickstart.md`.
5. Do not release the MVP without logout, session security, and server-side protection; sessions intentionally have no automatic expiry.

### Incremental Delivery

1. Finish setup and foundational session/auth support.
2. Deliver US1 login/logout as the first demonstrable increment.
3. Add US2 role and record authorization; validate each role boundary.
4. Add US4 role selection/switching after US1 and US2 foundations.
5. Add US3 password recovery/reset; validate generic response and one-session-only revocation.
6. Complete the security, accessibility, contract, migration, and quickstart checks in the final phase.

### Parallel Team Strategy

After Phase 2, assign US1 and US2 to separate workers against the agreed contract; US3 may proceed in parallel once Identity/email foundations are ready. Start US4 only after the shared authenticated-session and authorization boundaries are stable.
