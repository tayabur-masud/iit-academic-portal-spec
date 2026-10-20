# Configuration API Contract

**Status**: Planning contract for [spec.md](../spec.md). The implementation publishes the generated OpenAPI document through the existing ASP.NET Core OpenAPI setup.

## Security and Common Behavior

- All routes require the authenticated session cookie and the session's **active** Admin role through `PortalPolicies.AdminModule`. A role assignment that is not active does not authorize access.
- State-changing requests require the existing `X-CSRF-Token` anti-forgery header.
- JSON request/response DTOs are explicit and do not expose EF entities or normalized comparison fields.
- Invalid values, duplicate normalized values, and invalid references return HTTP 400 `ValidationProblemDetails` with field-keyed errors. Anonymous requests return 401; authenticated non-Admin requests return 403.
- Create returns 201 with the resource representation and `Location`; read/update return 200. Updates use PUT with the full writable representation. Set `isActive: false` in PUT to deactivate. `DELETE /{id}` soft deletes an unreferenced record and returns 204; it returns 409 when live dependent records exist. The record is kept in the database but is no longer returned by any route (GET, PUT, and DELETE on it return 404), is excluded from lists and counts, and its name and code can be used again. Deletion never cascades; use deactivation for referenced records.
- Every successful mutation stages an allowlisted business event through the existing audit recorder in the same transaction. The client cannot submit audit events.

## Resources

| Collection | Writable fields | Special rules |
|---|---|---|
| `/api/admin/configuration/institutions` | `name`, `type`, optional `code`, optional `address`, `isActive` | `type`: `School`, `College`, or `University`. |
| `/api/admin/configuration/education-boards` | `name`, optional `code`, `isActive` | Independent of institutions and students. |
| `/api/admin/configuration/education-groups` | `name`, `isActive` | Independent catalog. |
| `/api/admin/configuration/examination-types` | `name`, `isActive` | Previous-education categories only. |
| `/api/admin/configuration/degrees` | `name`, optional `code`, required `creditsToBeCompleted`, required `minimumCgpaForCompletion`, `isActive` | Independent of Batch and Semester. Credits must be positive and have at most two fractional digits. The minimum CGPA must be greater than 0 and at most 4.00, with at most two fractional digits. |
| `/api/admin/configuration/grades` | `letterGrade`, `marksLowerBound`, `marksUpperBound`, `upperBoundInclusive`, `numericGrade`, `comment`, `isActive` | One global scale shared across Degrees; letter grade is unique within the Grade catalog; active bands cover 0–100 continuously without overlap. |
| `/api/admin/configuration/semesters` | `name`, `isActive` | Independent of Degree and Batch. |
| `/api/admin/configuration/academic-sessions` | `name`, `isActive` | Globally unique normalized name; a Session is distinct from authentication sessions and may be shared by Batches. |
| `/api/admin/configuration/designations` | `name`, optional `code`, `isActive` | No Teacher assignment in this feature. |
| `/api/admin/configuration/halls` | `name`, optional `code`, optional `address`, `isActive` | University residential halls. Independent of students and every other resource. |
| `/api/admin/configuration/batches` | `name`, `degreeId`, `startingSemesterId`, `academicSessionId`, `isActive` | All three references are required and must identify active records. Normalized name uniqueness is within Degree. |
| `/api/admin/configuration/geography/divisions` | `name`, `isActive` | Name is unique globally. Geographic records have no code field. |
| `/api/admin/configuration/geography/districts` | `divisionId`, `name`, `isActive` | Name is unique within Division; parent must exist and be active for new/reassigned children. |
| `/api/admin/configuration/geography/upazilas` | `districtId`, `name`, `isActive` | Name is unique within District; parent must exist and be active for new/reassigned children. |
| `/api/admin/configuration/geography/city-corporations` | `districtId`, `name`, `isActive` | Name is unique within City Corporations of the District; parent must exist and be active for new/reassigned records. No code field. |

For each collection, expose `GET` (list) and `POST` (create); for each `/{id}`, expose `GET` (detail), `PUT` (update/deactivate), and `DELETE` (soft delete if unreferenced). IDs are UUIDs. Responses add `id` and the persisted display fields, including `creditsToBeCompleted` for Degrees; they never return normalized fields. A blocked delete returns `409 Conflict` with a safe explanation and the option to deactivate; a missing ID returns 404.

## Collection Queries and Responses

- `search` is optional and matches normalized names; where relevant to academic catalogs, code search uses normalized values. Geography and Batch search use names only.
- `status` is `active` (default), `inactive`, or `all`.
- Geographic District lists accept `divisionId`; Upazila and City Corporation lists accept `districtId`. A parent filter never returns rows from another branch.
- Where a collection can grow, support bounded keyset paging using `pageSize` (default 25, maximum 100) and opaque `cursor`. Return `{ "items": [...], "nextCursor": "..." | null }`. Small bounded catalogs may return a single page while preserving the same response shape.
- Existing references resolve to their display record even when inactive; include `isActive` so the client can label them. New assignments to inactive values fail validation.

## Validation and Audit Events

- Duplicate academic-catalog names/codes and duplicate geographic names at the specified global/parent scope return field-specific validation errors after normalization; database uniqueness remains the final concurrency boundary.
- Degree create and update require `creditsToBeCompleted` greater than zero, with no more than two fractional digits; missing, zero, negative, or over-precision values return a field-keyed 400 on `creditsToBeCompleted`.
- Degree create and update require `minimumCgpaForCompletion` greater than 0 and at most 4.00, with no more than two fractional digits. A missing value returns a field-keyed 400 on `minimumCgpaForCompletion` ("Enter the minimum CGPA for completion."); zero, negative, over-4.00, or over-precision values return "Enter a CGPA greater than 0 and at most 4.00, with no more than two decimal places." An update that omits it is refused, not defaulted. Other catalogs refuse the field. Degree audit events record it.
- Missing, invalid, or inactive Batch Degree/Semester/Session references and duplicate normalized Batch names within a Degree return field-keyed validation errors without mutating data.
- Invalid/cross-branch geographic references return field-keyed validation errors without mutating data.
- A City Corporation requires one existing active District on create or reassignment; an unchanged inactive District reference may remain on update. Duplicate normalized City Corporation names are rejected within that District.
- A Grade scale mutation is accepted only when the resulting active bands cover 0–100 exactly once, with no gaps or overlaps. Invalid mutations return field-keyed errors and write no Grade or audit changes.
- Feature-owned event names follow `configuration.{resource}.{created|updated|deactivated|deleted}`. Metadata and change summaries are allowlisted and contain no unrelated personal data. A successful seed import records source citation/version and per-level counts in one bulk event.

### Grade Scale Replacement

- `GET /api/admin/configuration/grades` and `GET /api/admin/configuration/grades/{id}` use the standard list/detail behavior, including search and active/inactive/all status filtering.
- `PUT /api/admin/configuration/grades/scale` atomically replaces the desired Grade set. The request contains the complete desired band list; an item with an existing `id` updates that band, an item without an `id` creates a band, an included inactive band is deactivated, and an existing unreferenced band omitted from the list is soft-deleted.
- Each band contains `letterGrade`, `marksLowerBound`, `marksUpperBound`, `upperBoundInclusive`, `numericGrade`, `comment`, and `isActive`. The letter grade is globally unique within the Grade catalog. Percentage bounds are decimal numbers from 0 through 100; lower bounds are inclusive and upper bounds are exclusive except the band ending at 100, which sets `upperBoundInclusive: true`. `numericGrade` is greater than or equal to 0 and at most 4.00 with at most two decimal places.
- The active subset of the complete request must cover every mark from 0 through 100 exactly once. It must be sorted by `marksLowerBound`, start at 0, have each next lower bound equal to the previous upper bound, and end at 100 inclusive. No cross-Degree field is accepted.
- Scale validation, every Grade row change, soft deletes, and all corresponding audit events commit in one transaction. Concurrent scale replacement requests are serialized before validating the resulting scale.

## Implementation Notes

- Every record in a response also carries `createdAtUtc`, `createdByUserId`, `createdBy`, `lastModifiedAtUtc`, `lastModifiedByUserId`, and `lastModifiedBy` (2026-10-14). The times are UTC and always present. The account ids are omitted when no signed-in account made the change (a seed script or a background process). `createdBy` and `lastModifiedBy` are the accounts' current emails, resolved when the record is read and omitted when the account was removed. They are set by the server and are not writable: a request that contains any of them is rejected with 400.

Recorded when the feature was built (2026-10-14):

- Requests with an unknown JSON field (for example `actorUserId`) are rejected with 400; the request model has only the writable fields above.
- A `PUT` that changes nothing returns 200 and records no audit event. `isActive` may be omitted on `PUT` to keep the current value; on `POST` it defaults to `true`.
- A reference that is unchanged on `PUT` is accepted even if its record has since been deactivated; a new or changed reference must be active.
- Collection routes accept `search`, `status`, `pageSize`, `cursor`, `page`, and (districts, upazilas, and city corporations) `divisionId` or `districtId`. The Angular Admin screens use cursor mode with infinite loading; they do not show numbered pages, totals, or a page-size selector. `page` (1-based) remains an API compatibility mode for non-UI clients and returns `page`, `totalCount` (everything matching the search, status, and parent filter), and `totalPages`. It cannot be combined with `cursor` (400 on `page`). A page past the end returns an empty `items` list with the totals. Cursor mode omits the totals and cannot shift while records arrive; numbered pages can shift if records are added or removed while someone browses.
- A duplicate that races past the application check is stopped by the unique index and returned as a 400 on `name`.
- `POST .../geography/import/validate` returns the report with 200 when the document is valid and 400 when it is not. The report has `isValid`, `canImport`, `targetEmpty`, `sourceCitation`, `sourceVersion`, `divisions`, `districts`, `upazilas`, `cityCorporations` (0 when the document lists none), and `issues` (each with a `path` such as `divisions[2].districts[0].name`). Validation never writes, and a valid document into non-empty tables reports `canImport: false`.

## Geographic Seed Import

The owner-provided JSON body uses [geographic-seed.schema.json](geographic-seed.schema.json). Each District may optionally include a `cityCorporations` array of name-only records; omitting it means the source provides none for that District. Both operations require the authenticated session's active Admin role and the existing anti-forgery token:

| Method and route | Behavior |
|---|---|
| `POST /api/admin/configuration/geography/import/validate` | Validate source citation/version, normalized-name uniqueness, and all parent links; return a complete report without writing data. |
| `POST /api/admin/configuration/geography/import` | Repeat complete validation, require empty target geography tables, then atomically insert the hierarchy and audit event; return counts for divisions, districts, upazilas, and city corporations. |

Invalid schema or semantic data returns 400 with the validation report and writes nothing. Non-empty target tables return 409; the importer never overwrites Admin-managed rows. The owner-approved source file is required before release acceptance.

## City Corporation Implementation Notes (2026-10-20)

- **Route and shape.** `/api/admin/configuration/geography/city-corporations` with the generic `name`, `districtId`, and `isActive` fields.
  A `code`, `divisionId`, or any other catalog field is a field-keyed 400; responses never include a code.
- **Validation messages.** A missing District is a 400 on `districtId`; an unknown, deleted, or non-District identifier "does not exist"; a
  new or changed District that is inactive is refused. A duplicate is a 400 on `name`: "A city corporation with this name already exists
  in this district." An Upazila of the same name in the same District does not conflict.
- **Lifecycle.** Deactivate with `PUT ... isActive: false`; delete is a soft delete with no dependents of its own. A District with a live
  City Corporation (active or inactive) cannot be deleted (409, "Deactivate it instead"); one that has only deleted City Corporations can.
- **Audit.** `configuration.city-corporation.{created|updated|deactivated|deleted}` with `name`, `isActive`, and `districtId`. The import's
  `configuration.geography.imported` metadata adds `cityCorporations`.
- **Import.** An optional `cityCorporations` list under each District. Each entry is an object with only `name`; a non-list, a non-object
  entry, a blank name, a code, or a normalized duplicate within the District is reported at its path, for example
  `divisions[0].districts[0].cityCorporations[1].name`. A live City Corporation also makes the target not empty.

## Grade Scale Implementation Notes (2026-10-21)

- **Reads.** `GET /api/admin/configuration/grades` takes `search` (letter grade) and `status` (`active` default, `inactive`, `all`; another
  value is a 400 on `status`) and returns the whole matching scale as `{ "items": [...], "nextCursor": null }`, highest lower bound first.
  `GET /grades/{id}` returns one band or 404. Each band has `id`, `letterGrade`, `marksLowerBound`, `marksUpperBound`,
  `upperBoundInclusive`, `numericGrade`, `comment`, `isActive`, and the created/last-modified facts; no Degree field exists.
- **The only write.** `PUT /grades/scale` with `{ "bands": [...] }` returns the saved scale (all statuses) in the list shape. `POST /grades`,
  `PUT /grades/{id}`, and `DELETE /grades/{id}` answer `405` with `Allow: GET` and a detail pointing to `PUT /grades/scale`.
- **Error keys.** `bands` for the scale as a whole (an empty request, no active band, and every gap, overlap, missing 0%, or missing 100%,
  each naming the uncovered or overlapping percentages, for example "No active band covers 75% to 76%."), and `bands[i].field` for band
  `i` of the request: `letterGrade` (missing, over 20 characters, or repeated, compared after normalizing), `marksLowerBound` and
  `marksUpperBound` (missing, outside 0-100, upper not above lower, and the band where a gap or overlap begins), `upperBoundInclusive`
  (set below 100, or the top band not including 100), `numericGrade` (missing, outside 0.00-4.00, or more than two decimals), `comment`
  (blank or over 500 characters), and `id` (repeated in the request, or no longer a live band). Unknown fields, such as `degreeId`, are a
  400. A refused replacement changes no band and records no event.
- **Audit.** One event per changed band: `configuration.grade.created`, `.updated`, `.deactivated`, or `.deleted`, with entity type `Grade`
  and the allowlisted fields `letterGrade`, `marksLowerBound`, `marksUpperBound`, `upperBoundInclusive`, `numericGrade`, `comment`, and
  `isActive`. Resubmitting an unchanged scale records nothing.
- **Order of writes.** Bands left out are soft deleted and saved first, then updates and creations, all in one transaction, so a letter
  grade freed by a deleted band can be reused in the same replacement. Two existing bands swapping letter grades in one replacement is
  refused with a `bands` error; do it in two saves.
- **Serialization.** On PostgreSQL the replacement takes `pg_advisory_xact_lock(3003014)` before reading the scale, so concurrent
  replacements run one after the other and each is checked against the scale the previous one saved.

