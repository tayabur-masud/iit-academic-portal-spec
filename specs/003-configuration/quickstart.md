# Quickstart: Configuration Validation

**Feature**: [Configuration](spec.md)
**Plan**: [Implementation plan](plan.md)  
**Contracts**: [API](contracts/configuration-api.md), [geographic seed schema](contracts/geographic-seed.schema.json)
**Data model**: [data-model.md](data-model.md)

## Prerequisites

- Node.js 24 LTS and .NET 10 SDK.
- PostgreSQL 18 with the existing portal database configured; the service README documents Development secrets and connection strings.
- The feature migration (`ConfigurationReferenceData`, `AddHalls`, then `SoftDeleteConfiguration`, then `AuditedEntityColumns`) applied through the existing schema-owner workflow. It only adds tables, and the runtime role's default privileges cover them.
- Development Admin account (`admin@iit.test`) configured with `DevelopmentSeed:Password`.
- For seed acceptance, the owner-approved JSON file with source citation/version. Do not substitute manually compiled or invented geography data.

## Build and Automated Tests

Run from the repository root after implementation:

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

The service suite must pass its SQLite pipeline tests. Run the PostgreSQL integration suite with the documented `AUDIT_TEST_POSTGRES` connection configured to verify actual migrations, nullable/composite unique indexes, restrictive foreign keys, and transactional audit behavior.

## End-to-End Scenarios

1. Start PostgreSQL, then run the service using its HTTPS profile from `iit-academic-portal-service`; start the Angular dev server from `iit-academic-portal`. Open `https://localhost:7286/swagger` and `http://localhost:4200`.
2. Sign in as the Development Admin. Verify the Configuration menu appears in the Admin module and opens `/admin/configuration`. Create, search, update, and deactivate one record in each independent catalog, including an Academic Session and a Hall (name, optional code, optional address). Verify the fixed institution-type choices and that similar names in separate catalogs are allowed.
3. Create a duplicate name with case, surrounding whitespace, and repeated internal spaces changed; verify a field-level validation error and no stored change. Repeat with an optional code differing only by case.
4. Create two active Degrees, Semesters, and Academic Sessions. Verify duplicate normalized Session labels are rejected globally, while multiple Batches can reference the same Session. Create Batches with a name and one active Session and verify success; omit the name or Session, or supply an inactive/nonexistent Session, and verify rejection. Create equal Batch names under different Degrees and verify both succeed; create the duplicate under one Degree and verify rejection regardless of Session. Deactivate a referenced Session and verify existing Batches still display it while new assignments are rejected.
5. (Needs the owner file.) As the active Admin, submit the owner-approved JSON body to `POST /api/admin/configuration/geography/import/validate`. Confirm the report cites the exact source/version and reports per-level counts without writing data. After a clean report, submit it to `POST /api/admin/configuration/geography/import`; verify every District and Upazila has the correct parent. Re-run and verify it reports conflicts without partial writes or overwrites.
6. Deactivate a referenced Division or District. Verify existing children remain visible, new children/assignments under it are rejected, and descendants are not deactivated. Delete an unreferenced record and verify 204 plus an audit event, that it no longer appears in lists or opens (404), that its name can be created again, and that the row is still in the database marked deleted (who and when); attempt to delete a referenced Degree or Division and verify 409, no cascade, and a deactivation alternative. Attempt a cross-branch District/Upazila parent and verify rejection.
7. Switch the session's active role away from Admin; verify the menu entry is absent and direct navigation is denied. Call POST, PUT, and DELETE configuration endpoints directly through Swagger and verify 403. Repeat anonymously for 401. Return to Admin and verify successful create/update/deactivate/delete operations appear in `/api/audit-events` with safe allowlisted changes.
8. Exercise loading, validation, empty, inactive, and error states in each relevant UI. Verify keyboard access, focus visibility, responsive list/form behavior, and infinite loading on a list with multiple batches: new rows append near the end, search/status changes reset to the first batch, manual Load more works, and a later-batch error retains prior rows and can be retried. Verify there are no numbered page controls or page-size selector, and no axe violations or extra scrollbar/overflow at 1280 and 375 px.

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
