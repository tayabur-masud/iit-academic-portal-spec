# Implementation Notes: Configuration

**Branch**: `feature/003-configuration` (spec repo and both submodules) · **Recorded**: 2026-10-14 · **Not committed or pushed**

## Status

| Story | State |
|---|---|
| US1 Academic catalogs (9, including Halls) | Implemented and tested (service, PostgreSQL, Angular, live API and UI) |
| US2 Batches | Implemented and tested |
| US3 Geography and seed import | Implemented and tested; the owner's files were imported into the Development database on 2026-10-14 (8 divisions, 64 districts, 492 upazilas), **but they cite no official source** |
| US4 Protection and references | Implemented and tested (authorization, atomic audit, reference-safe delete on PostgreSQL) |

All 37 tasks are done. T034 was run with the owner's files; its record, including the three data changes and the caveat that no
official source was cited, is the Geographic Import Record in `quickstart.md`. Results are in the Verification Record there.

## How it was built, and where it differs from the task text

The thirteen resources share one flow instead of thirteen copies, so the files differ from the paths the tasks named. The
behavior, routes, and rules are unchanged.

| Task text | What exists |
|---|---|
| Per-resource entities (T004 to T007, T014, T021) | Thirteen entities in `Domain/Configuration/` over a shared `ConfigurationRecord` (identifier, name, normalized name, active flag) |
| `CatalogService.cs`, `BatchService.cs`, `GeographyService` (T009, T016, T024) | `ConfigurationResource<T>` holds the one create, read, update, deactivate, and delete flow; each resource supplies only what differs (extra fields, the scope of name uniqueness, the references it validates). `CatalogService` is the entry point and `ConfigurationResources` the registry |
| `CatalogContracts.cs`, `BatchContracts.cs` (Application) | `ConfigurationModels.cs`; the API has its own explicit DTOs in `Api/Contracts/Configuration/` |
| Three migrations (T008, T015, T023) | **One** migration, `ConfigurationReferenceData` (the plan asked for one reviewed migration) |
| Three controllers | `CatalogsController`, `BatchesController`, `GeographyController`, all on a shared `ConfigurationControllerBase` |
| Separate Angular list and form files for catalogs, batches, and geography (T011, T018, T026) | One generic list and form in `catalogs/`, driven by the descriptors in `configuration.models.ts`; batches and the cascading Division, District, Upazila selection are descriptor features of the same components. A landing page groups the resources as IIT academics, Other catalogs, and Geography (regrouped 2026-10-14 at the owner's request; batches had been alone in a section) |
| `ConfigurationAuditEventDefinitions` (T003) | Done, generating the 52 `configuration.{resource}.{action}` definitions plus `configuration.geography.imported` |
| Maximum lengths (data-model left open) | Name 200, code 50, address 500, enforced by the API and the database |

Decisions made while building, not stated in the spec:

- Configuration lists now use cursor-based infinite loading: each response appends a bounded batch as the reader nears the end,
  with a manual Load more fallback and no numbered controls. The API retains its `page` response mode for non-UI compatibility.
- The seed scripts were re-checked after soft delete and the audit columns (T054). The columns need no change (defaults cover them), but the
  district and upazila scripts could attach new rows to a deleted parent, because raw SQL does not go through the application's deleted-row
  filter; they now look up live parents only and stop with an error if a parent is missing or deleted.
- Every entity records created by, created time, last modified by, and last modified time (decided 2026-10-14, tasks T049 to T053) through
  the shared `AuditedEntity` base class and a save interceptor, so a future entity gets them by deriving from it; a convention test fails the
  build otherwise. The exemptions (Identity, `AuthSession`, `AuditEvent`, the security outbox) are listed with reasons in that test. The
  convention is written in the service README ("Entity conventions"). Existing rows were backfilled with the migration time and no account.
  The constitution was not changed; adding the convention there would make it a project-wide governance rule and is left to the owner.
- Every delete is a soft delete (decided 2026-10-14, tasks T045 to T048): the row is kept and marked (`IsDeleted`, `DeletedAtUtc`,
  `DeletedByUserId`), a global query filter hides it, no portal feature restores it, a record with live dependents still cannot be
  deleted (409, checked in the application because a soft delete no longer trips a foreign key), and the unique name and code indexes
  became filtered indexes over live rows so a deleted record's name and code can be used again. This reverses the original
  requirement for physical deletion (FR-016); the foreign keys stay `RESTRICT` as a guard against any physical delete.
- Halls (university residential halls: name, optional code, optional address, independent) were added on 2026-10-14 as an addendum
  (T038 to T043) with their own migration, `AddHalls`; they reuse the coded-catalog flow and the shared address rules. The existing 23
  halls come from a saved SQL seed with fixed ids (`database/postgresql/seed-halls.sql`, tested in `HallSeedScriptTests`), which was
  run twice on the Development database and once on a brand-new database with identical ids and names. The geography has the same kind
  of seed (`seed-divisions.sql`, `seed-districts.sql`, `seed-upazilas.sql`, T044, tested in `GeographySeedScriptTests`); the Development
  database's geography, first loaded through the import endpoint with random ids, was replaced by the fixed-id seed after confirming the two
  were identical by name and parent, and a second run left it unchanged.
- Infinite-loading lists use bounded API batches (25 by default, maximum 100); no page-size selector is presented in the browser.
- An update that changes nothing is a no-op: no write and no audit event.
- A reference that is unchanged on update is accepted even if its record was later deactivated; a new or changed
  reference must be active.
- Unknown JSON fields are rejected, so a client cannot name the audit actor or any other field.
- The geography seed validator rejects any field other than names and nesting, including `code`, with the path.

## Defects the tests found (and fixed)

1. **PostgreSQL reports a RESTRICT foreign-key violation as `23001`, not `23503`.** The SQLite pipeline tests passed,
   but on PostgreSQL deleting a referenced record would have returned 500 instead of 409. Found by
   `ConfigurationDeleteIntegrityTests` and verified live (409 with "Deactivate it instead").
2. **Business events staged with no actor.** `IAuditEventRecorder.Stage` did not default the actor from the request
   context. Fixed in the recorder (business events only) and noted in the 002 implementation notes.

## Verification

See the Verification Record in `quickstart.md`: 544 service tests with PostgreSQL, 114 frontend tests, 35 live API
checks, and a live headless-Chrome run with axe-core on every Configuration screen.

## Not done

- **An approved official source for the geography**: the imported data carries a citation saying none was supplied, so SC-005 and FR-014's
  "approved official source" are not claimed.
- The import was run on the Development database only; staging and production need the same flow (the seed is kept outside the repo).
- A screen-reader pass and a keyboard-only walkthrough by a person.
- The SC-006 usability sessions (95% of Admins find and update or deactivate a record within two minutes).
- The geography import has no screen; it is used through Swagger as the quickstart describes. Say so if a screen is wanted.
- No numeric performance target exists in the spec; lists are bounded (page size 25, maximum 100) and cursor-paged.
