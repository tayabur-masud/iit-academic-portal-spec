# Implementation Notes: Configuration

**Branch**: `feature/003-configuration` (spec repo and both submodules) · **Recorded**: 2026-10-14 · **Not committed or pushed**

## Status

| Story | State |
|---|---|
| US1 Academic catalogs (8) | Implemented and tested (service, PostgreSQL, Angular, live API and UI) |
| US2 Batches | Implemented and tested |
| US3 Geography and seed import | Implemented and tested; the owner's files were imported into the Development database on 2026-10-14 (8 divisions, 64 districts, 492 upazilas), **but they cite no official source** |
| US4 Protection and references | Implemented and tested (authorization, atomic audit, reference-safe delete on PostgreSQL) |

All 37 tasks are done. T034 was run with the owner's files; its record, including the three data changes and the caveat that no
official source was cited, is the Geographic Import Record in `quickstart.md`. Results are in the Verification Record there.

## How it was built, and where it differs from the task text

The twelve resources share one flow instead of twelve copies, so the files differ from the paths the tasks named. The
behavior, routes, and rules are unchanged.

| Task text | What exists |
|---|---|
| Per-resource entities (T004 to T007, T014, T021) | Twelve entities in `Domain/Configuration/` over a shared `ConfigurationRecord` (identifier, name, normalized name, active flag) |
| `CatalogService.cs`, `BatchService.cs`, `GeographyService` (T009, T016, T024) | `ConfigurationResource<T>` holds the one create, read, update, deactivate, and delete flow; each resource supplies only what differs (extra fields, the scope of name uniqueness, the references it validates). `CatalogService` is the entry point and `ConfigurationResources` the registry |
| `CatalogContracts.cs`, `BatchContracts.cs` (Application) | `ConfigurationModels.cs`; the API has its own explicit DTOs in `Api/Contracts/Configuration/` |
| Three migrations (T008, T015, T023) | **One** migration, `ConfigurationReferenceData` (the plan asked for one reviewed migration) |
| Three controllers | `CatalogsController`, `BatchesController`, `GeographyController`, all on a shared `ConfigurationControllerBase` |
| Separate Angular list and form files for catalogs, batches, and geography (T011, T018, T026) | One generic list and form in `catalogs/`, driven by the descriptors in `configuration.models.ts`; batches and the cascading Division, District, Upazila selection are descriptor features of the same components. A landing page groups the resources as IIT academics, Other catalogs, and Geography (regrouped 2026-10-14 at the owner's request; batches had been alone in a section) |
| `ConfigurationAuditEventDefinitions` (T003) | Done, generating the 48 `configuration.{resource}.{action}` definitions plus `configuration.geography.imported` |
| Maximum lengths (data-model left open) | Name 200, code 50, address 500, enforced by the API and the database |

Decisions made while building, not stated in the spec:

- Configuration lists use the same numbered pager as audit history (decided 2026-10-14): the service takes a `page` parameter and
  returns the total count and page count, the cursor mode stays available, and the pager is a shared component used by both screens.
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

See the Verification Record in `quickstart.md`: 467 service tests with PostgreSQL, 103 frontend tests, 35 live API
checks, and a live headless-Chrome run with axe-core on every Configuration screen.

## Not done

- **An approved official source for the geography**: the imported data carries a citation saying none was supplied, so SC-005 and FR-014's
  "approved official source" are not claimed.
- The import was run on the Development database only; staging and production need the same flow (the seed is kept outside the repo).
- A screen-reader pass and a keyboard-only walkthrough by a person.
- The SC-006 usability sessions (95% of Admins find and update or deactivate a record within two minutes).
- The geography import has no screen; it is used through Swagger as the quickstart describes. Say so if a screen is wanted.
- No numeric performance target exists in the spec; lists are bounded (page size 25, maximum 100) and cursor-paged.
