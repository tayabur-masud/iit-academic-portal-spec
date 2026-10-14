# Tasks: Teacher Management

**Input**: Design documents from `/specs/004-teacher-management/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: Include automated tests required by the project constitution for identity, authorization, audit, lifecycle, and data-integrity behavior. Email-delivery and email-specific tests are explicitly deferred until the shared email mechanism is ready.

**Organization**: Tasks follow the three P1 user stories. Pending-account authentication and audit/session infrastructure are foundational prerequisites.

## Phase 1: Setup

**Purpose**: Add the Teacher Management feature slice to the existing service, frontend, and test projects without new packages or projects.

- [X] T001 Create feature directories under `iit-academic-portal-service/src/IitAcademicPortal.Domain/Teachers/`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/`, `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Teachers/`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/`, and `iit-academic-portal/src/app/features/teachers/`.

---

## Phase 2: Foundational

**Purpose**: Establish user status, authentication enforcement, session revocation, and teacher audit definitions before story work.

- [X] T002 Add `TeacherAccountStatus` with exactly `PendingCredentialSetup`, `Active`, and `Deactivated`, add `AccountStatus` to `PortalUser`, and configure EF to default existing accounts to `Active` in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Identity/TeacherAccountStatus.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Domain/Identity/PortalUser.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs`; create the migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`.
- [X] T003 Reject accounts whose status is not `Active` during password sign-in and validated-session lookup in `iit-academic-portal-service/src/IitAcademicPortal.Application/Authentication/PortalAuthenticationService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/AuthSessionRepository.cs`; preserve existing generic sign-in failure behavior.
- [X] T004 [P] Add bulk active-session revocation for a user and a teacher-deactivation revocation reason in `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IAuthSessionRepository.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/AuthSessionRepository.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Domain/Sessions/SessionRevocationReason.cs`.
- [X] T005 [P] Define allowlisted Teacher business audit events for profile creation/update/deactivation, role changes, and Coordinator synchronization in `iit-academic-portal-service/src/IitAcademicPortal.Application/Auditing/TeacherAuditEventDefinitions.cs`; exclude passwords, setup proofs, and other secrets.

**Checkpoint**: Pending/deactivated authentication behavior, bulk revocation, and audit event definitions are ready before teacher operations are implemented.

---

## Phase 3: User Story 1 - Add a Teacher (Priority: P1)

**Goal**: An active Admin creates one TeacherProfile and one linked Identity user with Teacher role/default; the account remains pending and no email is sent.

**Independent Test**: Create a teacher with required display name and unique email; verify exactly one profile/account, Teacher role/default, PendingCredentialSetup status, no email attempt, duplicate email/staff ID rejection, and no partial writes on audit failure.

### Tests for User Story 1

- [X] T006 [P] [US1] Add API tests for teacher provisioning, duplicate linked email, duplicate supplied staff ID, pending status, no email attempt, and audit rollback in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherProvisioningTests.cs`.
- [X] T007 [P] [US1] Add Angular tests for required display name/email, optional employee/staff ID/phone/Designation, validation errors, pending-account confirmation, and Admin-only menu/route access in `iit-academic-portal/src/app/features/teachers/teacher-form.spec.ts`, `iit-academic-portal/src/app/layout/shell/shell.spec.ts`, and `iit-academic-portal/src/app/app.routes.spec.ts`.

### Implementation for User Story 1

- [X] T008 [P] [US1] Add `TeacherProfile` with required display name and Identity user link, optional employee/staff ID, phone, and Designation; enforce one profile per user, globally unique normalized staff ID when supplied (including inactive profiles), and restrictive Designation reference in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Teachers/TeacherProfile.cs`. Quote the data-model constraint: maximum string lengths are bounded consistently with existing Identity/configuration conventions; no new business-specific maximum is specified.
- [X] T009 [US1] Map `TeacherProfile` in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs` with unique `UserId`, unique non-null normalized staff ID, and restrictive foreign keys; create a versioned EF migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`.
- [X] T010 [US1] Implement provisioning in `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherProvisioningService.cs`: create a user with required unique email, `AccountStatus = PendingCredentialSetup`, Teacher role, and Teacher default; create its profile in the same explicit transaction, stage audit, send no email, and provide no password bypass.
- [X] T011 [US1] Add create and safe detail DTOs in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/TeacherContracts.cs` and expose `POST /api/admin/teachers` plus `GET /api/admin/teachers/{teacherId}` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Teachers/TeachersController.cs`; require active Admin and return no credential secrets.
- [X] T012 [US1] Add the Teacher Management entry to Admin-only navigation and register the active-Admin route, then add the typed teacher API client and create form in `iit-academic-portal/src/app/layout/shell/shell.ts`, `iit-academic-portal/src/app/app.routes.ts`, `iit-academic-portal/src/app/features/teachers/teacher-api.ts`, `iit-academic-portal/src/app/features/teachers/teacher-form.ts`, and `iit-academic-portal/src/app/features/teachers/teacher-form.html`; create the pending account without calling email transport.

**Checkpoint**: Admin can create and inspect a pending Teacher account safely; it cannot sign in before future credential-setup work.

---

## Phase 4: User Story 2 - Maintain Teacher Information and Roles (Priority: P1)

**Goal**: Admin maintains teacher details and permitted roles; Coordinator is synchronized from batch-coordinator assignments.

**Dependencies**: Requires US1’s TeacherProfile/account linkage and foundational account status. The batch-assignment producer does not exist and remains an integration dependency.

**Independent Test**: Update profile fields, assign/remove Admin without removing Teacher, reject Student assignment, preserve a valid default role, block removal of the last active Admin, and verify the role synchronizer adds/retains/removes Coordinator based on authoritative active assignment state.

### Tests for User Story 2

- [X] T013 [P] [US2] Add API tests for role policy, default-role validity, Student-role rejection, Admin assignment/removal, and last-Admin conflict in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherRoleTests.cs`.
- [X] T014 [P] [US2] Add unit tests for Coordinator synchronization true/false state, multiple assignments, idempotency, default-role fallback, and audit rollback in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherCoordinatorRoleSynchronizerTests.cs`.

### Implementation for User Story 2

- [X] T015 [US2] Implement teacher profile update and role application services in `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherProfileService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherRoleService.cs`; allow Admin assignment/removal, prohibit Student, retain Teacher for active profiles, reconcile the default role, and perform last-Admin validation within the transaction.
- [X] T016 [US2] Add update and role DTOs in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/TeacherContracts.cs` and expose `PUT /api/admin/teachers/{teacherId}` plus `PUT /api/admin/teachers/{teacherId}/roles` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Teachers/TeachersController.cs` with field-keyed errors and active-Admin authorization.
- [X] T017 [US2] Implement the internal `ITeacherCoordinatorRoleSynchronizer` and idempotent application service in `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/ITeacherCoordinatorRoleSynchronizer.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherCoordinatorRoleSynchronizer.cs`; consume authoritative `hasActiveCoordinatorAssignments`, keep Teacher, prohibit Student, and do not expose a manual Coordinator role endpoint.
- [X] T018 [US2] Add Angular teacher profile editing and role management in `iit-academic-portal/src/app/features/teachers/teacher-form.ts` and `iit-academic-portal/src/app/features/teachers/teacher-form.html`; display Teacher as required, Admin as Admin-managed, Student as unavailable, and Coordinator as assignment-derived.

**Checkpoint**: Profile and role changes are independently usable; role invariants and the synchronizer contract are covered. Full batch-assignment trigger integration waits for the future assignment owner.

---

## Phase 5: User Story 3 - Find and Deactivate a Teacher (Priority: P1)

**Goal**: Admin searches the directory and deactivates a teacher while disabling the account, revoking sessions, and preserving records/audit history.

**Dependencies**: Requires US1’s TeacherProfile and foundational account-status/session-revocation changes.

**Independent Test**: Search/filter/paginate teacher records; deactivate one; verify account status Deactivated, all active sessions revoked, profile and audit history retained, sign-in and existing sessions denied, and last-Admin protection enforced.

### Tests for User Story 3

- [X] T019 [P] [US3] Add API tests for directory search/filter/pagination, active/pending/deactivated states, deactivation session revocation, audit atomicity, and last-Admin protection in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherDirectoryAndDeactivationTests.cs`.
- [X] T020 [P] [US3] Add Angular tests for teacher directory loading, empty/error states, search/filter/pagination, role/status display, and deactivation confirmation in `iit-academic-portal/src/app/features/teachers/teacher-list.spec.ts`.

### Implementation for User Story 3

- [X] T021 [US3] Implement bounded teacher directory queries and search/filter/status behavior in `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherDirectoryService.cs`; include only required profile and safe account/role fields.
- [X] T022 [US3] Implement deactivation in `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherLifecycleService.cs`: set profile inactive and account Deactivated, bulk-revoke all active sessions, preserve Identity role/profile/history, stage audit, and commit all changes in one transaction.
- [X] T023 [US3] Add list/filter and deactivation routes in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Teachers/TeacherDirectoryController.cs` and corresponding DTOs in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/TeacherContracts.cs`; do not expose physical DELETE.
- [X] T024 [US3] Implement the Admin teacher directory, status badges, search/filter/pagination, and confirm-deactivation flow in `iit-academic-portal/src/app/features/teachers/teacher-list.ts` and `iit-academic-portal/src/app/features/teachers/teacher-list.html` using shared design-system classes.

**Checkpoint**: Admin can find and deactivate teachers; inactive accounts and every existing session lose access while historical records remain.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verify database-provider behavior and update the runnable feature guide, including the explicit Coordinator-producer integration gate.

- [X] T025 [P] Add PostgreSQL 18 integration tests for existing-user status backfill to Active, pending/deactivated authentication rejection, one-to-one TeacherProfile, employee/staff-ID uniqueness across active/inactive profiles, restrictive Designation reference, transaction behavior, and concurrent last-Admin protection in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherManagementPostgresTests.cs`.
- [X] T026 [P] Update `specs/004-teacher-management/quickstart.md` with the implemented API/UI paths, migration command, pending-account expectations, Admin role checks, deactivation/session behavior, and the Coordinator integration gate: the future assignment owner must call the synchronizer in the assignment transaction and add end-to-end producer tests before claiming automatic synchronization is complete.
- [X] T027 [P] Add an API authorization matrix in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherManagementAuthorizationTests.cs`: verify anonymous requests receive 401 and Student, Teacher, and Coordinator active roles receive 403 for list, detail, create, update, role-change, and deactivate routes; verify denied writes change no profiles, users, roles, sessions, or audit events.
- [X] T028 Run `dotnet test` from `iit-academic-portal-service/`, then run `npm test -- --watch=false` and `npm run build` from `iit-academic-portal/`; record outcomes and any blocked external integration checks in `specs/004-teacher-management/quickstart.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No new packages or projects; create feature directories.
- **Foundational (Phase 2)**: Account status, authentication/session enforcement, bulk revocation, and audit definitions block all stories.
- **US1 (Phase 3)**: Starts after foundational work; creates the TeacherProfile/user lifecycle.
- **US2 (Phase 4)**: Depends on US1’s TeacherProfile/account linkage. This feature tests the synchronizer directly; the future assignment owner must call it before committing every assignment change and add end-to-end producer tests. Until then, do not claim portal-wide automatic synchronization is complete.
- **US3 (Phase 5)**: Depends on US1’s TeacherProfile and foundational account/session status. US2 and US3 can proceed in parallel after US1 if profile/role services and lifecycle/directory services remain in separate files.
- **Polish (Phase 6)**: Depends on all stories. T027 depends on routes from T011, T016, and T023; T028 depends on all implementation/test/documentation tasks. Email delivery and its end-to-end tests are explicitly outside this feature. Coordinator producer integration is a gate for the future assignment-owning feature, which must test the synchronizer call and assignment mutation in one unit of work.

### User Story Dependencies

- **US1 (P1)**: Independent after foundation; provides teacher provisioning and pending-account behavior.
- **US2 (P1)**: Requires US1 profile/account linkage; role changes and sync contract are otherwise testable independently.
- **US3 (P1)**: Requires US1 profile/account linkage and foundation session revocation.

### Parallel Opportunities

- T004 and T005 can proceed in parallel after T002; T003 and T004 touch separate authentication/session files after T002.
- Within each story, API test authoring and Angular test authoring can proceed in parallel because they use separate projects/files.
- US2 and US3 can proceed in parallel after US1 if the role service/controller and directory/lifecycle service/controller remain separately owned.
- T025 PostgreSQL tests, T026 quickstart review, and T027 authorization-matrix tests can proceed in parallel after all stories; T028 runs the final validation after them.

## Parallel Example: User Stories 2 and 3

```text
After US1 is complete:
- Worker A: US2 profile/role update service, roles endpoint, and Coordinator sync contract (T013-T018)
- Worker B: US3 directory query, deactivation lifecycle service, directory endpoint, and UI (T019-T024)
Coordinate shared PortalDbContext and TeacherContracts edits before integrating the work.
```

## Implementation Strategy

### MVP First (User Story 1)

1. Complete Setup and Foundational tasks.
2. Complete US1: provision a TeacherProfile and linked pending Identity user with Teacher role/default, without email delivery.
3. Validate duplicate handling, atomic profile/user/audit creation, and pending-account sign-in denial independently.
4. Do not claim teachers can sign in until the separate email/credential-setup work activates their account.

### Incremental Delivery

1. Deliver US1 provisioning and pending-account boundary.
2. Add US2 profile/role maintenance and the internal Coordinator synchronizer contract.
3. Add US3 directory search and safe deactivation/session revocation.
4. Verify PostgreSQL and frontend/service gates; document the deferred email and batch-assignment producer integrations.

### Format Validation

Every task line starts `- [ ] T###` or `- [X] T###`; `[P]` appears only for independent work, and every user-story task has `[US1]`, `[US2]`, or `[US3]`. Setup, Foundational, and Polish tasks have no story label. IDs are sequential and every task description contains an exact file path.

## Phase 7: Convergence

- [X] T029 Extend the API authorization matrix to verify profile/account state, roles, and target-user sessions stay unchanged after denied requests; assert no business audit events are added and authenticated 403s produce the expected `access.denied` security events in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherManagementAuthorizationTests.cs` per FR-007/SC-003/T027 (partial).
- [X] T030 Add PostgreSQL-backed concurrent last-Admin tests that invoke the actual role-removal and deactivation application services, rather than duplicating the lock/check/write sequence; assert one conflicting operation is rejected and an active Admin remains in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherManagementPostgresTests.cs` per FR-014/plan-concurrency/T025 (partial).
- [X] T031 Add a failed-audit test through `TeacherProfileService.UpdateAsync` proving profile fields and Identity email/username remain unchanged and no `teacher.updated` event persists when audit staging fails in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/TeacherManagement/TeacherProfileTests.cs` per FR-009/T015 (partial).
