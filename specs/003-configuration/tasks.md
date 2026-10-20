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
- [X] T066 Run the Configuration PostgreSQL 18 integration tests with `AUDIT_TEST_POSTGRES` against an isolated empty database, verify `DegreeCredits` applies and its storage constraints pass, and record the result in `specs/003-configuration/quickstart.md` per FR-005 and the plan's provider-specific migration testing decision. **Done 2026-10-20:** the PostgreSQL suite ran on fresh databases; `DegreeCredits` applies and its `numeric(8,2)`, required, and positive checks pass (quickstart.md, Degree Credits Addendum Status).

## Phase 11: City Corporation Addendum - User Story 3 (Priority: P1)

**Goal**: Add City Corporation as a code-free geographic catalog with exactly one District parent, preserving existing geography behavior and allowing optional owner-supplied seed records.

**Independent Test**: As an active Admin, create, view, search, filter by District, update, deactivate, and delete an unreferenced City Corporation. Verify required active District references, normalized name uniqueness within a District, same-name acceptance in another District, no code field, District deletion protection, import counts/atomicity, and no invented seed data.

### Tests for User Story 3 Addendum

> Add focused tests before implementation and verify they fail for the missing City Corporation behavior.

- [X] T067 [P] [US3] Add City Corporation API tests for required District, missing/nonexistent/inactive District rejection, parent-filtered lists, no code fields, normalized name uniqueness within a District and same-name acceptance across Districts, deactivate/delete behavior, and a District delete conflict while City Corporations remain in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GeographyTests.cs`.
- [X] T068 [P] [US3] Extend synthetic geography import tests to cover an optional `cityCorporations` array under each District, count/report fields, duplicate City Corporation names within a District, correct District links, target-empty checks, and all-or-nothing behavior in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GeographyImportTests.cs`.
- [X] T069 [P] [US3] Add PostgreSQL integration assertions for the `CityCorporations` migration, restrictive District foreign key, and live-row unique index on `(DistrictId, NormalizedName)` in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationPostgresTests.cs`.
- [X] T070 [P] [US3] Add Angular tests for the City Corporation list/form descriptor, District filtering, Division-to-District cascade, parent selection, and validation errors in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.spec.ts`, `iit-academic-portal/src/app/features/configuration/catalogs/catalog-list.spec.ts`, and `iit-academic-portal/src/app/features/configuration/configuration-home.spec.ts` (the generic catalog form and list serve the geography resources; there is no separate `geography/` folder).

### Implementation for User Story 3 Addendum

- [X] T071 [US3] Add `CityCorporation` in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/CityCorporation.cs` with inherited `Name`, `NormalizedName`, `IsActive`, soft-delete, and audit fields plus required `DistrictId`; it MUST have no code fields, MUST reference exactly one District, and its normalized name MUST be unique within CityCorporation records for that District.
- [X] T072 [US3] Map `CityCorporation` in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs` with a required restrictive District foreign key, a District index, and a partial unique index on `(DistrictId, NormalizedName)` for non-deleted rows; generate the additive EF Core migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`.
- [X] T073 [US3] Add and register `CityCorporationResource` in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResources.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/CatalogService.cs` with District-scoped name uniqueness, `districtId` filtering, no code field, and a required active District for create/reassignment while allowing an unchanged inactive District reference; extend `DistrictResource.HasDependentsAsync` so a live City Corporation blocks District deletion.
- [X] T074 [US3] Add `city-corporations` to the geography route allowlist in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/GeographyController.cs`; reuse the existing generic `districtId` request/query/response field and preserve active-Admin, anti-forgery, validation, audit, and lifecycle behavior.
- [X] T075 [US3] Extend the optional nested `cityCorporations` seed property and name-only schema in `specs/003-configuration/contracts/geographic-seed.schema.json`, and update `GeographySeedValidator`, `GeographyImportService`, and import report fields in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/GeographySeedValidator.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/GeographyImportService.cs`; validate per-District normalized uniqueness, include City Corporation counts, require its target table to be empty, and insert all records atomically without inventing seed rows.
- [X] T076 [US3] Add the `city-corporations` descriptor to `iit-academic-portal/src/app/features/configuration/configuration.models.ts` with a required District reference, `districtId` parent filter, and Division-to-District cascade; reuse the existing generic list/form and expose no code input.
- [X] T077 [US3] Update `specs/003-configuration/data-model.md`, `specs/003-configuration/contracts/configuration-api.md`, and `specs/003-configuration/quickstart.md` with the City Corporation entity, endpoint, optional seed shape, District-scoped uniqueness, lifecycle behavior, migration validation, and an explicit note that production seed acceptance awaits owner-approved source/version data.

**Checkpoint**: City Corporation CRUD, District integrity, the optional import extension, and shared Admin/audit behavior are independently verified without altering existing geographic rows or inventing seed data.

---

## Phase 12: City Corporation Validation & Cross-Cutting Concerns

**Purpose**: Validate the new geographic resource and preserve existing configuration behavior.

- [ ] T078 Run `dotnet test` and the focused PostgreSQL Configuration integration tests from `iit-academic-portal-service/`, plus `npm test -- --watch=false` and `npm run build` from `iit-academic-portal/`; execute the City Corporation quickstart scenarios in `specs/003-configuration/quickstart.md` with synthetic seed data and record results without claiming production source fidelity. **Status (partial):** every automated suite, the PostgreSQL-gated tests, and the production build pass, with synthetic import fixtures (results in quickstart.md, City Corporation Validation Results); the five quickstart scenarios have not been run by hand in a browser, so this stays open until they are.

---

## Phase 13: Degree Minimum CGPA Addendum - User Story 1 (Priority: P1)

**Goal**: Store a required minimum CGPA for completion on each Degree (greater than 0, at most 4.00, two decimals), shipped in the same migration as the City Corporation table. Graduation eligibility does not use it yet.

**Independent Test**: As an active Admin, create and edit a Degree with a minimum CGPA; verify it persists and is returned. Reject a missing, zero, negative, over-4.00, or over-precision value without changing the Degree. Verify other catalogs refuse the field and that a Degree created before the migration receives 2.50.

- [X] T079 [P] [US1] Add API tests for Degree `minimumCgpaForCompletion` create/read/update, the bounds 0.01 and 4.00, rejection of missing, zero, negative, over-4.00, and over-precision values and of an update that omits it, refusal on non-Degree catalogs, and its audit record in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/CatalogCrudTests.cs` and `ConfigurationAuditTransactionTests.cs`.
- [X] T080 [P] [US1] Add PostgreSQL tests for the `numeric(3,2)` column, its range check, the absence of a column default, and the 2.50 backfill of a Degree that exists before the migration in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/ConfigurationPostgresTests.cs`.
- [X] T081 [P] [US1] Add Angular tests for the Degree-only required CGPA input, its client validation, edit loading, request serialization, and service errors in `iit-academic-portal/src/app/features/configuration/catalogs/catalog-form.spec.ts`.
- [X] T082 [US1] Add `MinimumCgpaForCompletion` to `Degree.cs`, map it in `PortalDbContext.cs` as required `numeric(3,2)` with `CK_Degrees_MinimumCgpaForCompletion_Range`, and validate, audit, and project it in `ConfigurationModels.cs`, `ConfigurationResource.cs`, `ConfigurationResources.cs`, `ConfigurationAuditEventDefinitions.cs`, and `CatalogContracts.cs`.
- [X] T083 [US1] Replace the City Corporation migration with one migration, `CityCorporationsDegreeMinimumCgpaAndGrades`, that also adds the column with a 2.50 backfill for existing Degrees and then drops the column default.
- [X] T084 [US1] Add the CGPA input to the Degree form in `catalog-form.ts`, `catalog-form.html`, and `configuration.models.ts`.
- [X] T085 [US1] Update `spec.md`, `plan.md`, `research.md`, `data-model.md`, `contracts/configuration-api.md`, `quickstart.md`, and both READMEs with the field, its rules, the backfill, and the shared migration.
- [ ] T086 Walk through quickstart scenario 3 by hand and apply the migration to the Development database. **Status (partial):** the migration was applied to the Development database on 2026-10-21 and both existing Degrees got 2.50 (quickstart.md, Development Database Record); scenario 3 has not been walked through by hand; every automated suite, the PostgreSQL-gated tests, and the production build pass (results in quickstart.md).

---

## Phase 14: Grade Scale Addendum - User Story 1 (Priority: P1)

**Goal**: Provide one Admin-maintained Grade scale shared by every Degree, seeded with the ten screenshot bands and kept as a complete active 0–100 partition through atomic full-scale replacement.

**Independent Test**: As an active Admin, read the ten seeded bands, submit a complete valid replacement, and verify the changes persist globally with one audit trail. Verify invalid gaps, overlaps, duplicate live letter grades, out-of-range values, and invalid numeric grades leave both the scale and audit history unchanged; verify concurrent replacements serialize and that Grade has no Degree, student, or course relationship.

### Tests for User Story 1 Addendum

- [X] T087 [P] [US1] Add Grade scale API/service tests for all ten initial bands, inclusive lower and exclusive upper boundaries (including 0 and 100), duplicate normalized letter grades, percentages outside 0–100, numeric grades outside 0.00–4.00 or with more than two decimal places, gaps, overlaps, invalid full-set submissions with no Grade or audit changes, and audit-failure rollback in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GradeScaleTests.cs`.
- [X] T088 [P] [US1] Add PostgreSQL 18 tests for the Grade migration's table, live-row unique letter-grade index, numeric constraints, all ten seeded values, preservation of Admin-edited seed rows, and concurrent full-scale replacements serialized by the transaction-scoped advisory lock in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Configuration/GradeScalePostgresTests.cs`.
- [X] T089 [P] [US1] Add Vitest tests for loading and editing the complete Grade scale, boundary display, valid replacement submission, field-keyed validation feedback, and loading/empty/error states in `iit-academic-portal/src/app/features/configuration/grades/grade-scale-editor.spec.ts`.

### Implementation for User Story 1 Addendum

- [X] T090 [P] [US1] Add `Grade` in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/Grade.cs` with the shared audited/soft-delete fields, letter-grade name and normalized name, `MarksLowerBound`, `MarksUpperBound`, `UpperBoundInclusive`, `NumericGrade`, `Comment`, and `IsActive`; preserve these constraints verbatim: “Percentage bounds are decimal values between 0 and 100; no extra decimal-rounding rule is imposed by the feature”, “NumericGrade is a decimal between 0.00 and 4.00 with at most two decimal places”, and “Comment is required”; add no Degree, student, or course reference.
- [X] T091 [US1] Map Grade in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs` and add the Grade table to the existing unapplied EF migration under `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`, renamed `CityCorporationsDegreeMinimumCgpaAndGrades` (owner decision 2026-10-21: one migration with the City Corporation and Degree minimum CGPA changes, not a separate Grade migration), with a required comment, percentage bounds, `NumericGrade` as `numeric(3,2)`, per-band range checks, and a unique normalized letter-grade index over live rows; seed these exact initial bands with stable IDs and insert-only behavior: A+ 80–100 inclusive, 4.00, Excellent; A 75–80, 3.75, Better; A- 70–75, 3.50, Good; B+ 65–70, 3.25, Above Average; B 60–65, 3.00, Average; B- 55–60, 2.75, Below Average; C+ 50–55, 2.50, Satisfactory; C 45–50, 2.25, Not satisfactory; D 40–45, 2.00, Pass; F 0–40, 0.00, Fail. Every lower bound is inclusive and every upper bound exclusive except A+ includes 100; later migrations or seed runs must not overwrite Admin edits. The seed is an insert-only `INSERT ... ON CONFLICT DO NOTHING` (`InitialGradesSql`), with ids in `GradeSeed.InitialIds`.
- [X] T092 [US1] Implement `GradeScaleService` and application request/response contracts in `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/GradeScaleService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/GradeScaleContracts.cs`; accept the complete desired band set, acquire a PostgreSQL transaction-scoped advisory lock before validating, require the active bands to cover every percentage from 0 through 100 exactly once, update rows by supplied IDs, create rows without IDs, deactivate included inactive rows, soft-delete omitted unreferenced rows, and commit all row changes with allowlisted audit events in one transaction.
- [X] T093 [US1] Add `GradeScaleController` and explicit API DTOs in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/GradeScaleController.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/Configuration/GradeScaleContracts.cs`; expose active-Admin/anti-forgery-protected `GET /api/admin/configuration/grades`, `GET /api/admin/configuration/grades/{id}`, and `PUT /api/admin/configuration/grades/scale`, return field-keyed validation errors, and route every Grade mutation through the full-scale workflow rather than per-record writes.
- [X] T094 [US1] Add Grade scale request/response methods to `iit-academic-portal/src/app/features/configuration/configuration-api.ts` and implement the custom full-set editor in `iit-academic-portal/src/app/features/configuration/grades/grade-scale-editor.ts` and `iit-academic-portal/src/app/features/configuration/grades/grade-scale-editor.html`; submit the complete set and present accessible loading, empty, validation, success, and error states without a Degree selector or student/course grading workflow.
- [X] T095 [US1] Add the `grades` resource under “IIT academics” in `iit-academic-portal/src/app/features/configuration/configuration.models.ts`, mark it `ownEditor` so the generic catalog list and form refuse it, and register its dedicated editor route before `configuration/:resource` in `iit-academic-portal/src/app/app.routes.ts` so `/admin/configuration/grades` opens the custom scale editor instead of generic per-record CRUD. The Configuration home page lists it from the descriptor, so `configuration-home.html` needs no change.
- [X] T096 [US1] Align `specs/003-configuration/contracts/configuration-api.md` with the Grade-specific route exception: retain list/detail reads but state that `PUT /grades/scale` is the only Grade mutation route and that create, update, deactivate, and soft-delete are represented through full-set replacement; update `specs/003-configuration/quickstart.md` with the Grade migration prerequisite and runnable valid/invalid boundary, audit, global-sharing, and replacement scenarios.

**Checkpoint**: The active Grade scale is globally shared, complete, and auditable; all writes are serialized full-set replacements, and the initial screenshot bands are seeded without overwriting later Admin changes.

## Phase 15: Grade Validation & Cross-Cutting Concerns

**Purpose**: Verify Grade-specific provider behavior and the end-to-end Admin editing workflow.

- [ ] T097 Run `dotnet test` and the focused Grade PostgreSQL 18 integration tests from `iit-academic-portal-service/`, plus `npm test -- --watch=false` and `npm run build` from `iit-academic-portal/`; record actual Grade results and any PostgreSQL environment limitation in `specs/003-configuration/quickstart.md` without claiming unrun checks passed. **Status (partial):** all automated suites, including the Grade PostgreSQL tests, and the build pass (quickstart.md, Grade Scale Validation Results); kept open with T078 and T086 until the hand-run scenarios are done.
- [ ] T098 Walk through Grade quickstart scenarios 9–11 using an active Admin session and a disposable PostgreSQL database; verify the seeded values, boundaries, valid full-scale edit, atomic rejection and unchanged audit history for invalid replacements, and the absence of Degree/student/course assignment or calculation behavior, then record outcomes in `specs/003-configuration/quickstart.md`. **Status:** not run.

### Grade Addendum Dependencies & Parallel Opportunities

- Grade work depends on the existing Configuration normalization, active-Admin authorization, audited-entity, soft-delete, and audit-event foundations (T002, T003, T045–T053); it does not depend on Batch or geography implementation.
- T087, T088, and T089 target separate test files and can be authored in parallel before implementation. T090 can proceed independently; T091 depends on T090; T092 depends on the entity and persistence model; T093 depends on the service contract; T094 and T095 depend on the API shape; T096 should be completed with the endpoint implementation before frontend integration is considered complete. T097–T098 depend on the implementation tasks.
- MVP for this addendum is T090–T095: the Grade model, seeded migration, atomic replacement API, and Admin editor. Do not mark the MVP complete until T087–T089 pass and the scale invariants are verified.

### Parallel Example: Grade Scale

```text
Before implementation:
- Worker A: T087 service/API behavior tests in GradeScaleTests.cs
- Worker B: T088 PostgreSQL migration/concurrency tests in GradeScalePostgresTests.cs
- Worker C: T089 Angular editor tests in grade-scale-editor.spec.ts
- Worker D: T090 Grade domain entity in Grade.cs

After test contracts are stable:
- Worker A: T090–T093 backend entity, persistence, service, and controller
- Worker B: T094–T095 Angular API, editor, and route integration
- Worker C: T096 contract and quickstart alignment
```

### Grade Addendum Format Validation

All new Grade tasks use unchecked `- [ ]` checkboxes, sequential IDs T087–T098, `[US1]` on story tasks, `[P]` only for independent test tasks, and explicit repository file paths. Previously completed tasks and their statuses are unchanged.

### City Corporation Addendum Dependencies

- Phase 11 depends on the existing Phase 2 shared normalization/audit foundation and the previously implemented US3 geography slice.
- Author T067-T070 in parallel against separate test files before implementation.
- T071 and T076 may proceed in parallel after tests; T072 depends on T071.
- T073 depends on T071; T074 depends on T073; T075 depends on T071 and T072. T077 follows the final request/import shape.
- T078 depends on all City Corporation implementation and documentation tasks.

### Parallel Example: City Corporation Addendum

```text
After the existing geography slice is stable, author in parallel:
T067 geography CRUD and parent-integrity tests
T068 seed validation/import tests
T069 PostgreSQL migration/constraint tests
T070 Angular parent filter and form/list tests

Then implement in dependency order:
T071 entity and T076 Angular descriptor after tests
T072 persistence/migration after T071
T073 resource registration and District dependent check after T071
T074 geography route after T073
T075 seed import extension after T071/T072
T077 docs after contract and import shapes stabilize
T078 full validation last
```

## Phase 16: Convergence

- [ ] T099 Run the Grade scale editor as an active Admin at 1280, 768, 375, and 320 px; verify keyboard access and visible focus for every field and action, associated validation feedback, contained table overflow with no page-level horizontal overflow, and zero axe WCAG 2.x A/AA violations; record the browser, viewport, and results in `specs/003-configuration/quickstart.md` per Constitution I and FR-021 (partial).
