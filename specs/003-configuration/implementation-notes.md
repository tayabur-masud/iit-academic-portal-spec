# Implementation Notes: Configuration

**Branch**: `feature/003-configuration` (spec repo and both submodules) · **Recorded**: 2026-10-14, addenda through 2026-10-21 · **Addenda (City Corporation, Degree minimum CGPA, Grade scale) not committed or pushed**

## Status

| Story | State |
|---|---|
| US1 Academic catalogs (9, including Halls) | Implemented and tested (service, PostgreSQL, Angular, live API and UI) |
| US2 Batches | Implemented and tested |
| US3 Geography and seed import | Implemented and tested; the owner's files were imported into the Development database on 2026-10-14 (8 divisions, 64 districts, 492 upazilas), **but they cite no official source** |
| US4 Protection and references | Implemented and tested (authorization, atomic audit, reference-safe delete on PostgreSQL) |
| Addendum: Halls, soft delete, created/modified (2026-10-14) | Implemented and tested |
| Addendum: Degree credits to be completed (2026-10-18) | Implemented and tested, including on PostgreSQL |
| Addendum: City Corporation (2026-10-20) | Implemented and tested with synthetic import data; not yet walked through by hand |
| Addendum: Degree minimum CGPA for completion (2026-10-20) | Implemented and tested; existing Degrees get 2.50; not yet walked through by hand |
| Addendum: Grade scale (2026-10-21) | Implemented and tested, including the seed and concurrency on PostgreSQL; not yet walked through by hand |

The original 37 tasks are done; T034 was run with the owner's files (the Geographic Import Record in `quickstart.md`, including the caveat
that no official source was cited). Of the addendum tasks, these remain open:

- **T065**: an owner-approved official geographic source.
- **T078, T086, T097**: kept open until their hand-run scenarios are done; every automated suite passes.
- **T098**: the hand-run Grade scenarios.

The City Corporation, Degree minimum CGPA, and Grade changes share one migration, `CityCorporationsDegreeMinimumCgpaAndGrades`, applied to the Development database on
2026-10-21 (see the Development database record in `quickstart.md`); no shared or Production database was changed.

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
checks, and a live headless-Chrome run with axe-core on every Configuration screen. The latest automated run (2026-10-21, after the Grade
scale) is in the City Corporation, Degree Minimum CGPA, and Grade Scale Validation Results there: 1153 service tests with PostgreSQL (1
skipped), 323 frontend tests, and a production build. The addenda have had no live browser or axe-core run.

## Not done

- **An approved official source for the geography**: the imported data carries a citation saying none was supplied, so SC-005 and FR-014's
  "approved official source" are not claimed.
- The import was run on the Development database only; staging and production need the same flow (the seed is kept outside the repo).
- A screen-reader pass and a keyboard-only walkthrough by a person.
- The SC-006 usability sessions (95% of Admins find and update or deactivate a record within two minutes).
- The geography import has no screen; it is used through Swagger as the quickstart describes. Say so if a screen is wanted.
- No numeric performance target exists in the spec; lists are bounded (page size 25, maximum 100) and cursor-paged.

## City Corporation addendum (2026-10-20)

- `CityCorporation` mirrors `Upazila`: a District child with a name, status, soft delete, and audit columns, and no code. It is a sibling of
  Upazila, not a child of it, and the two are separate tables, so the same name may be an Upazila and a City Corporation of one District.
- `CityCorporationResource` reuses the shared reference check, so a new or changed District must be active while an unchanged inactive one
  is accepted. `DistrictResource.HasDependentsAsync` now also counts live City Corporations.
- The audit event key is `city-corporation`, following the kebab-case keys already used (`academic-session`, `examination-type`).
- The import accepts the optional `cityCorporations` list, counts it in the report and the audit metadata, and counts its table in the
  target-empty check. The owner's list of 13 city corporations (2026-10-21) is in `seed-city-corporations.sql`, built like the other geography seeds; like them it
  cites no official source.
- Angular needed only a descriptor; the generic list and form already provide the District column, the Division-to-District cascade, and
  the `districtId` filter. The tests are in `catalogs/catalog-form.spec.ts` and `catalogs/catalog-list.spec.ts`, not a `geography/` spec.
- Student addresses still reference only Upazilas. Choosing a City Corporation in a student address is not part of this addendum.
- Not done: the hand-run quickstart scenarios and an approved City Corporation dataset. The migration is applied to the Development database.
- The table ships in `CityCorporationsDegreeMinimumCgpaAndGrades`, shared with the Degree minimum CGPA and the Grade scale below. The earlier
  `CityCorporations` and `CityCorporationsDegreeMinimumCgpaAndGrades` migrations were folded into it before either reached any shared database.

## Degree minimum CGPA addendum (2026-10-20)

- `Degree.MinimumCgpaForCompletion` is required, greater than 0 and at most 4.00, with two decimals (`numeric(3,2)`), validated in
  `DegreeResource` and by the check `CK_Degrees_MinimumCgpaForCompletion_Range`. It is part of the Degree's "completion requirements"
  flag, so other catalogs refuse it like `creditsToBeCompleted`, and it is in the audit allowlist.
- The migration backfills 2.50 for existing Degrees (owner decision) and then drops the default, so a new Degree must always send a value
  and an update that leaves it out is refused instead of being reset.
- SQLite, used by the test host, stores decimals as text and compared `'4.0'` to `4` as text, which refused 4.00. The check therefore
  casts to a number on SQLite only; PostgreSQL keeps the plain numeric check, and the migration is unaffected.
- Graduation eligibility (feature 007) still checks credits only; using the CGPA waits until the portal calculates a student's CGPA.
- Every existing test that creates a Degree now sends 2.50, through the shared helpers where it could.

## Grade scale addendum (2026-10-21)

- **One migration, no seed in it.** At the owner's request the Grade table went into the existing, unapplied migration instead of a new
  one. It was regenerated and renamed `CityCorporationsDegreeMinimumCgpaAndGrades` so the name says what it holds; the Degree CGPA backfill
  (2.50, then no default) was carried over unchanged. The ten initial bands were first seeded inside the migration with hand-picked
  sequential ids; on 2026-10-21 the owner moved them to `database/postgresql/seed-grades.sql`, like the geography seeds, with version 5
  ids of `grade|LETTER`, and the seed was removed from the migration without a new one.
- **Seed only into an empty table.** Unlike geography, the bands have a rule across rows (the active ones cover 0-100 exactly once), so
  adding missing rows one by one could break a scale an Admin has changed. The script therefore adds the ten bands only when `Grades`
  holds no row at all, live or deleted, and otherwise adds nothing and says so. It takes the same advisory lock as a portal replacement.
  To reseed, remove the rows with `DELETE FROM "Grades"` (nothing refers to a grade); rows deleted in the screen still count.
- **Not a generic catalog.** Grade has its own `GradeScaleService`, `IGradeScaleStore`, and `GradeScaleController`. The generic flow saves
  one record at a time, and the 0-100 coverage rule spans records. The controller answers single-band writes with 405.
- **Replacement.** Every band is checked first (all errors at once), then the coverage of the active bands. The transaction then takes
  the advisory lock, re-reads the live bands, rejects ids that no longer exist, soft deletes the omitted bands and saves, and then applies
  updates and creations and saves again before committing. Saving deletions first lets a freed letter grade be reused in the same
  replacement; two existing bands swapping letters is refused rather than worked around.
- **SQLite.** The test host cannot order by a decimal and stores decimals as text, so the store orders the (small) scale in memory, and
  the SQLite check constraints compare numbers through `CAST(... AS REAL)`. PostgreSQL uses plain numeric checks.
- **SQLite has no seed.** The test host builds its schema without migrations, so API tests load the ten bands through the replacement;
  the seed itself is covered on PostgreSQL.
- **Editor.** `grades/grade-scale-editor` is a table of the bands with Add band, Remove… (confirmed in the row with "Yes, remove" or Cancel), Active, Save grade scale, and Discard changes. It
  sends `upperBoundInclusive` as true exactly for the band ending at 100, so the Admin never sets it by hand. It repeats the service's
  checks and messages before sending, shows service errors against the same band, and keeps edits on failure. A "Record history" card
  shows the latest change to any band.
- **Not done:** the hand-run scenarios 9-11 (T098). The migration is applied to the Development database.

