# Quickstart: Configuration Validation

**Feature**: [Configuration](spec.md)
**Plan**: [Implementation plan](plan.md)  
**Contracts**: [API](contracts/configuration-api.md), [geographic seed schema](contracts/geographic-seed.schema.json)
**Data model**: [data-model.md](data-model.md)

## Prerequisites

- Node.js 24 LTS and .NET 10 SDK.
- PostgreSQL 18 with the existing portal database configured; the service README documents Development secrets and connection strings.
- The existing feature migrations (`ConfigurationReferenceData`, `AddHalls`, `SoftDeleteConfiguration`, and `AuditedEntityColumns`) applied through the schema-owner workflow. `DegreeCredits` is generated; the owner confirmed the target Degree table is empty on 2026-10-18. Reverify that it is still empty immediately before applying the migration, which adds the required column without a default. If rows exist, stop and obtain owner-approved totals through a deployment-only backfill; do not put those values in source or spec files. The runtime role's default privileges cover schema changes.
- Development Admin account (`admin@iit.test`) configured with `DevelopmentSeed:Password`.
- For seed acceptance, the owner-approved JSON file with source citation/version. Do not substitute manually compiled or invented geography data.

## Build and Automated Tests

Run from the repository root after implementation:

Do not run `dotnet ef database update` against a target database until the Degree-credit prerequisite above is satisfied and recorded.

## Degree Credits Addendum Status (2026-10-18)

| Check | Result |
|---|---|
| Focused backend catalog tests | 84 passed, 0 failed |
| Full backend API suite | 501 passed, 49 PostgreSQL tests skipped, 0 failed |
| Focused PostgreSQL configuration tests | 9 skipped because `AUDIT_TEST_POSTGRES` is unset; test assembly builds successfully. **Run since (2026-10-20, T066):** with `AUDIT_TEST_POSTGRES` set, every PostgreSQL test passed on fresh databases, including `Degree_credit_total_is_stored_as_required_positive_numeric_with_two_decimal_places` (`numeric(8,2)`, `NOT NULL`, positive check) |
| Full frontend suite | 116 passed (15 files), 0 failed |
| Frontend production build | Succeeds |

Degree create/read/update validation and the Degree-only form are implemented. The PostgreSQL precision/required/positive-value assertion is authored but could not execute without the database connection. `DegreeCredits` adds `numeric(8,2) NOT NULL` and a positive-value constraint with no default or embedded backfill; its generated SQL was inspected and the EF model snapshot is current. The owner confirmed the target Degree table is empty on 2026-10-18, so no mapping file or credit total is recorded here. The migration has not been applied; reverify the table is empty before applying it. **Update:** it was applied to the Development database together with the feature 007 migration (see the 007 implementation notes).

```powershell
Push-Location iit-academic-portal-service
dotnet tool restore
dotnet ef database update -p src/IitAcademicPortal.Infrastructure -s src/IitAcademicPortal.Api
dotnet test
Pop-Location

Push-Location iit-academic-portal
npm ci
npm test -- --watch=false
npm run build
Pop-Location
```

The service suite must pass its SQLite pipeline tests. Run the PostgreSQL integration suite with the documented `AUDIT_TEST_POSTGRES` connection configured to verify actual migrations, Degree credit backfill/positive-value constraints, nullable/composite unique indexes, restrictive foreign keys, and transactional audit behavior.

The `CityCorporationsDegreeMinimumCgpaAndGrades` migration also creates the Grade table and seeds the ten screenshot bands (one migration at the owner's request, instead of a separate Grade migration). Its seed is insert-only, so running it again never overwrites Grade rows changed by Admins or restores a deleted band.

## End-to-End Scenarios

1. Start PostgreSQL, then run the service using its HTTPS profile from `iit-academic-portal-service`; start the Angular dev server from `iit-academic-portal`. Open `https://localhost:7286/swagger` and `http://localhost:4200`.
2. Sign in as the Development Admin. Verify the Configuration menu appears in the Admin module and opens `/admin/configuration`. Create, search, update, and deactivate one record in each independent catalog, including an Academic Session and a Hall (name, optional code, optional address). Verify the fixed institution-type choices and that similar names in separate catalogs are allowed.
3. Create a Degree with a positive `creditsToBeCompleted` value such as `120` and a `minimumCgpaForCompletion` such as `2.50`; verify create, read, and update preserve both values. Try omitting either field and submitting zero, a negative value, or more than two fractional digits, and a CGPA above `4.00`; each must return a field-keyed 400 and leave the Degree unchanged. After the `CityCorporationsDegreeMinimumCgpaAndGrades` migration, a Degree that existed before shows 2.50.
4. Create a duplicate name with case, surrounding whitespace, and repeated internal spaces changed; verify a field-level validation error and no stored change. Repeat with an optional code differing only by case.
5. Create two active Degrees with valid required-credit totals, Semesters, and Academic Sessions. Verify duplicate normalized Session labels are rejected globally, while multiple Batches can reference the same Session. Create Batches with a name and one active Session and verify success; omit the name or Session, or supply an inactive/nonexistent Session, and verify rejection. Create equal Batch names under different Degrees and verify both succeed; create the duplicate under one Degree and verify rejection regardless of Session. Deactivate a referenced Session and verify existing Batches still display it while new assignments are rejected.
5. (Needs the owner file.) As the active Admin, submit the owner-approved JSON body to `POST /api/admin/configuration/geography/import/validate`. Confirm the report cites the exact source/version and reports per-level counts without writing data. After a clean report, submit it to `POST /api/admin/configuration/geography/import`; verify every District and Upazila has the correct parent. Re-run and verify it reports conflicts without partial writes or overwrites.
6. Deactivate a referenced Division or District. Verify existing children remain visible, new children/assignments under it are rejected, and descendants are not deactivated. Delete an unreferenced record and verify 204 plus an audit event, that it no longer appears in lists or opens (404), that its name can be created again, and that the row is still in the database marked deleted (who and when); attempt to delete a referenced Degree or Division and verify 409, no cascade, and a deactivation alternative. Attempt a cross-branch District/Upazila parent and verify rejection.
7. Switch the session's active role away from Admin; verify the menu entry is absent and direct navigation is denied. Call POST, PUT, and DELETE configuration endpoints directly through Swagger and verify 403. Repeat anonymously for 401. Return to Admin and verify successful create/update/deactivate/delete operations appear in `/api/audit-events` with safe allowlisted changes.
8. Exercise loading, validation, empty, inactive, and error states in each relevant UI. Verify keyboard access, focus visibility, responsive list/form behavior, and infinite loading on a list with multiple batches: new rows append near the end, search/status changes reset to the first batch, manual Load more works, and a later-batch error retains prior rows and can be retried. Verify there are no numbered page controls or page-size selector, and no axe violations or extra scrollbar/overflow at 1280 and 375 px.
9. Open the Grade scale editor as Admin and verify the ten initial bands appear in one shared catalog regardless of Degree. Check boundary mapping at 0, 39.99, 40, 44.99, 45, 79.99, 80, and 100 against the screenshot table. Update a comment or numeric grade and submit the complete scale; verify the change is visible after reload and audit history records it.
10. Replace the Grade scale with a valid modified band set while preserving continuous coverage. Try a gap, overlap, percentage below 0 or above 100, invalid letter-grade duplicate, numeric grade outside 0.00–4.00, or more than two numeric-grade decimal places; verify the server rejects the whole replacement without changing bands or audit history. Deactivate or omit a band only when the submitted replacement still provides complete active coverage, and verify omitted unreferenced bands are soft-deleted.
11. Verify Grade entries are not linked to a Degree, student, or course and that this feature neither assigns a Grade nor calculates marks, results, or CGPA.

## Geographic Import

Use Swagger UI at `https://localhost:7286/swagger` with a signed-in active Admin session. Paste the owner-approved JSON document into the request body for the validate endpoint, review its report, then submit it to the import endpoint. Swagger must send the normal session cookie and `X-CSRF-Token` header.

A document that breaks the schema (for example one with a `code` field) returns 400 with the report and the path of each problem. Expected results: validation writes no data and emits a complete report; import accepts only a fully valid document and commits the hierarchy and its audit event atomically. Non-Admin sessions receive 403. Until the owner supplies the approved file, production seed verification is not complete.

## Verification Record (2026-10-14)

| Check | Result |
|---|---|
| `dotnet build -c Release` | 0 warnings, 0 errors |
| `dotnet test`, SQLite pipeline only | 496 passed, 48 PostgreSQL tests skipped, 0 failed |
| `dotnet test` with `AUDIT_TEST_POSTGRES` (PostgreSQL 18.3) | **544 passed, 0 skipped, 0 failed** (includes the Configuration migration, unique-index, restrictive-key, and delete-integrity tests as the restricted runtime role) |
| Frontend `ng test` (Vitest) | 114 passed (15 files), 0 failed |
| Frontend `ng build` | Succeeds |
| Live API run against PostgreSQL (scenarios 2, 3, 4, 6, 7; scenario 5 for validate-only and refusal) | 35 of 35 checks passed; all test records removed afterwards |
| Live UI run in headless Chrome (home, lists, forms, batch selectors, cascading upazila form, delete conflict and "Deactivate instead") | axe-core WCAG 2.x A/AA: no violations on any screen; no second scrollbar; no horizontal overflow at 1280, 768, 375, and 320 px |

Not done: scenario 5 with the owner-approved file (task T034, waiting for the owner), a screen-reader pass, and a usability
session for SC-006. See `implementation-notes.md`.

## Geographic Import Record (2026-10-14, T034)

Run through the Admin-authorized endpoints on the Development database (PostgreSQL), signed in as `admin@iit.test`.

| Item | Value |
|---|---|
| Source files | `bd-divisions.json`, `bd-districts.json`, `bd-upazilas.json`, supplied by the project owner |
| Source citation recorded | "Project-owner-supplied files ... No official source was cited; the owner chose to import them as supplied." |
| Version recorded | "Files supplied 2026-10-14 (8 divisions, 64 districts, 494 upazilas as supplied)" |
| Validation (`/import/validate`) | 200, `isValid` true, `canImport` true, `targetEmpty` true, no issues; nothing was written |
| Import (`/import`) | 200; **8 divisions, 64 districts, 492 upazilas** stored in one transaction with one `configuration.geography.imported` audit event |
| Parent links | Every stored district and upazila matches the seed's parent, checked through the API against the seed |
| Second import | 409; no rows changed |

The files are community-style data (they carry `bn_name`, `lat`, `long`, and numeric ids), so they were converted to the seed
schema (names and nesting only; the other fields are dropped). Three changes were made by the importer's rules and the owner's
decisions, and the source files were not edited:

1. Two exact repeats in Natore were skipped: upazila ids 361 (Natore Sadar) and 362 (Baraigram) repeat 357 and 358. The importer
   rejects duplicate names within a district, so 492 of the 494 supplied upazilas were imported.
2. Upazila id 230 "Raipura, Narsingdi" was stored as "Raipura"; the name is already unique within Narsingdi.
3. The source and version above state that no official source was cited.

**Caveat on SC-005 and FR-014**: those require data from the *approved official source*. The owner did not supply one, so the
data's fidelity to an official source is **not claimed**; only that it is structurally valid, complete, and imported as
supplied. If an official source is chosen later, the geography tables must be emptied (deleting bottom-up) before importing it,
because the importer never overwrites existing rows.

The seed built from the files is kept beside them as `bangladesh-geography-seed.json`, outside the repository, for repeating
the import in other environments.

## Halls Verification (2026-10-14)

Live run on the Development database (the `AddHalls` migration applied) in headless Chrome: the empty Halls list, the form (name,
code, address), creating a hall, the duplicate-name error after normalizing, editing (the address loads), delete with confirmation,
and the list columns (Name, Code, Status). axe-core WCAG 2.x A/AA reported no violations on the list or the form, and the form has no
horizontal overflow at 375 px. The test hall was deleted afterwards.

### Hall seed (2026-10-14)

`seed-halls.sql` was run on the Development database twice (23 halls both times) and on a brand-new database (the same 23 ids and names,
checked by diffing the two lists). Through the API, the seeded halls list with their address and no code, a duplicate name that differs
only in case and spacing is rejected, search finds the three hostels, and a seeded hall opens by its fixed id. One unrelated note: a PUT
of an unchanged seeded hall returned 400 once, on the first requests after a cold server start; it returned 200 on every later attempt,
including repeats of the identical request sequence, and the cause was not found.

### Geography seed scripts (2026-10-14)

`seed-divisions.sql`, `seed-districts.sql`, and `seed-upazilas.sql` are the repeatable way to load the geography from the owner's files. On the Development
database they produced 8 divisions, 64 districts, and 492 upazilas with fixed ids (a second run left every table unchanged), and a brand-new database
seeded the same way has identical ids and names; before the Development rows (first loaded through the import endpoint with random ids) were replaced,
their names and parents were confirmed identical to the script output and nothing referenced them. Through the API, Dhaka has its fixed id and 13 districts, a
duplicate division name after normalizing is rejected, and search finds "Cox's Bazar Sadar". Because the tables are now seeded, the import endpoint returns 409 on this
database (scenario 5 needs empty tables).

## Soft Delete Verification (2026-10-14)

After the `SoftDeleteConfiguration` migration was applied to the Development database (all 23 halls and the 8, 64, and 492 geography rows
unchanged, none deleted), a probe hall was created, deleted (204), and then returned 404 on read and on a second delete, dropped out of the
list and the count, and could be created again with the same name and code. The database rows were kept with `IsDeleted` true, the deletion time
(UTC), and the Admin's account id. Deleting Dhaka, which has districts, returned 409 with the deactivate hint. The probe rows were removed
afterwards. PostgreSQL tests confirm the foreign keys still refuse a physical delete of a referenced row and that every unique index covers
live rows only (10 name, 5 code, and 3 parent-scoped indexes).

## Created and Last Modified Verification (2026-10-14)

After the `AuditedEntityColumns` migration was applied to the Development database (the 23 halls and the 8, 64, and 492 geography rows
unchanged; the existing rows show the migration time and no account), the edit screen was checked in headless Chrome against the real API:
a seeded hall shows "Created ... by System" and "Last modified ... by System"; a hall created in the screen shows the Admin's email on both
rows with the same time; after an edit, "Last modified" moves and "Created" stays. axe-core reported no violations, and there was no second
scrollbar and no horizontal overflow at 375 px. The probe hall was removed afterwards. Automated tests cover create, update, deactivate,
delete, a no-op update, the server clock, a client trying to supply the values (400), a removed account, a change outside a request, the
PostgreSQL columns and defaults, and the convention that fails when a new entity is not derived from `AuditedEntity` or exempt.

## City Corporation Extension Validation

Run these checks after the City Corporation migration and resource/UI changes are implemented. The currently documented geography seed contains no City Corporation source data; use synthetic City Corporation fixtures for automated import tests and do not claim production seed-source acceptance until the owner supplies an approved dataset and citation.

1. As Admin, create a City Corporation under an active District. Verify it appears in the Geography group, its District is displayed, no code field is required or returned, and the list can be filtered by `districtId`.
2. Submit a City Corporation with a missing, nonexistent, or inactive District and verify a field-keyed rejection with no stored change. Rename it to a normalized duplicate in the same District and verify rejection; use the same name in a different District and verify it is allowed.
3. Deactivate a District that has City Corporations. Verify its City Corporations remain visible through existing references, new City Corporations under it are rejected, and deleting the District returns a conflict until all live child records are removed or deleted.
4. Validate and import a synthetic seed containing a City Corporation in a District. Verify the validation report includes the City Corporation count without writing; verify import creates the correct District link and count atomically; submit a duplicate City Corporation name under one District and verify the whole import is rejected without partial rows or an audit event.
5. Run PostgreSQL integration tests to verify the CityCorporations migration, restrictive District foreign key, and unique `(DistrictId, NormalizedName)` index for non-deleted records.

### City Corporation Validation Results (2026-10-20)

Run on Windows 11 with .NET 10 SDK, Node 24, and a local PostgreSQL 18 server. All import data was synthetic; no City Corporation seed was
loaded, and none exists until the owner supplies an approved source and version.

| Check | Result |
|---|---|
| `dotnet test` (Release) with `AUDIT_TEST_POSTGRES` | 1153 passed, 1 skipped (the feature 006 Coordinator release-gate test), with the Degree minimum CGPA and the Grade scale included |
| `npm test -- --watch=false` | 323 passed (29 files), with the Degree minimum CGPA and the Grade scale editor included |
| `npm run build` | Succeeded; initial bundle 455.83 kB (budget 500 kB); the Grade editor is its own lazy chunk (16.04 kB) |
| Migration `CityCorporationsDegreeMinimumCgpaAndGrades` | Creates the `CityCorporations` table, its live-row unique index on (`DistrictId`, `NormalizedName`), a `NormalizedName` index, and a restrictive key to `Districts`, and adds the Degree minimum CGPA column (see below); applied to the disposable PostgreSQL test databases |

Scenarios 1-5 above are covered by automated tests: CRUD, filters, District rules, lifecycle, and District delete protection in
`GeographyTests`; import counts, District links, duplicates, path reporting, and the second-import conflict in `GeographyImportTests`; the
index, key, and soft-delete reuse on PostgreSQL in `ConfigurationPostgresTests`; authorization and audit in `ConfigurationAuthorizationTests`
and `ConfigurationAuditTransactionTests`; and the screens in `catalog-form.spec.ts`, `catalog-list.spec.ts`, and `configuration-home.spec.ts`.

**Not done:** the scenarios have not been walked through by hand in a browser. The migration was applied to the Development database on
2026-10-21 (below).

### Degree Minimum CGPA Validation Results (2026-10-20)

The same runs as above include the new Degree field. Covered by automated tests: the required value, the 0.01 and 4.00 bounds, rejection
of 0, -1, 4.01, 5, and 2.555, an update that omits it, refusal on another catalog, and the audit record (`CatalogCrudTests`,
`ConfigurationAuditTransactionTests`); the `numeric(3,2)` column, its check constraint, no column default, and a Degree created before
the migration receiving 2.50 (`ConfigurationPostgresTests`, which migrates a fresh database to `Courses`, inserts a Degree, and migrates
on); and the form field, its client validation, and service errors (`catalog-form.spec.ts`). `dotnet ef migrations
has-pending-model-changes` reports none.

**Not done:** scenario 3 has not been walked through by hand. The migration was applied to the Development database on 2026-10-21 (below).

### Grade Scale Validation Results (2026-10-21)

Same runs as the table above. `dotnet ef migrations has-pending-model-changes` reports none.

| Area | Automated coverage |
|---|---|
| API (`GradeScaleTests`, SQLite host) | The ten bands stored and listed highest first with their values; marks 0, 39.99, 40, 44.99, 45, 50, 74.999, 75, 79.99, 80, and 100 each in exactly one band; search and status filters; a replacement that updates, creates, deactivates, and soft deletes with one event each; an unchanged resubmission recording nothing; 19 invalid replacements (gap, overlap, no 0%, top not inclusive, inclusive below 100, below 0, above 100, lower not below upper, numeric grade above 4, negative, three decimals, duplicate letter, blank letter, blank comment, missing grade, unknown id, repeated id, no active band, empty) each refused whole with no band or audit change; inactive overlapping bands; a deleted letter reused; unknown fields refused; single-band writes answering 405; Admin-only access with the anti-forgery token; and audit-failure rollback |
| PostgreSQL (`GradeScalePostgresTests`, each on a fresh database) | The migration seeds the ten bands with their stable ids; the bound, inclusive-at-100, numeric-grade, and comment checks, the `numeric`/`numeric(3,2)` column types, the live-only unique letter grade, and no foreign keys; re-running the seed keeps an Admin's edit and does not restore a deleted band; two concurrent valid replacements, held together at the lock, both commit and leave a complete 0-100 partition (three rounds) |
| Angular (`grade-scale-editor.spec.ts`, plus `configuration-home.spec.ts` and `catalog-list.spec.ts`) | Loading, empty, and load-error states with retry; the bands and the "includes 100%" note; the complete request (ids kept, new bands without one, removed bands left out, `upperBoundInclusive` set for the band ending at 100); client checks for gaps and per-band values before anything is sent; service errors shown against the scale and the band with edits kept; a failed save; discarding changes; the Grades link under IIT academics; and the generic list refusing `grades` |

**Not done:** scenarios 9-11 have not been walked through by hand (T098). The migration was applied to the Development database on
2026-10-21 (below).

### Development Database Record (2026-10-21)

At the project owner's request, `dotnet ef database update` applied `20261020183156_CityCorporationsDegreeMinimumCgpaAndGrades` to the
local Development database (`iit_academic_portal`); `dotnet ef migrations list` then showed no pending migration. Checked afterwards with
`psql`:

| Check | Result |
|---|---|
| `Grades` | 10 bands, exactly the seed: A+ 80-100 4.00, A 75-80 3.75, A- 70-75 3.50, B+ 65-70 3.25, B 60-65 3.00, B- 55-60 2.75, C+ 50-55 2.50, C 45-50 2.25, D 40-45 2.00, F 0-40 0.00 |
| `Degrees.MinimumCgpaForCompletion` | Both existing Degrees backfilled to 2.50; the column has no default |
| `CityCorporations` | Created, empty (no City Corporation data has been supplied) |

No shared, staging, or Production database was changed.

