# Tasks: Configuration

**Input**: Design documents from `/specs/003-configuration/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Testing**: Automated tests are included because the project constitution requires appropriate unit, API, authorization, and data-integrity coverage.

**Organization**: Tasks are grouped by the four P1 user stories. The active Admin policy and business-audit transaction rule apply to every mutation from the first delivered story.

## Phase 1: Setup

**Purpose**: Establish the feature slice in the existing service, frontend, and test projects without adding packages or projects.

- [X] T001 Create Configuration feature scaffolding under `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/`, `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/`, and `iit-academic-portal/src/app/features/configuration/`.

---

## Phase 2: Foundational

**Purpose**: Shared policies required by catalog, batch, and geography implementations.

- [X] T002 Implement shared name and optional-code normalization in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/NameNormalizer.cs`: trim, collapse repeated whitespace, compare using invariant case normalization, preserve trimmed display values, and treat whitespace-only optional codes as absent; cover these rules in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/NameNormalizerTests.cs`.
- [X] T003 Define allowlisted `configuration.{resource}.{created|updated|deactivated|deleted}` business event definitions and metadata constraints in `iit-academic-portal-service/src/IitAcademicPortal.Application/Auditing/ConfigurationAuditEventDefinitions.cs`.

**Checkpoint**: Shared normalization and audit event definitions are available before user-story implementation.

---

## Phase 3: User Story 1 - Maintain Academic Reference Catalogs (Priority: P1)

**Goal**: Admins independently maintain educational institutions, boards, groups, examination types, degrees, semesters, Academic Sessions, designations, and halls.

**Independent Test**: As active Admin, create, read, search, update, deactivate referenced, and delete unreferenced entries in all nine catalogs; verify required fields, fixed institution types, normalized uniqueness, optional catalog codes, Admin authorization, and audit events without creating student or teacher records.

### Implementation

- [X] T004 [P] [US1] Add `EducationalInstitution` (`Name`, `NormalizedName`, fixed `Type` values School/College/University, nullable `Code`/`NormalizedCode`, nullable `Address`, `IsActive`) and `EducationBoard` (`Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, `IsActive`) entities in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/EducationalInstitution.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/EducationBoard.cs`.
- [X] T005 [P] [US1] Add `EducationGroup` (`Name`, `NormalizedName`, `IsActive`) and `ExaminationType` (`Name`, `NormalizedName`, `IsActive`) entities in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/EducationGroup.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/ExaminationType.cs`.
- [X] T006 [P] [US1] Add `Degree` (`Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, `IsActive`) and `Semester` (`Name`, `NormalizedName`, `IsActive`) entities in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Degree.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Semester.cs`.
- [X] T007 [P] [US1] Add `AcademicSession` (`Name`, `NormalizedName`, `IsActive`, distinct from authentication `AuthSession`) and `Designation` (`Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, `IsActive`) entities in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/AcademicSession.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Designation.cs`.
- [X] T008 [US1] Map the eight original catalog entities in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs`: unique `NormalizedName` per catalog, unique non-null `NormalizedCode` where codes exist, globally unique Academic Session `NormalizedName`, and restrictive delete behavior; add the EF migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`.
- [X] T009 [US1] Implement catalog create/read/list/search/update/deactivate/delete-if-unreferenced application operations and field-keyed validation in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/CatalogService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/CatalogContracts.cs`; normalize names/codes, reject duplicates, use `AdminModule`, and stage allowlisted audit events in the same unit of work.
- [X] T010 [US1] Expose the catalog collection/item routes from `contracts/configuration-api.md` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/CatalogsController.cs` and define request/response DTOs in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/Configuration/CatalogContracts.cs`; return 204 for unreferenced deletes and 409 for referenced records.
- [X] T011 [US1] Implement responsive catalog lists and create/edit forms for the catalogs using shared IIT styles in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-list.ts`, `iit-academic-portal/src/app/features/configuration/catalogs/catalog-list.html`, `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.ts`, and `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.html`.
- [X] T012 [US1] Add xUnit coverage for catalog CRUD, fixed institution types, normalized name/code conflicts, independent catalog namespaces, deactivation, and delete-if-unreferenced behavior in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CatalogCrudTests.cs`.
- [X] T013 [US1] Add Vitest coverage for catalog loading, empty/error states, search, field validation, and create/edit flows in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-list.spec.ts` and `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.spec.ts`.

**Checkpoint**: All independent catalogs can be maintained and tested without Batch, geography, student, or teacher workflows.

---

## Phase 4: User Story 2 - Configure Academic Batches (Priority: P1)

**Goal**: Admins manage Batches with required Degree, starting Semester, and Academic Session references; the same normalized Batch Name is allowed under different Degrees.

**Dependencies**: US1 supplies Degree, Semester, and Academic Session catalogs.

**Independent Test**: Create and update Batches with all three active references; verify missing/inactive references fail, Session may be shared, normalized names are unique within a Degree regardless of Session, and Batch operations do not assign students or advance semesters.

### Implementation

- [X] T014 [US2] Add the Batch entity in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Batch.cs` with required `Name`, `NormalizedName`, `DegreeId`, `StartingSemesterId`, required `AcademicSessionId`, and `IsActive`.
- [X] T015 [US2] Configure Batch foreign keys and unique index (`DegreeId`, `NormalizedName`) in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs`; require all three active references and use restrictive delete behavior, then add the migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`.
- [X] T016 [US2] Implement Batch create/read/list/search/update/deactivate/delete-if-unreferenced operations in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/BatchService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/BatchContracts.cs`; require a non-empty normalized `Name`, Degree, starting Semester, and Academic Session, and scope normalized-name uniqueness to Degree only.
- [X] T017 [US2] Implement Batch routes and DTO mapping from `contracts/configuration-api.md` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/BatchesController.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/Configuration/BatchContracts.cs`, using active-Admin authorization and field-keyed validation errors.
- [X] T018 [US2] Implement the Batch list/form at `iit-academic-portal/src/app/features/configuration/batches/batch-list.ts`, `iit-academic-portal/src/app/features/configuration/batches/batch-list.html`, `iit-academic-portal/src/app/features/configuration/batches/batch-form.ts`, and `iit-academic-portal/src/app/features/configuration/batches/batch-form.html`; require Degree, starting Semester, and Session selectors.
- [X] T019 [US2] Add xUnit tests for required/missing/inactive references, shared Sessions, normalized Batch Name uniqueness per Degree across Sessions, CRUD, deletion conflicts, and no student/progression side effects in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/BatchTests.cs`.
- [X] T020 [US2] Add Vitest tests for required selectors, field errors, create/edit, and Batch list states in `iit-academic-portal/src/app/features/configuration/batches/batch-form.spec.ts` and `iit-academic-portal/src/app/features/configuration/batches/batch-list.spec.ts`.

**Checkpoint**: Batch CRUD works with mandatory active references and the per-Degree Name rule.

---

## Phase 5: User Story 3 - Maintain Bangladesh Geographic Reference Data (Priority: P1)

**Goal**: Admins maintain and import Division → District → Upazila reference data with no code fields and parent-scoped normalized-name uniqueness.

**Dependencies**: Shared normalization/audit foundation only; this story can run in parallel with US1 after Phase 2.

**Independent Test**: Validate the owner-provided JSON and parent relationships; verify a valid import commits all rows atomically, invalid data writes nothing, names are unique at global/parent scope, deletes with dependents return 409, and there are no geographic code fields.

### Implementation

- [X] T021 [P] [US3] Add `Division`, `District`, and `Upazila` entities with `Name`, `NormalizedName`, `IsActive`, and their required parent IDs, with no code fields, in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Division.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/District.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Upazila.cs`.
- [X] T022 [US3] Map geographic relationships in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs`: unique Division `NormalizedName` globally; unique District (`DivisionId`, `NormalizedName`); unique Upazila (`DistrictId`, `NormalizedName`); restrictive parent deletes; no code columns.
- [X] T023 [US3] Add the geographic tables, indexes, and foreign keys through an EF migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`.
- [X] T024 [US3] Implement complete-document seed validation and all-or-nothing import in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/GeographySeedValidator.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/GeographyImportService.cs`; validate source citation/version, required names, normalized uniqueness, and every parent link before writing, require empty target tables, and never overwrite Admin-managed data.
- [X] T025 [US3] Implement Division/District/Upazila CRUD and `POST /api/admin/configuration/geography/import/validate` plus `/import` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/GeographyController.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/Configuration/GeographyContracts.cs`; use active-Admin authorization, anti-forgery, field errors, and 409 for referenced deletes.
- [X] T026 [US3] Implement geography management lists/forms with cascading parent selection in `iit-academic-portal/src/app/features/configuration/geography/division-list.ts`, `iit-academic-portal/src/app/features/configuration/geography/district-list.ts`, `iit-academic-portal/src/app/features/configuration/geography/upazila-list.ts`, and their `.html` templates.
- [X] T027 [US3] Add xUnit tests for name-scope uniqueness, no geographic code fields, invalid/cross-branch parents, validate-only no-write behavior, invalid-document atomicity, successful import audit, and dependent-delete conflicts in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GeographyTests.cs` and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GeographyImportTests.cs`.
- [X] T028 [US3] Add Vitest tests for cascading Division → District → Upazila selection and loading/empty/error/inactive states in `iit-academic-portal/src/app/features/configuration/geography/geography-forms.spec.ts`.

**Checkpoint**: Geographic CRUD and Admin-authorized seed import preserve the full hierarchy and never write partial or invented data.

---

## Phase 6: User Story 4 - Protect Configuration Changes and References (Priority: P1)

**Goal**: Only active Admin sessions can use the Configuration menu, route, or mutation endpoints; every successful change is audited atomically.

**Dependencies**: Cross-cutting verification depends on resource operations from US1, US2, and US3. Every story’s mutation endpoint must enforce `AdminModule` and stage its business audit event as it is implemented.

**Independent Test**: For each resource, verify anonymous requests receive 401, non-Admin active roles receive 403, the Configuration menu and route are absent/denied for non-Admins, successful writes create safe audit events in the same transaction, audit failure rolls back writes, and delete never cascades.

### Implementation

- [X] T029 [US4] Add `/admin/configuration` under the active-Admin `roleArea('Admin')` routes in `iit-academic-portal/src/app/app.routes.ts` and add the Admin-only “Configuration” menu item in `iit-academic-portal/src/app/layout/shell/shell.ts`.
- [X] T030 [US4] Add Angular tests for Configuration menu visibility, active-route state, direct non-Admin navigation denial, and Admin access in `iit-academic-portal/src/app/layout/shell/shell.spec.ts` and `iit-academic-portal/src/app/app.routes.spec.ts`.
- [X] T031 [US4] Add API authorization integration tests for anonymous 401 and non-Admin 403 across catalog, Batch, geography CRUD, and both geography import endpoints in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationAuthorizationTests.cs`.
- [X] T032 [US4] Verify successful catalog, Batch, geographic, and seed-import operations stage allowlisted events through `IAuditEventRecorder` and save them with data changes in the same unit of work; cover atomic success and audit-failure rollback in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationAuditTransactionTests.cs`.
- [X] T033 [US4] Add PostgreSQL integration tests proving unreferenced deletes succeed and referenced Degree, Semester, Session, Division, and District deletes return conflict without cascade/orphaning in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationDeleteIntegrityTests.cs`.
- [X] T034 [US4] After the owner supplies the approved geography JSON and source/version citation, run the Admin-authorized validate/import flow and record source/version, counts, and validation outcome in `specs/003-configuration/quickstart.md`.

**Checkpoint**: Authorization, audit atomicity, and reference-safe deletion hold across all Configuration resources.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Verify provider-specific behavior, accessibility, and end-to-end readiness.

- [X] T035 [P] Add PostgreSQL 18 migration/index integration coverage for globally unique catalog and Session names, optional unique codes, Batch (`DegreeId`, `NormalizedName`), geography parent-scope names, and restrictive foreign keys in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationPostgresTests.cs`.
- [X] T036 [P] Review and update `specs/003-configuration/quickstart.md` against the final Configuration UI/API routes, owner seed prerequisite, migration commands, and active-Admin validation scenarios.
- [X] T037 Run `dotnet test` from `iit-academic-portal-service/` and `npm test -- --watch=false` plus `npm run build` from `iit-academic-portal/`; record results in `specs/003-configuration/quickstart.md`.

---

## Phase 8: Addendum - Halls (added 2026-10-14)

**Purpose**: One more independent catalog, requested after the original scope: university residential halls with a required name, optional code, and optional address (FR-024). It reuses every rule of the other catalogs, so it is a small slice across the same layers.

- [X] T038 [US1] Add the `Hall` entity in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Hall.cs` and map it (`Halls` table, unique `NormalizedName`, unique non-null `NormalizedCode`, address limited to 500) in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs`; add the `AddHalls` migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`.
- [X] T039 [US1] Add `HallResource` (`halls`, optional code, optional address, shared address reader) in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResources.cs`, register it in `CatalogService.cs`, and allow the route in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/CatalogsController.cs`; audit events `configuration.hall.{created|updated|deactivated|deleted}` come from the shared flow.
- [X] T040 [US1] Add the `halls` descriptor (group "Other catalogs", code and address) in `iit-academic-portal/src/app/features/configuration/configuration.models.ts`; the generic list and form serve it.
- [X] T041 [US1] Extend the xUnit coverage with Halls in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CatalogCrudTests.cs`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationAuthorizationTests.cs`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationAuditTransactionTests.cs`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationPostgresTests.cs`, and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationDeleteIntegrityTests.cs` (plus hall-specific address tests), and the Vitest coverage in `iit-academic-portal/src/app/features/configuration/configuration-home.spec.ts` and `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.spec.ts`.
- [X] T043 [US1] Add `iit-academic-portal-service/database/postgresql/seed-halls.sql` (the 23 University of Dhaka halls and hostels, fixed version 5 UUIDs, default address "University of Dhaka, Dhaka-1000, Bangladesh", `ON CONFLICT DO NOTHING`) and its PostgreSQL tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/HallSeedScriptTests.cs` (fixed ids, normalized names match the application rule, idempotent, keeps admin edits, skips an existing same-named hall).
- [X] T044 [US3] Add the geography seed scripts `iit-academic-portal-service/database/postgresql/seed-divisions.sql`, `seed-districts.sql`, and `seed-upazilas.sql` (8, 64, and 492 rows from the owner's files; fixed version 5 UUIDs; parents found by normalized name; `ON CONFLICT DO NOTHING`; a missing parent stops the script and adds nothing) and their PostgreSQL tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GeographySeedScriptTests.cs`.
- [X] T045 [US4] Make every configuration delete a soft delete: add `IsDeleted`, `DeletedAtUtc`, `DeletedByUserId` and `MarkDeleted` in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/ConfigurationRecord.cs`; map them with a global query filter and live-rows-only unique indexes in `PortalDbContext.cs`; add the `SoftDeleteConfiguration` migration; remove the physical `Remove` from `IConfigurationStore`/`ConfigurationStore` and add `ExistsAsync`.
- [X] T046 [US4] Change the delete flow in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResource.cs` and `ConfigurationResources.cs`: refuse (409) while live dependents exist (Degree, Semester, Academic Session by Batches; Division by Districts; District by Upazilas), otherwise mark the record deleted with the clock and the signed-in actor, stage the `deleted` audit event, and save them together.
- [X] T047 [US4] Add the PostgreSQL and API tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationDeleteIntegrityTests.cs` (rewritten: foreign keys still refuse a physical delete, deleted rows are marked and hidden, names and codes reusable, scoped names reusable, unique indexes cover live rows only), `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CatalogCrudTests.cs`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/BatchTests.cs`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GeographyTests.cs`, `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GeographyImportTests.cs`, and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/HallSeedScriptTests.cs` (a seeded hall an Admin deleted stays deleted).
- [X] T048 [US4] Update the delete wording in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.html` and its spec, and record the decision in the spec (FR-016, clarification 2026-10-14), data model, research, contract, design system, and service README.
- [X] T049 [US4] Add the shared `iit-academic-portal-service/src/IitAcademicPortal.Domain/AuditedEntity.cs` (created by, created time, last modified by, last modified time) and derive `ConfigurationRecord` from it.
- [X] T050 [US4] Stamp the columns on every save: add `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/AuditedEntityInterceptor.cs` (server clock and signed-in account), register it in `PersistenceServiceCollectionExtensions.cs`, make `IAuditRequestContext` a singleton in `PortalApi.cs`, map the four columns for every derived entity in `PortalDbContext.cs`, and add the `AuditedEntityColumns` migration (existing rows get the migration time and no account).
- [X] T051 [US4] Return the values from the Configuration API: `createdAtUtc`, `createdByUserId`, `createdBy`, `lastModifiedAtUtc`, `lastModifiedByUserId`, `lastModifiedBy` (emails resolved with one lookup per response in `CatalogService.cs`), and show a "Record history" card on the edit screen in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.html`.
- [X] T052 [US4] Add the tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/AuditedEntityStampTests.cs` (create, update, deactivate, delete, no-op, server clock, client cannot supply, removed account, outside a request), `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/AuditedEntityConventionTests.cs` (every entity derives or is exempt with a reason; columns mapped as required), `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/AuditedEntityPostgresTests.cs` (columns, defaults, a SQL-added row), and the Vitest cases in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.spec.ts`.
- [X] T053 [US4] Write the convention for future entities in `iit-academic-portal-service/README.md` ("Entity conventions"), `specs/design-system.md` ("Record history"), `specs/003-configuration/spec.md` (FR-025), `specs/003-configuration/data-model.md`, `specs/003-configuration/research.md`, and `specs/003-configuration/contracts/configuration-api.md`; note the defaults in `iit-academic-portal-service/database/postgresql/seed-halls.sql`, `iit-academic-portal-service/database/postgresql/seed-divisions.sql`, `iit-academic-portal-service/database/postgresql/seed-districts.sql`, and `iit-academic-portal-service/database/postgresql/seed-upazilas.sql`.
- [X] T054 [US3] Re-check the seed scripts against soft delete and the audit columns and fix what they missed: the district and upazila scripts looked parents up with plain SQL and so could match a deleted division or district (the application hides deleted rows through a query filter that raw SQL bypasses). They now match only parents that are not deleted, treat a row as present by its fixed id or by its live name, and stop with a clear error if a parent is missing or deleted; the script headers say that deleted rows stay deleted and that the audit columns take the database defaults. Tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GeographySeedScriptTests.cs` cover deleted rows staying deleted on a re-run, a live parent chosen over a deleted one of the same name, and a deleted division being refused.
- [X] T042 [US1] Update `specs/003-configuration/spec.md`, `specs/003-configuration/data-model.md`, `specs/003-configuration/contracts/configuration-api.md`, `specs/003-configuration/plan.md`, `specs/003-configuration/research.md`, `specs/003-configuration/quickstart.md`, `specs/003-configuration/implementation-notes.md`, `iit-academic-portal/README.md`, and `iit-academic-portal-service/README.md` for Halls.

**Checkpoint**: Halls can be maintained like every other independent catalog, with no relationship to a student, teacher, or other resource.

---

## Phase 9: Degree Credits Addendum - User Story 1 (Priority: P1)

**Goal**: Maintain a required positive credit total on each Degree for Student Management graduation-eligibility calculations.

**Independent Test**: As an active Admin, create and edit a Degree with a positive `Credits to be completed` amount; verify the value persists and appears in responses. Reject a missing, zero, negative, or more-than-two-decimal value without changing the Degree. Verify non-Degree catalogs do not accept this field and existing Degree rows are migrated only from approved values.

### Tests for Degree Credits

- [X] T055 [P] [US1] Add API tests for Degree `creditsToBeCompleted` create/read/update, field-keyed rejection of missing, zero, negative, and over-precision values, and rejection of the field on non-Degree catalogs in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CatalogCrudTests.cs`.
- [X] T056 [P] [US1] Add Angular tests for the Degree-only required numeric field, positive values with up to two fractional digits, edit-value loading, request serialization, and validation errors in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.spec.ts`.
- [X] T057 [P] [US1] Add PostgreSQL tests for the Degree credit column's `numeric(8,2)` storage, required/positive constraints, and migration against an empty Degree table in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationPostgresTests.cs`. The assertions are authored; execution awaits `AUDIT_TEST_POSTGRES`.
- [X] T058 [P] [US1] Confirm whether the target database contains Degree rows; the owner confirmed the Degree table is empty on 2026-10-18, recorded in `specs/003-configuration/quickstart.md`. Do not record any credit totals in source or spec files.

### Implementation for Degree Credits

- [X] T059 [US1] Add `CreditsToBeCompleted` to `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Degree.cs` as a required positive decimal with at most two fractional digits, and map it in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs` as `numeric(8,2)` with required and positive-value enforcement.
- [X] T060 [US1] Generate a versioned EF migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/` that adds the required positive Degree credit column without a default or embedded totals, relying on the T058-confirmed empty table. If a target contains rows, stop and backfill through a deployment-only owner-approved process before enforcing the required constraint.
- [X] T061 [US1] Extend `ConfigurationInput`, `ConfigurationItem`, and the Degree-specific configuration resource in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationModels.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResource.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResources.cs` to require a positive credit total with at most two fractional digits for Degrees and reject the field for other resources.
- [X] T062 [US1] Add `creditsToBeCompleted` to Degree-only request/response mapping in `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/Configuration/CatalogContracts.cs`, preserving omission from non-Degree responses and field-keyed validation errors.
- [X] T063 [US1] Add the Degree credit field to `ConfigurationItem`, the Degree resource descriptor, and the generic catalog form in `iit-academic-portal/src/app/features/configuration/configuration.models.ts`, `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.ts`, and `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.html`; show a required numeric input with a positive minimum and 0.01 step only for Degrees.
- [X] T064 [US1] Update the Degree API contract and `specs/003-configuration/quickstart.md` with the required credit field, invalid-value cases, PostgreSQL migration/backfill prerequisite, and validation commands/results in `specs/003-configuration/contracts/configuration-api.md` and `specs/003-configuration/quickstart.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No new projects or packages; establish only the feature slice directories.
- **Foundational (Phase 2)**: Depends on Setup; shared name normalization and event definitions block all catalog writes.
- **User Stories (Phases 3–6)**: Start after Foundational. US1 and US3 can proceed in parallel. US2 depends on US1 because Batches reference Degree, Semester, and Academic Session. US4 is cross-cutting and completes after US1–US3; Admin authorization and audit enforcement are still required within every story from its first endpoint.
- **Polish (Phase 7)**: Depends on all four stories; approved geography dataset verification additionally depends on owner delivery.
- **Degree Credits Addendum (Phase 9)**: Extends US1 after the existing catalog work. Credit tests and data preparation precede implementation; the migration depends on an owner-approved mapping for every existing Degree row or confirmation that no Degree rows exist.

### User Story Dependencies

- **US1 (P1)**: Independent after Foundational; establishes academic catalogs, including Academic Session. The Degree Credits Addendum extends this story and depends on the existing Degree catalog and approved existing-row credit mapping.
- **US2 (P1)**: Depends on US1 for Degree, Semester, and Academic Session references.
- **US3 (P1)**: Independent after Foundational; geographic seed is an external owner input for production-source verification.
- **US4 (P1)**: Cross-cutting verification after US1, US2, and US3. Security and audit implementation are not deferred to this phase.

### Parallel Opportunities

- T004–T007 create independent entity files and can run in parallel after T002.
- US1 and US3 can proceed in parallel after Foundational because their entities and routes are independent.
- Within US2 and US3, frontend forms and backend API work can proceed in parallel after their DTO/contract shapes are agreed.
- T035 and T036 can run independently after story completion.
- Degree-credit API, Angular, and PostgreSQL tests (T055–T057) and existing-degree mapping preparation (T058) use separate files and can proceed in parallel; the migration must wait for both the model and approved mapping.

## Parallel Example: User Stories 1 and 3

```text
After T002–T003 complete:
- Worker A: US1 catalog entity/mapping tasks T004–T008
- Worker B: US3 geography entity/mapping tasks T021–T023
After model/contract shapes are stable:
- Worker A: US1 catalog API/service/UI tasks T009–T013
- Worker B: US3 importer/API/UI tasks T024–T028
```

## Parallel Example: Degree Credits Addendum

```text
Task T055: API validation tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CatalogCrudTests.cs
Task T056: Angular Degree-form tests in iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.spec.ts
Task T057: PostgreSQL Degree-credit tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationPostgresTests.cs
Task T058: Existing-Degree credit mapping in iit-academic-portal-service/database/postgresql/degree-credit-requirements.json
```

## Implementation Strategy

### MVP First (User Story 1)

1. Complete Setup and Foundational tasks.
2. Complete US1 catalog CRUD, validation, active-Admin authorization, audit, and tests.
3. Stop and validate US1 independently using its acceptance scenarios before starting dependent Batch work.

### Incremental Delivery

1. Complete US1 catalogs and Academic Session; validate and deliver independently.
2. Add US2 Batch management with required references; verify per-Degree name uniqueness.
3. Add US3 geography management and Admin-authorized seed import; production-source verification waits for the owner file.
4. Complete US4 cross-cutting authorization/audit verification, then run PostgreSQL and frontend/service gates.
5. Complete the Degree Credits Addendum using owner-approved values for existing Degrees, then validate Degree create/update and migration behavior.

### Format Validation

Every task line uses `- [ ] T###`, optional `[P]`, and a required `[US#]` label for user-story tasks. Setup, Foundational, and Polish tasks have no story label. Task IDs are sequential, and every task description names its target source, test, or documentation path.

## Phase 10: Convergence

- [ ] T065 Obtain and validate the owner-approved official geographic dataset and source/version, then reconcile the currently imported Division/District/Upazila data and record verified source fidelity in `specs/003-configuration/quickstart.md` per FR-014, SC-005, and US3/AC-5 (partial).
- [ ] T066 Run the Configuration PostgreSQL 18 integration tests with `AUDIT_TEST_POSTGRES` against an isolated empty database, verify `DegreeCredits` applies and its storage constraints pass, and record the result in `specs/003-configuration/quickstart.md` per FR-005 and the plan's provider-specific migration testing decision (partial).
