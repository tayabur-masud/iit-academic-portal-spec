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
- [ ] T022 [US3] When the academic-results owner publishes its admission-keyed read-only contract for passed-course completed credits and current cumulative CGPA from published results, implement and register the `IPublishedAdmissionResultsProvider` adapter in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/AcademicResults/PublishedAdmissionResultsProvider.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs`; add PostgreSQL/API integration coverage for the eligible credit and CGPA thresholds in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/GraduationEligibilityTests.cs`; until then keep both values independently unavailable. **Status (blocked):** the academic-results owner has not published this contract. The unavailable fallback remains registered and local threshold behavior is tested with a fake provider.

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

---

## Phase 9: Student Address Locality Addendum - User Story 1 (Priority: P1)

**Goal**: Let Admins use either an Upazila or City Corporation as the locality for each mailing/permanent address, with both choices constrained to the selected District and existing Upazila references preserved.

**Independent Test**: Create and update profiles using both locality types. Verify each address contains one locality under its submitted District, the API returns the locality and derived District/Division, invalid or conflicting locality submissions write nothing, existing Upazila references survive migration, and a referenced City Corporation cannot be deleted.

### Tests for User Story 1 Address Addendum

- [X] T029 [P] [US1] Add API tests for mailing and permanent addresses using either locality, required `districtId`, rejection when both or neither locality IDs are supplied, cross-District/missing/deleted locality rejection, unchanged inactive locality behavior, and no partial profile updates in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/StudentAdmissionTests.cs`.
- [X] T030 [P] [US1] Add PostgreSQL tests for the Student address locality migration, both restrictive City Corporation foreign keys, at-most-one-locality check constraints, preservation of existing Upazila IDs, and a City Corporation referenced by a Student profile blocking Configuration deletion in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/StudentPostgresTests.cs` and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationDeleteIntegrityTests.cs`.
- [X] T031 [P] [US1] Add Angular tests proving both Upazilas and City Corporations load only for the selected District, switching Division/District clears the previous locality, either locality type serializes with `districtId`, and mixed/missing locality validation is shown in `iit-academic-portal/src/app/features/students/student-form.spec.ts`.

### Implementation for User Story 1 Address Addendum

- [X] T032 [US1] Add `MailingCityCorporationId` and `PermanentCityCorporationId` to `StudentProfile` in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Students/StudentProfile.cs`; preserve the existing Upazila IDs and enforce these model rules verbatim: “exactly one locality reference is required when MailingAddressDetails is supplied” and “exactly one locality reference is required when PermanentAddressDetails is supplied”; neither address may store both locality types.
- [X] T033 [US1] Map the City Corporation address foreign keys in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs` and add a versioned migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/` with nullable `MailingCityCorporationId`/`PermanentCityCorporationId`, restrictive foreign keys, indexes, and checks preventing both locality IDs being set together; update `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/ConfigurationStore.cs` so a live City Corporation referenced by either address blocks soft deletion, and cover the delete conflict in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationDeleteIntegrityTests.cs`.
- [X] T034 [US1] Update address validation and lookups in `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/StudentAdmissionService.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IStudentStore.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/StudentStore.cs` to require address details, a `districtId` validation context, and exactly one of `upazilaId` or `cityCorporationId`; verify the selected locality belongs to the District, require a newly selected locality to be active while preserving an unchanged inactive reference, derive Division, and persist only the selected locality ID.
- [X] T035 [US1] Update address request/response mapping in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/StudentManagementContracts.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Students/StudentsController.cs` to accept `districtId` plus exactly one locality ID and return the selected locality type/ID/name with resolved District/Division; return field-keyed 400 errors for missing District, missing/both locality IDs, nonexistent/deleted localities, and District mismatch.
- [X] T036 [US1] Update address models and cascading forms in `iit-academic-portal/src/app/features/students/student.models.ts`, `iit-academic-portal/src/app/features/students/student-api.ts`, `iit-academic-portal/src/app/features/students/student-form.ts`, and `iit-academic-portal/src/app/features/students/student-form.html` to load both Configuration locality catalogs under the selected District, offer exactly one locality per address, reset it when the District changes, submit `districtId` as validation context, and display the returned locality with its derived District and Division.

**Checkpoint**: Both address-locality types work without breaking existing Upazila records, and District/locality integrity is enforced by the API and database.

---

## Phase 10: Published CGPA Eligibility Addendum - User Story 3 (Priority: P1)

**Goal**: Require the selected Degree's minimum CGPA for graduation eligibility, comparing it with the current cumulative CGPA for that admission from published course results using configured Grade numeric values; Student Management remains read-only for results.

**Dependencies**: Existing US1/US2 admission and identifier records. Positive result-source integration remains blocked on the academic-results owner publishing its contract (T022/T026); tests and unavailable-source behavior are independently implementable.

**Independent Test**: With a fake admission-keyed published-results provider, evaluate current cumulative CGPA below, equal to, and above the Degree minimum; verify below is ineligible, equality/above meets the CGPA condition, unavailable Degree/result data is identified and never eligible, another admission's CGPA is not reused, and a newly published result changes the next eligibility read without persisting CGPA in Student Management.

### Tests for User Story 3 CGPA Addendum

- [X] T037 [P] [US3] Add API/service tests for minimum CGPA below/equal/above thresholds, missing Degree minimum, completed credits available while CGPA is unavailable (and vice versa), current cumulative CGPA keyed to the requested admission, result updates affecting the next evaluation, no cross-admission CGPA transfer, and the unchanged unavailable-provider behavior in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Students/GraduationEligibilityTests.cs`.
- [X] T038 [P] [US3] Add Angular eligibility tests for the minimum-CGPA requirement, displayed current cumulative CGPA and Degree minimum, below/equal/above threshold states, independent unavailable-credit/CGPA messages, and refresh after an admission update in `iit-academic-portal/src/app/features/students/graduation-eligibility.spec.ts`.

### Implementation for User Story 3 CGPA Addendum

- [X] T039 [US3] Replace `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/ICompletedCreditsProvider.cs` with `IPublishedAdmissionResultsProvider.cs` in the same folder (the interface name T022 uses): an admission-keyed published-results contract returning completed passed-course credits and current cumulative CGPA independently, with a separate unavailable reason for each value; register an unavailable implementation until the owner adapter in T022 is ready, and do not calculate or persist either result value in Student Management.
- [X] T040 [US3] Extend the Degree reference projection in `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IStudentStore.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/StudentStore.cs` to return the selected Degree's configured `MinimumCgpaForCompletion` alongside its required credits; do not copy either Degree setting onto StudentProfile or StudentAdmissionRecord.
- [X] T041 [US3] Update `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/GraduationEligibilityService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Students/StudentModels.cs` to add a `MinimumCgpa` requirement, compare the selected Degree minimum with that admission's current cumulative published CGPA, identify missing values as `Unavailable`, and set `Eligible` only when credits, CGPA, Hall, and registration number all meet their requirements.
- [X] T042 [US3] Update `GraduationEligibilityResponse` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/StudentManagementContracts.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Students/StudentsController.cs` to return `minimumCgpa` and `currentCumulativeCgpa` when available and a `MinimumCgpa` requirement status/message; omit unavailable numeric values and never return a computed or persisted Student-owned CGPA.
- [X] T043 [US3] Update `GraduationEligibility` types and the eligibility panel in `iit-academic-portal/src/app/features/students/student.models.ts`, `iit-academic-portal/src/app/features/students/graduation-eligibility.ts`, and `iit-academic-portal/src/app/features/students/graduation-eligibility.html` to show the Degree minimum, current cumulative CGPA, and distinct unmet/unavailable requirements without adding course-result entry or calculation controls.

**Checkpoint**: Eligibility compares published, current cumulative CGPA for the same admission with the selected Degree minimum; no result values are calculated or stored by Student Management.

---

## Phase 11: Addendum Validation & Cross-Cutting Concerns

**Purpose**: Verify migration, address, result-read, and existing Student Management behavior together.

- [X] T044 Run focused Student API/PostgreSQL tests, the full service test suite, `npm test -- --watch=false`, and `npm run build`; record the actual migration/provider configuration and results in `specs/007-student-management/quickstart.md` without claiming unavailable external-provider checks passed. Recorded in quickstart.md, Addendum Validation Results (2026-10-21).
- [ ] T045 Walk through quickstart address and eligibility scenarios with an active Admin on a disposable PostgreSQL database; verify both locality types and District filtering, migration preservation of existing Upazila references, below/equal/above minimum CGPA from published results, unavailable-source behavior, and re-evaluation after result publication; record outcomes in `specs/007-student-management/quickstart.md`. **Status:** not run; the published-result scenarios also need the academic-results owner (T022).

## Addendum Dependencies & Execution Order

- **US1 address addendum (T029–T036)** depends on the existing Configuration Upazila/City Corporation catalogs and the implemented Student profile/address flow. T029–T031 target separate test files and can be authored in parallel. T032 precedes T033; T034/T035 depend on the locality lookup shape; T036 follows the request/response shape. T030 also validates the Configuration deletion dependency.
- **US3 CGPA addendum (T037–T043)** depends on existing selected-Degree and admission identifier reads. T037 and T038 can be authored in parallel. T039/T040 establish provider and Degree reference shapes before T041; T042 follows the service response; T043 follows the response contract. Actual owner integration remains T022/T026-blocked, while fallback behavior is testable with fakes.
- **Validation (T044–T045)** follows both addenda. PostgreSQL migration tests do not require the academic-results owner; positive end-to-end result-source tests require T022's external contract.

## Parallel Example: Student Addenda

```text
Before implementation:
- Worker A: T029 Student address API tests in StudentAdmissionTests.cs
- Worker B: T030 PostgreSQL address migration and dependent-delete tests
- Worker C: T031 Angular locality-selection tests in student-form.spec.ts
- Worker D: T037 eligibility service/API tests in GraduationEligibilityTests.cs
- Worker E: T038 Angular CGPA panel tests in graduation-eligibility.spec.ts

After test contracts stabilize:
- Address worker: T032–T036 in dependency order
- Eligibility worker: T039–T043 in dependency order
```

## Addendum Implementation Strategy

1. Deliver the US1 address locality addendum independently, preserving current Upazila data and verifying the City Corporation migration on PostgreSQL.
2. Deliver US3 threshold and unavailable-source handling using a fake provider; do not block that work on the academic-results owner.
3. Complete T022/T026 only after the owner publishes the admission-keyed results contract, then run the positive integration and manual published-result refresh scenario.

## Addendum Format Validation

New task lines continue sequential IDs T029–T045, use `[US1]` or `[US3]` only in story phases, use `[P]` only for independent tests, and include explicit repository paths. Existing task IDs and completion states are preserved; T022 now describes both published-results values, while T026 remains open for its existing results-owner contract prerequisite.

## Phase 12: Convergence

- [ ] T046 When the academic-results owner publishes its contract, extend the existing T026 follow-up and record the owner-provided current cumulative CGPA alongside passed-course credits in `specs/007-student-management/research.md`, `specs/007-student-management/contracts/student-management-api.md`, and `specs/007-student-management/quickstart.md`; verify the published admission CGPA is compared to the Degree minimum without Student Management calculation or persistence per FR-006 and FR-012 (partial).