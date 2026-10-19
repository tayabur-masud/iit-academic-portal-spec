---
description: "Implementation tasks for Course Management"
---

# Tasks: Course Management

**Input**: Design documents from `specs/008-course-management/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/courses-api.md`, `quickstart.md`

**Tests**: Automated tests are included because the project constitution requires coverage for business-critical, authorization, audit, and persistence rules.

**Organization**: Tasks are grouped by user story. Every implementation task is independently testable where practical.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure.

No setup tasks are needed: the existing Angular app, ASP.NET Core service, and test projects are already initialized.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Add shared catalog capabilities required before Course-specific behavior can be implemented.

- [X] T001 [P] Add per-resource update-conflict and delete-capability hooks to `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResource.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/ConfigurationControllerBase.cs` so Course can reject deactivation with 409 and opt out of DELETE without changing other catalogs.
- [X] T002 [P] Add a `degreeId` filter from query binding through the configuration query and store to support Degree-scoped course lists in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/Configuration/CatalogContracts.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationModels.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/ConfigurationStore.cs`.

**Checkpoint**: Existing catalog resources retain their behavior; Course-specific lifecycle and Degree-filter hooks are available.

---

## Phase 3: User Story 1 - Maintain the Course Catalog (Priority: P1) [MVP]

**Goal**: Let active Admins maintain audited Courses under Degrees with valid credits, globally unique codes, and safe optional prerequisites.

**Independent Test**: In the Admin UI and API, create, view, search, update, and deactivate Courses in two Degrees. Verify required fields, both optional credit components and their sum rule, global code uniqueness, valid prerequisite rules and cycle rejection, dependent-course deactivation conflict, audit history, Admin-only access, and no changes to enrollment or result records.

### Tests for User Story 1

> Write the tests first and confirm they fail for the missing Course behavior before implementing it.

- [X] T003 [P] [US1] Add application-level tests for required code/type/Degree, positive two-decimal total credit, optional theory/lab values between zero and total inclusive with no more than two decimal places, matching sums when both components are present, accepting different Courses with the same name when their codes differ, same-Degree active prerequisites, self/cycle rejection, rejecting Degree changes that invalidate this Course's or an active dependent's prerequisite without modifying affected Courses, and dependent-course deactivation in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CourseCatalogTests.cs`.
- [X] T004 [P] [US1] Add HTTP contract tests for Course list filtering, create/read/update response fields, field-keyed errors for a component over total, rejection of Degree changes that invalidate affected prerequisites without partial updates, 409 dependent-deactivation behavior, absence of DELETE, and before/after assertions that Course create/update/deactivate leave seeded `StudentAdmissions` and `StudentAcademicHistory` records unchanged in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CourseCatalogApiTests.cs`.
- [X] T005 [P] [US1] Add active-Admin authorization and atomic audit-rollback coverage for Course requests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationAuthorizationTests.cs` and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationAuditTransactionTests.cs`.
- [X] T006 [P] [US1] Add Angular form and list tests for Course fields, optional Degree-filtered prerequisite choices, rejecting a supplied theory/lab value greater than total, server errors for invalidating Degree changes, server field errors, credit display, and deactivation conflict in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.spec.ts` and `iit-academic-portal/src/app/features/configuration/catalogs/catalog-list.spec.ts`.
- [X] T007 [P] [US1] Add PostgreSQL integration assertions for Course numeric/check constraints, global live-code uniqueness, restrictive Degree/self-reference FKs, and prerequisite indexes in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationPostgresTests.cs`.

### Implementation for User Story 1

- [X] T008 [P] [US1] Create the audited Course entity in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Course.cs` with required `name`, `code`, `creditHours`, `type`, and `degreeId`; nullable `theoryCreditHours`, `labCreditHours`, and `prerequisiteCourseId`; active state; and inherited audit fields. Enforce the data-model rules verbatim: `creditHours` must be greater than zero and have no more than two decimal places; each supplied theory/lab value must be between zero and `creditHours`, inclusive, and have no more than two decimal places; type is exactly `Mandatory` or `Elective`; code is unique across all non-deleted Courses, including inactive ones; prerequisite is a different active Course in the same Degree; no name-uniqueness rule is added.
- [X] T009 [P] [US1] Extend Angular configuration item/request types, resource descriptors, reference metadata, and Degree filter choices for Course in `iit-academic-portal/src/app/features/configuration/configuration.models.ts`; preserve the exact optionality of theory, lab, and prerequisite fields and the exact type values `Mandatory` and `Elective`.
- [X] T010 [US1] Map Course storage in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs` and add the generated EF Core migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`. Persist `creditHours`, `theoryCreditHours`, and `labCreditHours` as precision `(8,2)`; enforce `creditHours > 0`, each supplied component between zero and total inclusive, and equal total when both components are supplied; add a partial unique normalized-code index over non-deleted rows, Degree/prerequisite indexes, restrictive FKs, and the required enum mapping.
- [X] T011 [US1] Add only the Course-specific prerequisite-chain and active-dependent queries to `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IConfigurationStore.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/ConfigurationStore.cs`; avoid loading the full Course catalog to validate one prerequisite chain.
- [X] T012 [US1] Implement and register the Course application resource in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResources.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResource.cs`. Override the shared catalog's global name-uniqueness rule for Course so different codes may share a name. Require globally unique code, valid active Degree, positive total and optional components between zero and total inclusive, same-Degree active non-self prerequisite, acyclic chains, and rejection of a Degree change if it would invalidate this Course's prerequisite or any active dependent Course's prerequisite, without modifying affected Courses; retain conflict on deactivation while any active Course depends on it. Include Course-specific fields in response projection and audit snapshots.
- [X] T013 [US1] Add Course request/response fields, the `courses` catalog route, Degree-filter mapping, and per-resource DELETE opt-out in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/Configuration/CatalogContracts.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/CatalogsController.cs`; retain shared Admin/anti-forgery policy, paging, 400 field errors, and 409 conflict responses.
- [X] T014 [US1] Add conditional Course form controls, optional prerequisite selection filtered to active Courses in the selected Degree, self-exclusion when editing, client-side rejection of supplied theory/lab values greater than total, and clear server conflict/error feedback for Degree changes that invalidate prerequisite relationships in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.ts` and `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.html`.
- [X] T015 [US1] Display Course code, type, Degree, credit total, and active status in the existing searchable/paged catalog list in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-list.ts` and `iit-academic-portal/src/app/features/configuration/catalogs/catalog-list.html`.

**Checkpoint**: US1 works end-to-end independently; valid catalog operations succeed and invalid relationships or unauthorized requests fail without partial writes.

---

## Phase 4: Polish & Cross-Cutting Concerns

**Purpose**: Validate the implementation against the end-to-end scenarios and regression suite.

- [ ] T016 Run the automated and manual scenarios in `specs/008-course-management/quickstart.md`; resolve failures, verify all new application/UI/PostgreSQL coverage passes, and confirm the Course migration applies without modifying enrollment or result tables. **Status (partial):** every automated suite, the PostgreSQL-gated tests, and the production build pass (results in quickstart.md), and the migration was applied to the PostgreSQL test databases; the manual browser scenarios have not been run, so this stays open until they are.
- [ ] T017 Run the SC-006 usability validation by giving the same Course-creation task to 10 active Admin participants who have not previously used Course Management, without assistance; record each completion time and whether assistance was needed in `specs/008-course-management/usability-results.md` and pass only if at least 9 create a valid Course within three minutes. **Status:** not run; it needs real Admin participants.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No code changes; existing app, service, and test projects are used.
- **Foundational (Phase 2)**: T001 and T002 are independent shared-catalog changes; both must finish before US1 tests and implementation.
- **User Story 1 (Phase 3)**: Depends on T001 and T002. It is the only user story and can be implemented independently after the foundation.
- **Polish (Phase 4)**: Depends on all US1 implementation and tests.

### User Story Dependencies

- **User Story 1 (P1)**: Depends only on Phase 2; no dependency on another story.

### Within User Story 1

- Author and run T003-T007 first; the tests should fail until implementation is complete.
- T008 and T009 can proceed in parallel after Phase 2 and the test tasks.
- T010 and T011 depend on T008; T012 depends on T008 and T011.
- T013 depends on T001, T002, and T012; T014 and T015 depend on T009 and T013.
- T016 runs after all implementation and test tasks; T017 runs after the Course UI is complete and records the SC-006 result.

### Parallel Opportunities

- Phase 2: T001 and T002 touch separate shared-catalog boundaries.
- Test authoring: T003, T004, T005, T006, and T007 target distinct application/API/security/UI/PostgreSQL test files and can be authored in parallel.
- Initial implementation: T008 (backend domain entity) and T009 (Angular resource model/descriptor) touch separate projects and can proceed in parallel.

## Parallel Example: User Story 1

```text
After Phase 2, author the independent tests in parallel:
T003 application Course rules
T004 HTTP catalog contract
T005 authorization and audit transaction
T006 Angular catalog form/list
T007 PostgreSQL schema constraints

After tests are in place, implement in dependency order:
T008 Course entity and T009 Angular models/descriptors in parallel
T010 persistence and T011 prerequisite queries after T008
T012 application resource after T008 and T011
T013 API contract/route after T012
T014 and T015 Angular screens after T009 and T013
T017 usability validation after the UI and end-to-end flow are ready
```

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 2 shared-catalog prerequisites.
2. Author the focused US1 test suite.
3. Implement the entity, persistence/migration, prerequisite rules, REST contract, and Admin UI.
4. Complete Phase 4 validation; deliver US1 as the MVP.

### Incremental Delivery

The feature contains one user story. Deliver the catalog only after its create/read/update/deactivate flows, validation, authorization, audit, migration, and quickstart scenarios pass together. Do not add enrollment, teaching, or result behavior.

## Notes

- Every task has a sequential ID and at least one explicit repository path.
- `[P]` indicates distinct files/components with no dependency on unfinished work.
- `[US1]` maps to the sole user story in `spec.md`.
- EF Core migration files are generated under the path in T010; do not handwrite a timestamp placeholder.
- The Course API intentionally omits DELETE; use audited deactivation.

## Phase 5: Convergence

- [ ] T018 Make prerequisite validation fail closed when the 1,000-link traversal guard is reached without establishing that the chain is acyclic, and add a regression test for a cycle beyond the guard in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResources.cs` and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CourseCatalogTests.cs` per FR-004/SC-004 (partial).