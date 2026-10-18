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
| `/api/admin/configuration/degrees` | `name`, optional `code`, required `creditsToBeCompleted`, `isActive` | Independent of Batch and Semester. Credits must be positive and have at most two fractional digits. |
| `/api/admin/configuration/semesters` | `name`, `isActive` | Independent of Degree and Batch. |
| `/api/admin/configuration/academic-sessions` | `name`, `isActive` | Globally unique normalized name; a Session is distinct from authentication sessions and may be shared by Batches. |
| `/api/admin/configuration/designations` | `name`, optional `code`, `isActive` | No Teacher assignment in this feature. |
| `/api/admin/configuration/halls` | `name`, optional `code`, optional `address`, `isActive` | University residential halls. Independent of students and every other resource. |
| `/api/admin/configuration/batches` | `name`, `degreeId`, `startingSemesterId`, `academicSessionId`, `isActive` | All three references are required and must identify active records. Normalized name uniqueness is within Degree. |
| `/api/admin/configuration/geography/divisions` | `name`, `isActive` | Name is unique globally. Geographic records have no code field. |
| `/api/admin/configuration/geography/districts` | `divisionId`, `name`, `isActive` | Name is unique within Division; parent must exist and be active for new/reassigned children. |
| `/api/admin/configuration/geography/upazilas` | `districtId`, `name`, `isActive` | Name is unique within District; parent must exist and be active for new/reassigned children. |

For each collection, expose `GET` (list) and `POST` (create); for each `/{id}`, expose `GET` (detail), `PUT` (update/deactivate), and `DELETE` (soft delete if unreferenced). IDs are UUIDs. Responses add `id` and the persisted display fields, including `creditsToBeCompleted` for Degrees; they never return normalized fields. A blocked delete returns `409 Conflict` with a safe explanation and the option to deactivate; a missing ID returns 404.

## Collection Queries and Responses

- `search` is optional and matches normalized names; where relevant to academic catalogs, code search uses normalized values. Geography and Batch search use names only.
- `status` is `active` (default), `inactive`, or `all`.
- Geographic District lists accept `divisionId`; Upazila lists accept `districtId`. A parent filter never returns rows from another branch.
- Where a collection can grow, support bounded keyset paging using `pageSize` (default 25, maximum 100) and opaque `cursor`. Return `{ "items": [...], "nextCursor": "..." | null }`. Small bounded catalogs may return a single page while preserving the same response shape.
- Existing references resolve to their display record even when inactive; include `isActive` so the client can label them. New assignments to inactive values fail validation.

## Validation and Audit Events

- Duplicate academic-catalog names/codes and duplicate geographic names at the specified global/parent scope return field-specific validation errors after normalization; database uniqueness remains the final concurrency boundary.
- Degree create and update require `creditsToBeCompleted` greater than zero, with no more than two fractional digits; missing, zero, negative, or over-precision values return a field-keyed 400 on `creditsToBeCompleted`.
- Missing, invalid, or inactive Batch Degree/Semester/Session references and duplicate normalized Batch names within a Degree return field-keyed validation errors without mutating data.
- Invalid/cross-branch geographic references return field-keyed validation errors without mutating data.
- Feature-owned event names follow `configuration.{resource}.{created|updated|deactivated|deleted}`. Metadata and change summaries are allowlisted and contain no unrelated personal data. A successful seed import records source citation/version and per-level counts in one bulk event.

## Implementation Notes

- Every record in a response also carries `createdAtUtc`, `createdByUserId`, `createdBy`, `lastModifiedAtUtc`, `lastModifiedByUserId`, and `lastModifiedBy` (2026-10-14). The times are UTC and always present. The account ids are omitted when no signed-in account made the change (a seed script or a background process). `createdBy` and `lastModifiedBy` are the accounts' current emails, resolved when the record is read and omitted when the account was removed. They are set by the server and are not writable: a request that contains any of them is rejected with 400.

Recorded when the feature was built (2026-10-14):

- Requests with an unknown JSON field (for example `actorUserId`) are rejected with 400; the request model has only the writable fields above.
- A `PUT` that changes nothing returns 200 and records no audit event. `isActive` may be omitted on `PUT` to keep the current value; on `POST` it defaults to `true`.
- A reference that is unchanged on `PUT` is accepted even if its record has since been deactivated; a new or changed reference must be active.
- Collection routes accept `search`, `status`, `pageSize`, `cursor`, `page`, and (districts and upazilas) `divisionId` or `districtId`. `page` (1-based) is the numbered-page mode the Admin screens use: the response then also carries `page`, `totalCount` (everything matching the search, status, and parent filter), and `totalPages`. It cannot be combined with `cursor` (400 on `page`). A page past the end returns an empty `items` list with the totals. Cursor mode omits the totals and cannot shift while records arrive; numbered pages can shift if records are added or removed while someone browses.
- A duplicate that races past the application check is stopped by the unique index and returned as a 400 on `name`.
- `POST .../geography/import/validate` returns the report with 200 when the document is valid and 400 when it is not. The report has `isValid`, `canImport`, `targetEmpty`, `sourceCitation`, `sourceVersion`, `divisions`, `districts`, `upazilas`, and `issues` (each with a `path` such as `divisions[2].districts[0].name`). Validation never writes, and a valid document into non-empty tables reports `canImport: false`.

## Geographic Seed Import

The owner-provided JSON body uses [geographic-seed.schema.json](geographic-seed.schema.json). Both operations require the authenticated session's active Admin role and the existing anti-forgery token:

| Method and route | Behavior |
|---|---|
| `POST /api/admin/configuration/geography/import/validate` | Validate source citation/version, normalized-name uniqueness, and all parent links; return a complete report without writing data. |
| `POST /api/admin/configuration/geography/import` | Repeat complete validation, require empty target geography tables, then atomically insert the hierarchy and audit event; return per-level counts. |

Invalid schema or semantic data returns 400 with the validation report and writes nothing. Non-empty target tables return 409; the importer never overwrites Admin-managed rows. The owner-approved source file is required before release acceptance.
