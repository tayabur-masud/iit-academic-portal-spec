---

description: "Task list for feature implementation"

---

# Tasks: User Management

**Input**: Design documents from `/specs/006-user-management/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`

**Tests**: Included because the constitution and implementation plan require automated coverage for business-critical rules, authorization, transaction integrity, and credential security.

**Organization**: Tasks are grouped by the three P1 user stories. Shared notification and API contracts are established before story work.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel because it touches separate files and has no dependency on incomplete tasks.
- **[Story]**: Story traceability label; setup, foundational, and polish tasks have no story label.
- Every task names the implementation or test file path it changes.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Add the common setup-email notification definition and its versioned database seed.

- [X] T001 Add the `AccountCredentialSetup` event and initial template/revision in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/NotificationTemplateCatalog.cs`; generate the EF migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/` and update `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/PortalDbContextModelSnapshot.cs` without adding account tables or columns.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Define the shared account API types needed by directory, provisioning, and maintenance stories.

- [X] T002 Add explicit Admin account request/response DTOs and boundary validation for account identity, status, roles, default role, paging, and concurrency in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/UserManagementContracts.cs`; exclude passwords, hashes, setup proofs, and setup links from these account DTOs and all responses.

**Checkpoint**: Shared notification definition and API contracts are ready; story work can proceed against existing Identity, Admin authorization, audit, and session services.

---

## Phase 3: User Story 1 - Find and Review Portal Accounts (Priority: P1)

**Goal**: Let an active Admin find and inspect bounded account projections across Admin, Student, Teacher, and Coordinator roles without exposing secrets.

**Independent Test**: Search a fixture set by normalized email and stable user ID with role/status filters; verify stable bounded ordering and detail projection, and verify anonymous/non-Admin requests return no account data.

### Tests for User Story 1

- [X] T003 [P] [US1] Add API tests for active-Admin authorization, anonymous/non-Admin denial, email/ID search, role/status filtering, bounded page sizes, stable ordering, and secret-free account details in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/UserDirectoryTests.cs`.
- [X] T004 [P] [US1] Add Angular HTTP and directory-view tests for query parameters, incremental bounded loading, empty/error states, secret-free rendering, keyboard navigation, accessible names/announcements, and supported responsive viewports in `iit-academic-portal/src/app/features/users/user-list.spec.ts`.

### Implementation for User Story 1

- [X] T005 [US1] Implement a read-only account directory projection in `iit-academic-portal-service/src/IitAcademicPortal.Application/UserManagement/UserDirectoryService.cs` with email-or-ID search, role/status filters, page size default 25 and maximum 100, normalized-email/user-ID ordering, and no feature-profile mutation.
- [X] T006 [US1] Add `GET /api/admin/users` and `GET /api/admin/users/{userId}` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/UserManagementController.cs`, enforce the active-Admin policy server-side, and map only the fields defined by `specs/006-user-management/contracts/user-management-api.md`.
- [X] T007 [P] [US1] Add typed account models and `HttpClient` methods for bounded list/detail requests in `iit-academic-portal/src/app/features/users/user.models.ts` and `iit-academic-portal/src/app/features/users/user-api.ts`.
- [X] T008 [US1] Build the lazy-loaded `/admin/users` directory and account detail view with search, role/status filters, shared incremental loading, and loading/empty/error states; reuse IIT design-system controls/tokens and verify responsive, keyboard-accessible interaction in `iit-academic-portal/src/app/features/users/user-list.ts`, `iit-academic-portal/src/app/features/users/user-list.html`, and `iit-academic-portal/src/app/app.routes.ts`.

**Checkpoint**: An Admin can independently find and inspect all supported account types; unauthorized callers receive no account details.

---

## Phase 4: User Story 2 - Provision a Portal Account (Priority: P1)

**Goal**: Provision generic Admin and Student accounts in pending state, use one secure setup-email flow for every supported role owner, and activate an account only after valid password setup.

**Independent Test**: As an active Admin, create unique-email Admin and Student accounts and verify pending status, one allowed/default role, no password, audit, and one post-commit setup notification; reject duplicates without partial writes. Complete setup with a valid purpose-specific proof, reject expired/replayed/reset-purpose proofs, and verify pending-only fresh-link reissue and delivery-failure behavior. Teacher provisioning must use its existing owner; a newly provisioned Coordinator must use its batch-assignment owner, and each owner must request the same setup event after commit. User Management cannot directly create or assign Coordinator accounts.

### Tests for User Story 2

- [X] T009 [P] [US2] Add provisioning API tests for active-Admin authorization, anonymous/non-Admin create denial, Admin/Student creation, normalized-email duplicates, unsupported Teacher/Coordinator direct creation, pending/no-password state, role/default-role invariants, post-commit notification request, and atomic account/audit failure in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/UserProvisioningTests.cs`.
- [X] T010 [P] [US2] Add setup-token tests for purpose separation from password reset, public completion with valid proof, active-Admin-only reissue and anonymous/non-Admin denial, pending-only completion/reissue, password-policy failures, expiry/replay, security-stamp invalidation, safe audit, and delivery failure leaving the account pending in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/AccountCredentialSetupTests.cs`.
- [X] T011 [P] [US2] Add owner integration tests proving Teacher provisioning and the batch-assignment owner request the common `AccountCredentialSetup` event only after their account/profile or account/assignment transaction commits; verify a new Coordinator remains pending until setup completes in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/AccountCredentialSetupOwnerTests.cs`. Keep the Coordinator case a release gate until the batch-assignment owner exists and passes this test. **Status:** the Teacher owner cases pass; the Coordinator producer test is present as a skipped test with the gate as its reason, because no batch-assignment owner exists. Coordinator provisioning is not complete.
- [X] T012 [P] [US2] Add Angular tests for Admin/Student provisioning, safe pending-state feedback, setup-page validation, Admin-only reissue controls, accessible labels/keyboard use, and supported responsive viewports in `iit-academic-portal/src/app/features/users/user-provisioning.spec.ts` and `iit-academic-portal/src/app/features/users/account-credential-setup.spec.ts`.

### Implementation for User Story 2

- [X] T013 [US2] Implement transactional generic Admin/Student account provisioning in `iit-academic-portal-service/src/IitAcademicPortal.Application/UserManagement/UserProvisioningService.cs`; use normalized unique email, create exactly the selected single role/default role, set `PendingCredentialSetup`, create no password, and commit the allowlisted audit event atomically.
- [X] T014 [US2] Implement the purpose-specific `PortalAccountCredentialSetup` token and security-content provider in `iit-academic-portal-service/src/IitAcademicPortal.Application/UserManagement/AccountCredentialSetupService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/AccountCredentialSetupSecurityContentProvider.cs`; recheck pending state and registered destination at delivery time, generate links transiently, and never persist or log proof/content.
- [X] T015 [US2] Register the setup service/provider and notification event handling in `iit-academic-portal-service/src/IitAcademicPortal.Application/DependencyInjection.cs`; keep the setup token purpose distinct from password recovery and preserve the existing password-reset behavior.
- [X] T016 [US2] Add `POST /api/admin/users` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/UserManagementController.cs` after T014-T015; require active Admin and anti-forgery validation, accept only `Admin` or `Student`, commit account/audit before requesting the registered common setup notification, and return no credential or proof.
- [X] T017 [US2] Define a separate request-only setup DTO in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/AccountCredentialSetupContracts.cs` and implement atomic public setup completion in `iit-academic-portal-service/src/IitAcademicPortal.Application/UserManagement/AccountCredentialSetupService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/AccountCredentialSetupController.cs`; accept `userId`, `proof`, and `newPassword` only in this request, require a valid setup-purpose proof and pending status, apply the existing password policy, set the password, activate the account, rotate the security stamp, and commit safe audit data together.
- [X] T018 [US2] Implement active-Admin pending-only reissue in `iit-academic-portal-service/src/IitAcademicPortal.Application/UserManagement/AccountCredentialSetupService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/UserManagementController.cs`; rotate the security stamp before requesting a fresh event occurrence, audit safely, and reject Active/Deactivated accounts without exposing proof material.
- [X] T019 [US2] Rewire the existing Teacher provisioning owner to request `AccountCredentialSetup` after its existing account/profile transaction in `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherProvisioningService.cs`; retain TeacherProfile ownership and leave Coordinator provisioning to its assignment owner.
- [X] T020 [P] [US2] Add typed provisioning and reissue API methods in `iit-academic-portal/src/app/features/users/user-api.ts` and account-type/pending-state models in `iit-academic-portal/src/app/features/users/user.models.ts`.
- [X] T021 [US2] Add Admin/Student provisioning controls and safe pending/reissue feedback using IIT design-system controls/tokens with responsive, keyboard-accessible interaction in `iit-academic-portal/src/app/features/users/user-provisioning.ts` and `iit-academic-portal/src/app/features/users/user-provisioning.html`; present Teacher work through Teacher Management and do not offer direct Coordinator creation.
- [X] T022 [US2] Add the public credential setup page and route using IIT design-system controls/tokens with responsive, keyboard-accessible validation and error states in `iit-academic-portal/src/app/features/users/account-credential-setup.ts`, `iit-academic-portal/src/app/features/users/account-credential-setup.html`, and `iit-academic-portal/src/app/app.routes.ts`; submit the proof only to the setup endpoint and never expose it in logs or UI errors.

**Checkpoint**: Admin and Student generic provisioning plus the common setup flow work independently; Teacher provisioning reuses the flow through its owner; Coordinator remains assignment-derived.

---

## Phase 5: User Story 3 - Maintain Account Access and Lifecycle (Priority: P1)

**Goal**: Allow permitted identity/access changes and safe deactivation while preserving feature ownership, default-role validity, the last active Admin, sessions, and audit history.

**Independent Test**: As an active Admin, change a unique account email, apply only policy-approved role changes, and deactivate an eligible account; verify concurrency conflicts, shared last-Admin serialization, session revocation, atomic audit, retained history, and unchanged feature-owned records.

### Tests for User Story 3

- [X] T023 [P] [US3] Add API tests proving anonymous/non-Admin email-update, role-change, and deactivation requests are denied and an Admin cannot deactivate their own account; also cover email uniqueness/concurrency, Student-only roles, Teacher-required/Admin-editable roles, Coordinator rejection, default-role validity, owner delegation, audit rollback, deactivation, and session revocation in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/UserAccessAndLifecycleTests.cs`.
- [X] T024 [P] [US3] Add PostgreSQL-gated concurrent last-Admin removal/deactivation tests using `AUDIT_TEST_POSTGRES` and the same advisory lock as Teacher Management in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/UserManagementPostgresTests.cs`.
- [X] T025 [P] [US3] Add Angular tests for identity-edit conflicts, permitted role controls, Teacher Management delegation, keyboard-accessible confirmation, and deactivation results in `iit-academic-portal/src/app/features/users/user-maintenance.spec.ts`.

### Implementation for User Story 3

- [X] T026 [US3] Implement generic Admin/Student email updates in `iit-academic-portal-service/src/IitAcademicPortal.Application/UserManagement/UserAccessService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/UserManagementController.cs`; use Identity normalization, uniqueness, and expected-concurrency validation, and audit changes without arbitrary email metadata.
- [X] T027 [US3] Implement the account role endpoint in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/UserManagementController.cs`; delegate Teacher role changes to `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherRoleService.cs`, retain Teacher while active, allow only Admin add/remove, and reject Student or Coordinator violations without direct role-table writes.
- [X] T028 [US3] Implement generic Admin/Student deactivation in `iit-academic-portal-service/src/IitAcademicPortal.Application/UserManagement/UserAccessService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/UserManagementController.cs`; preserve account/role/history rows, revoke all sessions, stage audit atomically, and use the shared last-Admin transaction lock before any Admin-removal/deactivation decision.
- [X] T029 [US3] Add email-only identity editing, permitted access actions, Teacher Management links, and confirmed deactivation using IIT design-system controls/tokens with responsive, accessible interaction in `iit-academic-portal/src/app/features/users/user-maintenance.ts`, `iit-academic-portal/src/app/features/users/user-maintenance.html`, and the account detail view in `iit-academic-portal/src/app/features/users/user-list.ts`.

**Checkpoint**: Account maintenance is independently testable and does not duplicate or bypass TeacherProfile or Coordinator assignment ownership.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verify security, migration, integration boundaries, and the documented validation workflow across all three stories.

- [X] T030 Add cross-story regression coverage that confirms account responses, audits, logs, and permanent notification history contain no passwords, hashes, setup proofs, links, or rendered security bodies in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/AccountCredentialPrivacyTests.cs`.
- [ ] T031 Run the backend migration and full API test suite, PostgreSQL-gated migration/locking tests, Angular test suite, and production build; then observe 10 representative Admins completing account search and an allowed maintenance action, require at least 9 to finish within two minutes, and record the method/results and remaining operational gates in `specs/006-user-management/quickstart.md`. **Status (partial):** every automated suite and the production build ran and passed (results in quickstart.md). The SC-002 usability study with 10 representative Admins has not been run, so this task stays open until it is.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 establishes the notification definition and seed migration.
- **Foundational (Phase 2)**: T002 depends on T001 only for task sequencing and blocks story API implementation until common DTOs/validation are defined.
- **User Stories (Phases 3-5)**: Each story starts after T002. They are behaviorally independently testable; coordinate edits to the shared `UserManagementController.cs`, `user-api.ts`, and `user.models.ts` when stories are staffed in parallel.
- **Polish (Phase 6)**: Depends on all selected stories and their migrations being complete.

### User Story Dependencies

- **User Story 1 (P1)**: Starts after Phase 2; no dependency on another story.
- **User Story 2 (P1)**: Starts after Phase 2 and T001; uses its own pending-account/setup flow and does not require the directory UI. Coordinator setup-event production remains blocked on the batch-assignment owner and T011.
- **User Story 3 (P1)**: Starts after Phase 2; delegates Teacher mutations to Teacher Management and does not require User Story 2.
- The Student-profile owner, batch-coordinator assignment owner, first-Admin deployment bootstrap, and Production email-provider/Data Protection operations remain release gates, not new workflows owned by this feature.

### Within Each User Story

- Write the story tests before implementing the corresponding behavior.
- Complete service behavior before exposing it through API/UI; preserve the shared Identity, audit, notification, and session ownership boundaries.
- In User Story 2, complete T014-T015 before T016 so account provisioning cannot request an event before its security-content provider is registered.
- A story is complete only when its independent test criteria pass.

### Parallel Opportunities

- In User Story 1, T003 and T004 can be authored in parallel; backend directory work and frontend API tests can proceed independently after T002.
- In User Story 2, T009-T012 are separate test files and can be authored in parallel. The setup token/provider path depends on T001; generic provisioning, Teacher notification wiring, and frontend work can proceed on separate files after the contracts are settled.
- In User Story 3, T023-T025 can be authored in parallel. PostgreSQL concurrency tests and frontend maintenance work are separate from application service changes.
- T030 and operational validation T031 are final cross-story checks; do not run T031 until the selected implementation and migration are complete.

## Parallel Example: User Story 1

```text
Task T003: API directory authorization, filtering, paging, and projection tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/UserDirectoryTests.cs
Task T004: Angular account-list behavior tests in iit-academic-portal/src/app/features/users/user-list.spec.ts
Task T005: Directory projection service in iit-academic-portal-service/src/IitAcademicPortal.Application/UserManagement/UserDirectoryService.cs
```

## Parallel Example: User Story 2

```text
Task T009: Generic provisioning API tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/UserProvisioningTests.cs
Task T010: Credential setup proof/reissue tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/AccountCredentialSetupTests.cs
Task T011: Teacher and Coordinator setup-event owner integration tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/AccountCredentialSetupOwnerTests.cs
Task T012: Provisioning and setup UI tests in iit-academic-portal/src/app/features/users/
```

## Parallel Example: User Story 3

```text
Task T023: User access/lifecycle API tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/UserAccessAndLifecycleTests.cs
Task T024: PostgreSQL last-Admin concurrency tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/UserManagementPostgresTests.cs
Task T025: Account maintenance UI tests in iit-academic-portal/src/app/features/users/user-maintenance.spec.ts
```

## Implementation Strategy

### MVP First (User Story 1)

1. Complete Setup and Foundational phases.
2. Complete User Story 1 and verify its independent authorization, directory, and detail criteria.
3. Stop for review/demo with an Admin-only account directory that has no mutation controls.

### Incremental Delivery

1. Add User Story 2 for Admin/Student provisioning and secure setup; validate Teacher owner integration before enabling the shared event for Teacher provisioning.
2. Add User Story 3 for allowed identity/access maintenance and deactivation; verify PostgreSQL shared last-Admin locking before release.
3. Complete cross-story privacy tests and the quickstart validation gates.

### External Release Gates

- Confirm Student profile and batch-coordinator assignment owners before claiming profile/Coordinator end-to-end integration.
- Configure the first active Admin through deployment bootstrap; Admin-only pages cannot bootstrap their own authority.
- Configure and verify the public setup URL, durable shared Data Protection key ring, and Production email provider/content-retention policy before production activation.

## Notes

- Generic provisioning in this feature creates only Admin and Student Identity accounts. Teacher provisioning and Teacher-specific mutation remain owned by Teacher Management; Coordinator is derived by the batch-assignment owner.
- The planned EF migration seeds the setup notification template/revision only; no account column/table is planned.
- `[P]` marks tasks in separate files with no incomplete-task dependency. Every task has a sequential ID, checkbox, required story label where applicable, and implementation/test path.

## Phase 7: Convergence

- [ ] T032 Conduct the SC-002 usability study with 10 representative Admins completing account search and an allowed maintenance action; record the task prompt, participant count, completion count, and elapsed times in `specs/006-user-management/quickstart.md` per SC-002 (partial).
- [ ] T033 When the batch-assignment owner is implemented, request `AccountCredentialSetupService.InitialRequest(userId)` after its new Coordinator account/assignment transaction commits and replace the skipped producer test with a passing integration test in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/UserManagement/AccountCredentialSetupOwnerTests.cs` per FR-009 (missing; blocked on the owner).