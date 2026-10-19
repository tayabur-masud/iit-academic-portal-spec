# Tasks: Student Management

**Input**: Design documents from `/specs/007-student-management/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: Automated API, data-integrity, audit, authorization, and UI tests are required by the project constitution and the plan.

**Organization**: Tasks are grouped by the three independently testable user stories. The pending-account setup, audit, and active-Admin policies apply across all stories.

## Phase 1: Setup

**Purpose**: Establish the Student Management feature locations in the existing projects without adding packages or projects.

- [X] T001 Create Student Management source and test folders under `iit-academic-portal-service/src/IitAcademicPortal.Domain/Students/`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/`, `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/`, `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/`, and `iit-academic-portal/src/app/features/students/`.

---

## Phase 2: Foundational

**Purpose**: Add the shared Student profile, multiple-admission data model, and persistence boundary used by all three stories.

- [X] T002 Add `StudentProfile`, `StudentAdmissionRecord`, and `StudentAcademicHistory` in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Students/StudentProfile.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Domain/Students/StudentAdmissionRecord.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Domain/Students/StudentAcademicHistory.cs`; derive business records from `AuditedEntity`, put the unique required `UserId`, required `ApplicantMobile`, separate optional mailing/permanent address details and Upazila references, and zero or more StudentAcademicHistory entries on StudentProfile, make admission `StudentProfileId`, `DegreeId`, and `BatchId` required, keep admission `AssignedHallId` and `RegistrationNumber` optional, define Category as the six fixed values, add optional catalog IDs plus name snapshots for school/college, university, institution, board, and group, allow typed fallback for unlisted values, and keep all other applicable FR-003 fields as optional text snapshots.
- [X] T003 Map StudentProfile, StudentAdmissionRecord, and StudentAcademicHistory in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs`; enforce one profile per `UserId`, many admissions per profile, StudentAcademicHistory parented to StudentProfile, restrictive foreign keys for selected institution/board/group catalog records, mailing/permanent Upazila foreign keys, at most one admission per `(StudentProfileId, DegreeId, BatchId)`, unique non-null `RegistrationNumber`, restrictive Degree/Batch/Hall/account references, Batch-to-Degree consistency in application validation, and add the versioned migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`.
- [X] T004 Define the student profile/admission unit-of-work contract in `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IStudentStore.cs`, implement it in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/StudentStore.cs`, and register it in the existing dependency-injection extension; first-time account/profile/admission creation and Student-account reuse must share the request's `PortalDbContext` transaction.

**Checkpoint**: The audited student model, relationships, migration, and persistence transaction boundary are available to every story.

---

## Phase 3: User Story 1 - Enter a Student Admission Record (Priority: P1)

**Goal**: An active Admin creates a program admission under one person-level Student profile; the first admission creates the Student account and later admissions reuse it.

**Independent Test**: Create an initial admission and then a second admission for the same normalized email under a different Degree/Batch. Verify one Student profile/account, two distinct admissions, one setup email only for initial account creation, retained academic-history values, rejection of missing mobile/non-Student email/duplicate same Degree+Batch admission, and no personal data access for unauthorized users.

### Tests for User Story 1

- [X] T005 [P] [US1] Add API tests for initial and returning admission create/read/list, same normalized Student email reusing the same account/profile, separate admissions for different Degree/Batch pairs, duplicate same-profile/Degree/Batch rejection, non-Student email conflict, required profile contact fields, all six history categories, valid catalog IDs and name snapshots, typed fallback for unlisted values, institution-type filtering, inactive/wrong-type catalog rejection, mailing/permanent details with valid Upazila references and resolved parents, invalid/missing address locations, and unauthorized access in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/StudentAdmissionTests.cs` and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/StudentAcademicHistoryCatalogTests.cs`.
- [X] T006 [P] [US1] Add tests proving first account/profile/admission and audits commit atomically, a failed write leaves none created, returning admissions reuse the profile without changing account status or requesting setup again, and setup activation/proof failures follow the existing flow in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/StudentProvisioningTransactionTests.cs`.
- [X] T007 [P] [US1] Add Angular tests for creating and viewing multiple program admissions under one Student profile, required email/mobile/Degree/Batch fields, all six profile academic-history categories, catalog selection and name snapshots, typed fallback for unlisted catalog values, cascading Division/District/Upazila selectors for both addresses, address validation/error states, and omission of file-upload controls in `iit-academic-portal/src/app/features/students/student-form.spec.ts`.

### Implementation for User Story 1

- [X] T008 [US1] Implement admission validation and creation in `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/StudentAdmissionService.cs`; preserve FR-002/FR-003 form fields, enforce the fixed StudentAcademicHistory Category, validate chosen school/college, university, institution, board, and group catalog IDs and institution types, store catalog name snapshots, allow typed values for unlisted records, and treat other applicable history values as optional text. Use `PortalUser.Email` as canonical identity, require mobile, create StudentProfile plus Student account only when none exists, reuse an existing Student account/profile for the same normalized email, attach a profile to an existing Student account when absent, validate optional mailing/permanent address details and Upazila references, reject a non-Student account or duplicate same-profile/Degree/Batch admission, and request setup notification only after a new account's profile/admission transaction commits.
- [X] T009 [US1] Define profile/admission request and response DTOs and implement `GET /api/admin/students`, `POST /api/admin/students`, and `GET /api/admin/students/{admissionId}` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/StudentManagementContracts.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Students/StudentsController.cs`; return the new/reused account outcome, require `AdminModule`, use field-keyed Problem Details, bounded paging, and never expose a password or setup proof.
- [X] T010 [US1] Implement typed StudentProfile/admission models and API access in `iit-academic-portal/src/app/features/students/student.models.ts` and `iit-academic-portal/src/app/features/students/student-api.ts`, plus a searchable paginated admission list showing shared Student identity and distinct program admissions and the common profile/admission/history form with fixed Category selection, optional source-value text fields, and cascading Division/District/Upazila address controls in `student-list.ts`, `student-list.html`, `student-form.ts`, and `student-form.html`; register the Admin-only route and navigation entry in `iit-academic-portal/src/app/app.routes.ts` and `iit-academic-portal/src/app/layout/shell/shell.ts`.

**Checkpoint**: An Admin can enter and read a common admission; account, audit, and student data are atomic, and the existing credential-setup flow is reused.

---

## Phase 4: User Story 2 - Complete Student Identifiers Later (Priority: P1)

**Goal**: An active Admin independently assigns or corrects Hall and registration number after admission.

**Independent Test**: Create an admission with both values unset, assign Hall alone, then registration number alone; verify each is retained and audited. Reuse another student's registration number and verify rejection without changing either record. Verify non-Admin updates are denied.

### Tests for User Story 2

- [X] T011 [P] [US2] Add API tests for initially absent Hall/registration number, independent later assignment/correction, inactive or missing Hall rejection, duplicate registration number rejection without writes, audit changes, and unauthorized updates in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/StudentIdentifierTests.cs`.
- [X] T012 [P] [US2] Add Angular tests proving Hall and registration number can be edited independently, existing values are retained on an unrelated update, field errors are shown, and unauthorized writes fail in `iit-academic-portal/src/app/features/students/student-form.spec.ts`.

### Implementation for User Story 2

- [X] T013 [US2] Implement full student update and identifier validation in `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/StudentAdmissionService.cs`; `AssignedHallId` and `RegistrationNumber` remain optional and can be changed independently, a newly assigned Hall must be active, and a present registration number must be unique across all admissions including inactive accounts.
- [X] T014 [US2] Add `PUT /api/admin/students/{id}` mapping and field-error/status handling in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Students/StudentsController.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/StudentManagementContracts.cs`; preserve unchanged Hall/registration values and stage the update audit event in the same unit of work.
- [X] T015 [US2] Add the student edit and later-identifier controls to `iit-academic-portal/src/app/features/students/student-form.ts` and `student-form.html`; load current values, permit saving either identifier independently, and retain focus/error behavior for field-keyed service errors.

**Checkpoint**: Hall and registration number can be maintained independently with uniqueness, authorization, and audit integrity.

---

## Phase 5: User Story 3 - Calculate Graduation Eligibility (Priority: P1)

**Goal**: Compute eligibility from the selected Degree's required credits, passed-course credits, Hall, and registration number without owning results or blocking result publication.

**Independent Test**: Evaluate fewer credits, missing credit source, sufficient credits with either identifier missing, and all requirements met. Verify only the final case is eligible, each unmet/unavailable requirement is identified, and the calculation does not change or block academic results.

### Tests for User Story 3

- [X] T016 [P] [US3] Add eligibility tests for below-required credits, exact threshold, missing Degree requirement, unavailable completed-credit provider, missing Hall, missing registration number, and all requirements met across two admissions for one profile in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/GraduationEligibilityTests.cs`; prove provider totals are keyed per admission, PGDIT credits do not carry into EMIT, unavailable data never yields eligibility, and the check does not block or modify final-result calculation/publication.
- [X] T017 [P] [US3] Add Angular tests for eligible, ineligible, unmet-requirement, and credit-source-unavailable states in `iit-academic-portal/src/app/features/students/graduation-eligibility.spec.ts`.

### Implementation for User Story 3

- [X] T018 [US3] Define the read-only `ICompletedCreditsProvider` result contract in `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/ICompletedCreditsProvider.cs`, keyed by `StudentAdmissionRecord.Id`, distinguishing an available total from unavailable source data; register an unavailable implementation until the academic-results owner supplies a provider, and do not persist completed credits in StudentProfile or StudentAdmissionRecord.
- [X] T019 [US3] Implement `GraduationEligibilityService` in `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/GraduationEligibilityService.cs` to read the Degree for the selected admission, obtain completed credits for that admission from `ICompletedCreditsProvider`, require that admission's Hall and registration number, and return its eligibility plus unmet/unavailable requirements without modifying results or counting credits from another admission.
- [X] T020 [US3] Add the eligibility response DTO and `GET /api/admin/students/{admissionId}/graduation-eligibility` action in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/StudentManagementContracts.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Students/StudentsController.cs`; protect the route with `AdminModule` and return an explicit unavailable-credit reason when the provider has no data.
- [X] T021 [US3] Display computed eligibility, completed/required credits when available, and each missing or unavailable requirement in `iit-academic-portal/src/app/features/students/graduation-eligibility.ts` and `graduation-eligibility.html`; do not add a clearance action or block final-result publication.
- [ ] T022 [US3] When the academic-results owner publishes the passed-course credit contract, implement and register its read-only adapter for `ICompletedCreditsProvider` in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/AcademicResults/CompletedCreditsProvider.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs`, then add an integration test for the eligible threshold case in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/GraduationEligibilityTests.cs`; until then keep the unavailable provider behavior. **Status (blocked):** no academic-results owner exists. `UnavailableCompletedCreditsProvider` is registered, and the threshold logic is covered against a fake provider in `GraduationEligibilityTests`.

**Checkpoint**: Eligibility is correct for available provider data and safely ineligible when completed credits are unavailable; no student-owned result or credit copy is created.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verify provider-specific persistence, the full quickstart, and regression gates.

- [X] T023 [P] Add PostgreSQL 18 tests for one StudentProfile per portal account, multiple admissions per profile, one admission per profile/Degree/Batch, mailing/permanent Upazila foreign keys, StudentAcademicHistory-to-StudentProfile relationships and text-snapshot column storage, restrictive Degree/Batch/Hall foreign keys, and unique non-null registration numbers in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/StudentPostgresTests.cs`.
- [X] T024 Run the service test suite, frontend tests, and production build from `specs/007-student-management/quickstart.md`; execute PostgreSQL tests when `AUDIT_TEST_POSTGRES` is configured and record actual results in that quickstart.
- [ ] T025 Conduct a timed usability study with representative Admin participants after the admission form is implemented; record participant count, completion times, assistance, and results in `specs/007-student-management/quickstart.md`, and verify at least 90% can enter a completed paper form within five minutes without developer assistance per SC-002. **Status:** not run; it needs real Admin participants.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Establishes the existing-project feature locations; no packages or projects are added.
- **Foundational (Phase 2)**: Creates the audited StudentProfile, many-to-one admission model, schema, and transaction/store boundary; blocks all user stories.
- **User Stories (Phases 3–5)**: US1 depends on Foundation. US2 depends on US1 for an existing admission. US3 depends on US1 and US2 for Degree/Batch and identifier data. The external provider adapter in T022 additionally depends on the academic-results owner publishing its contract.
- **Polish (Phase 6)**: Depends on all implemented stories; provider-specific integration verification requires the external adapter and PostgreSQL test configuration.

### User Story Dependencies

- **US1 (P1)**: Independent after Foundation; delivers StudentProfile, initial account creation, and first program admission, with later program admissions reusing the identity.
- **US2 (P1)**: Depends on US1 records and adds later Hall/registration updates.
- **US3 (P1)**: Depends on the admission/Degree fields from US1 and identifiers from US2; the positive completed-credit integration case depends on the results provider contract.

### Parallel Opportunities

- T005, T006, and T007 cover separate backend, transaction, and frontend tests and can be authored in parallel after T001–T004.
- T011 and T012 test different backend/frontend files and can be authored in parallel after US1.
- T016 and T017 test different backend/frontend files and can be authored in parallel after US2.
- After provider contract T018 is stable, API eligibility work and the eligibility view can proceed in parallel.
- T023 PostgreSQL test authoring can proceed alongside story implementation after the schema model is stable.

## Parallel Example: User Story 1

```text
After T001–T004:
- API worker: T005, then T008–T009
- UI worker: T006, then T010
Integrate and validate the end-to-end admission/account transaction before completing US1.
```

## Implementation Strategy

### MVP First

1. Complete Setup and Foundation.
2. Complete US1 StudentProfile, first admission, returning-Student account reuse, academic history, atomic account creation, and Admin UI.
3. Validate the US1 independent test before starting later identifier and graduation work.

### Incremental Delivery

1. Deliver US1 as the first usable increment.
2. Add US2 for deferred Hall and registration data.
3. Add US3 with a safe unavailable-provider outcome; integrate the positive completed-credit case when the academic-results owner publishes the provider contract.
4. Run the PostgreSQL, full backend, frontend, and quickstart gates.

### Format Validation

Every task starts with `- [ ]`, has one sequential ID, includes `[US#]` exactly for story tasks, uses `[P]` only for independent work, and names its target file or directory.

## Phase 7: Convergence

- [ ] T026 [US3] Obtain and document the academic-results owner's read-only passed-course credit contract keyed by `StudentAdmissionRecord.Id` in `specs/007-student-management/research.md` and `specs/007-student-management/contracts/student-management-api.md`; then complete T022's provider adapter and positive per-admission eligibility integration test per FR-006, FR-012, and SC-003 (partial).
- [ ] T027 [US1] Verify keyboard-only operation, focus/error announcements, screen-reader labels, and responsive layouts for the Student list, admission/history form, and eligibility view; record tested assistive-technology and viewport results in `specs/007-student-management/quickstart.md` per Constitution I and the plan's accessible/responsive UI decision (partial).

## Phase 8: Convergence

- [X] T028 [US1] Reconcile the checked StudentAcademicHistory work in T002/T003/T005/T007/T008 and supporting notes in `specs/007-student-management/research.md` and `specs/007-student-management/checklists/requirements.md` with revised FR-003: the six categories are fixed; listed school/college, university, institution, board, and group values use catalog IDs plus immutable name snapshots; unlisted values use typed text snapshots; all other applicable history fields remain optional text snapshots. Treat the revised spec, data model, API contract, and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/StudentAcademicHistoryCatalogTests.cs` as authoritative and document that the earlier text-only/no-catalog wording is superseded (contradicts).