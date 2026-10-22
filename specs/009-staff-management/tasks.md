---

description: "Actionable tasks for Staff Management"
---

# Tasks: Staff Management
**Input**: Design documents from `/specs/009-staff-management/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: Required by the implementation plan and project constitution for this security-sensitive account lifecycle. Include API, PostgreSQL, and Angular tests.

**Organization**: Tasks are grouped under the sole P1 user story. The existing profile-only implementation is the starting point; these tasks cover the approved provisioning, account-attachment, and deactivation behavior that is not yet implemented.

## Phase 1: Setup

**Purpose**: Use the existing Angular and ASP.NET Core projects and their established test/tooling setup.

No project initialization, package, or framework setup is required. The existing application, test projects, authorization, audit, notification, session-revocation, and Designation infrastructure are reused.

---

## Phase 2: Foundational

**Purpose**: Establish shared prerequisites before the user story.

No new cross-story foundation is required. Existing active-Admin authorization, audited entities, User Management identity/role infrastructure, secure credential setup, and Teacher lifecycle patterns are available. The feature-specific lifecycle migration is implemented within User Story 1.

---

## Phase 3: User Story 1 - Maintain Staff Profile Details (Priority: P1) - MVP

**Goal**: Let an active Admin create a pending Admin account and StaffProfile together for a new email, attach a profile to an eligible existing Admin account without changing it, maintain manually verified staff details, and deactivate the linked profile/account together with session revocation and safety checks. My Profile remains view-only.

**Independent Test**: As an active Admin, create staff with a new email and verify the Admin account, StaffProfile, and required audit events commit atomically while the account remains pending until credential setup; attach a profile to an eligible existing active Admin without changing credentials, status, or setup-email state; verify ineligible accounts are rejected; update profile data only with explicit audited manual-verification confirmation; and deactivate another staff member while verifying account disablement, session revocation, preserved history, and self/last-Admin safeguards. Confirm My Profile has no write action.

### Tests for User Story 1

- [X] T001 [P] [US1] Extend API tests for new-email account/profile/audit atomicity, pending Admin role/default with no password, credential-setup request only after commit, setup-delivery failure, existing eligible Admin attachment without account changes or another setup request, ineligible/duplicate account rejection, and staff deactivation/session revocation/self/last-Admin/repeat behavior in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Staff/StaffManagementTests.cs`.
- [X] T002 [P] [US1] Extend PostgreSQL tests to apply the additive lifecycle migration, verify existing StaffProfiles default active and new profiles persist `IsActive`, preserve the unique UserId and restrictive foreign keys, and verify transaction/audit integrity for account provisioning and deactivation in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Staff/StaffManagementPostgresTests.cs`.
- [X] T003 [P] [US1] Extend Angular tests for email-based create/link outcomes, removal of the eligible-account picker, setup-request outcome display, deactivation confirmation and inactive state, request errors, and the view-only My Profile action in `iit-academic-portal/src/app/features/staff/staff-management.spec.ts` and `iit-academic-portal/src/app/layout/shell/shell.spec.ts`.

### Implementation for User Story 1

- [X] T004 [US1] Add the staff lifecycle state to `StaffProfile` in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Staff/StaffProfile.cs`: `IsActive` is a required Boolean that defaults true; provide deactivation behavior without a Staff Management reactivation operation, while retaining the existing Name, Mobile number, Designation, and audit rules.
- [X] T005 [US1] Map `StaffProfile.IsActive` in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs` and add a follow-up versioned migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`; add the required Boolean with a true default for existing profiles, preserve unique restrictive UserId and required restrictive DesignationId relationships, and do not modify applied migration `20261021174956_StaffProfiles`.
- [X] T006 [US1] Extend the Staff persistence boundary in `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IStaffStore.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/StaffStore.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs` to support the Staff-owned account/profile/audit unit of work, eligible account lookup by email, Admin-invariant locking, and lifecycle persistence/session-revocation coordination using existing transaction patterns.
- [X] T007 [US1] Update `iit-academic-portal-service/src/IitAcademicPortal.Application/Staff/StaffProfileService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Staff/StaffManagementModels.cs` to create by email: for a new email atomically create one PortalUser with only the Admin role, Admin default role, no password, and PendingCredentialSetup plus its StaffProfile and account/profile audit events; for an existing active Admin without a profile attach only the StaffProfile and its audit; reject deactivated, non-Admin, and already-profiled accounts; request the existing credential-setup notification only after the new-account transaction commits, with delivery failure leaving the account pending.
- [X] T008 [US1] Update create request/response DTOs in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/StaffManagementContracts.cs` and the active-Admin `POST /api/admin/staff` handling in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Staff/StaffController.cs` to accept email instead of a selected userId, expose whether a new account and setup request were produced without exposing credentials/proofs, and return field-keyed validation or conflict responses for rejected account states.
- [X] T009 [US1] Implement staff deactivation in a focused lifecycle service at `iit-academic-portal-service/src/IitAcademicPortal.Application/Staff/StaffLifecycleService.cs` and add its audit event in `iit-academic-portal-service/src/IitAcademicPortal.Application/Auditing/StaffAuditEventDefinitions.cs`; in one transaction mark StaffProfile inactive, deactivate the linked Admin account, revoke active sessions, and record the profile/account audit, preserving history and rejecting self-deactivation or removal of the last active Admin while treating repeat deactivation as a no-op.
- [X] T010 [US1] Add the active-Admin `POST /api/admin/staff/{staffProfileId}/deactivation` endpoint and response/error mapping in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Staff/StaffController.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/StaffManagementContracts.cs`; expose no reactivation, physical-delete, or role-edit operation.
- [X] T011 [US1] Update client request/response models and API calls in `iit-academic-portal/src/app/features/staff/staff.models.ts` and `iit-academic-portal/src/app/features/staff/staff-api.ts`, then change `iit-academic-portal/src/app/features/staff/staff-form.ts` and `iit-academic-portal/src/app/features/staff/staff-form.html` to create by email and show the returned account/setup outcome instead of selecting an account from the existing eligible-account picker; retain required field validation and per-save manual-verification confirmation.
- [X] T012 [US1] Add the deactivation action, confirmation state, inactive-profile display, and API error handling in `iit-academic-portal/src/app/features/staff/staff-list.ts` and `iit-academic-portal/src/app/features/staff/staff-list.html`; keep the action unavailable for the signed-in Admin's own account when identifiable and keep My Profile view-only.
- [X] T013 [US1] Update `specs/009-staff-management/contracts/staff-management-api.md`, `specs/009-staff-management/quickstart.md`, and `specs/009-staff-management/implementation-notes.md` to document email-based create-or-attach outcomes, post-commit credential setup, deactivation/session revocation/safeguards, the additive IsActive migration, and validation scenarios; remove stale eligible-account-picker and profile-only lifecycle statements while retaining the prior verification record as explicitly limited to the old implementation.

**Checkpoint**: The single P1 story is independently testable with new-account provisioning, eligible-account attachment, profile maintenance, and safe deactivation; existing profile behavior and view-only My Profile remain intact.

---

## Phase 4: Polish & Cross-Cutting Validation

**Purpose**: Run the planned focused and project quality gates and record actual results.

- [X] T014 Run the Staff API and PostgreSQL test classes in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Staff/StaffManagementTests.cs` and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Staff/StaffManagementPostgresTests.cs`, the Angular tests in `iit-academic-portal/src/app/features/staff/staff-management.spec.ts` and `iit-academic-portal/src/app/layout/shell/shell.spec.ts`, the service test project, and `npm test -- --watch=false` plus `npm run build` in `iit-academic-portal`; record actual commands, migration status, and outcomes in `specs/009-staff-management/quickstart.md`.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Already satisfied by the existing projects and tooling; no new dependencies or packages are introduced.
- **Foundational (Phase 2)**: Existing authorization, audit, account, notification, session, and Designation services satisfy the shared prerequisites.
- **User Story 1 (Phase 3)**: Depends on the existing infrastructure and approved contract; it does not depend on another feature story.
- **Polish (Phase 4)**: T014 follows the migration, implementation, documentation, and story tests.

### User Story Dependencies

- **User Story 1 (P1)**: Can proceed independently once the already-existing shared infrastructure is available. It is the MVP and only user story in this specification.

### Within User Story 1

- T001-T003 are independent test files and can be authored in parallel before implementation.
- T004 precedes T005; T006 provides persistence operations used by T007 and T009.
- T007 precedes T008. T009 uses T004/T006 and precedes T010.
- T011 can begin after the create DTO shape in T008 is agreed; T012 depends on the deactivation API shape in T010 and the client API update in T011.
- T013 follows implementation/contract stabilization; T014 follows all story work.

## Parallel Opportunities

- **Tests**: T001 API, T002 PostgreSQL, and T003 Angular tests can run in parallel because they modify separate test files.
- **Implementation**: Once the create DTO is agreed, the frontend create flow in T011 can proceed alongside backend deactivation work in T009-T010. Avoid concurrent edits to `StaffController.cs` and `StaffManagementContracts.cs` across T008/T010; sequence those tasks or coordinate one owner.

## Parallel Example: User Story 1

```text
Before implementation:
- Worker A: T001 API lifecycle and provisioning tests
- Worker B: T002 PostgreSQL migration and transaction tests
- Worker C: T003 Angular create/link and deactivation tests

After T008 establishes the create contract:
- Backend: T009, then T010
- Frontend: T011, then T012 (T012 also waits for T010's deactivation route)
- Documentation: T013 after the behavior and contract are stable
```

## Implementation Strategy

### MVP First

1. Complete T001-T003 as focused regression tests.
2. Complete T004-T008 for the lifecycle migration and atomic create-or-attach account workflow.
3. Complete T009-T012 for safe deactivation and the Admin UI.
4. Validate the full P1 independent test before considering the story complete.

### Incremental Delivery

1. Deliver safe new-account provisioning and eligible existing-account attachment with API/PostgreSQL tests.
2. Deliver profile deactivation with atomic account/session/audit behavior and Admin safeguards.
3. Deliver the updated create/deactivation UI, then run T014 and record the verification evidence.

## Format Validation

All 14 task entries use `- [ ] T###`; story tasks carry `[US1]`, parallel markers appear only on independent test files, and each task includes the exact repository file path(s) it changes or validates. Setup, Foundational, and Polish tasks have no story label.
***