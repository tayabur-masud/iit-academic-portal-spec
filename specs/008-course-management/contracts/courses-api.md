# Course Catalog API Contract

**Status**: Implemented. See [Implemented details](#implemented-details).

The Course catalog uses the existing Admin configuration REST API conventions. JSON property names are camelCase.

## Authorization

- Every operation requires an authenticated session actively using the Admin role.
- Every state-changing request requires the existing `X-CSRF-Token` anti-forgery header.
- Unauthorized requests return the standard 401 or 403 problem response and do not disclose Course data.

## Endpoints

### List and search

`GET /api/admin/configuration/courses`

Query parameters:

| Name | Description |
|---|---|
| `search` | Optional normalized search against Course name or code. |
| `status` | `active` (default), `inactive`, or `all`. |
| `degreeId` | Optional Degree filter, used to load valid prerequisite choices. |
| `pageSize` | Optional page size from 1 to 100; default 25. |
| `cursor` | Optional continuation token for cursor paging. |
| `page` | Optional 1-based numbered page; cannot be combined with `cursor`. |

`200 OK` returns the shared page shape: `items`, `nextCursor`, and, for numbered paging, `page`, `totalCount`, and `totalPages`.

### Read

`GET /api/admin/configuration/courses/{id}`

`200 OK` returns the Course fields and standard created/modified audit metadata. Unknown IDs return `404`.

### Create

`POST /api/admin/configuration/courses`

Example request:

```json
{
  "name": "Applied Mathematics I",
  "code": "MATH101",
  "creditHours": 3,
  "theoryCreditHours": 2,
  "labCreditHours": 1,
  "type": "Mandatory",
  "degreeId": "00000000-0000-0000-0000-000000000001",
  "prerequisiteCourseId": null,
  "isActive": true
}
```

`201 Created` returns the saved Course and a resource `Location`. `isActive` defaults to true when omitted. Theory and lab values may be omitted independently; each supplied value must be between zero and `creditHours`, inclusive, and if both are supplied, their sum must equal `creditHours`.

### Update and deactivate

`PUT /api/admin/configuration/courses/{id}`

The request contains the editable Course fields. Send `isActive: false` to deactivate; omit `prerequisiteCourseId` or send `null` to clear the optional prerequisite according to the shared update contract.

`200 OK` returns the saved Course. Attempting to deactivate a Course used as a prerequisite by any active Course returns `409 Conflict` and leaves all Course data unchanged.

Changing a Course's Degree returns a field-keyed `400 Bad Request` if the change would make its prerequisite or an active dependent Course's prerequisite cross-Degree; no affected Course is modified.

There is no Course `DELETE` operation in this feature. Use deactivation to remove a Course from active selections while preserving its catalog and audit history.

## Validation and Errors

- `400 Bad Request`: field-keyed validation problem for missing or invalid fields, duplicate code, invalid credit totals/breakdown, invalid type, or invalid Degree/prerequisite relationship.
- `401 Unauthorized` / `403 Forbidden`: the request has no authenticated Admin session or the active role is not Admin.
- `404 Not Found`: the Course or a referenced Degree/Course does not exist.
- `409 Conflict`: deactivation is blocked because active Courses depend on the target; also used for a uniqueness race rejected by the database.

Validation and conflict responses use the portal's standard Problem Details shape. A rejected request must not persist the Course change or its business audit event.

## Audit

Course create, update, and deactivation write standard business audit events atomically with the Course change. Audit actor and timestamps are server supplied. Existing Admin audit search/detail APIs remain the audit-history interface; this feature adds no separate audit endpoint.

## Implemented details

- **Type filter.** `GET .../courses?type=Mandatory|Elective` (case-insensitive) combines with `search`, `status`, `degreeId`, and
  both paging modes; another value, including a number, is a field-keyed 400 on `type`. Institutions take `type=School|College|University`
  the same way; other catalogs refuse `type`.
- **Order.** The list is ordered by normalized code then ID, in both cursor and numbered paging (other catalogs order by name).
- **Fields.** Request and response use `name`, `code`, `creditHours`, `theoryCreditHours`, `labCreditHours`, `type`, `degreeId`,
  `prerequisiteCourseId`, and `isActive`; responses add `id` and the standard created/modified facts. Absent optional values are omitted
  from responses. Other catalog fields (for example `address` or `creditsToBeCompleted`) are a field-keyed 400 on a Course, and the
  Course fields are a field-keyed 400 on every other catalog.
- **Field keys of rejections.** `code` (missing or duplicate), `type` (missing, or not `Mandatory`/`Elective`; numbers are refused),
  `creditHours`, `theoryCreditHours`, `labCreditHours` (a part over the total, or a sum that differs from the total, is reported on
  `labCreditHours`), `degreeId` (missing, unknown, inactive, or a change that would leave an active dependent course in another Degree),
  and `prerequisiteCourseId` (missing, inactive while the course is active, another Degree, itself, or a cycle).
- **Prerequisite on reactivation.** Turning a course active again requires its prerequisite to be active.
- **Deactivation conflict.** `409` with `detail` "Active courses have this course as their prerequisite, so it cannot be deactivated...";
  nothing is saved or audited.
- **No delete.** `DELETE /api/admin/configuration/courses/{id}` answers `405 Method Not Allowed` with `Allow: GET, POST, PUT` and a
  Problem Details `detail` of "A course cannot be deleted. Deactivate it instead."
- **Race on a duplicate code.** If two saves race past the check, the database's unique index wins and the loser gets a field-keyed 400
  on `code`.
- **Degree deletion.** A Degree that has courses cannot be deleted (409, "Deactivate it instead").
- **Audit.** Events `configuration.course.created`, `configuration.course.updated`, and `configuration.course.deactivated` record
  `name`, `code`, `isActive`, `type`, `degreeId`, `creditHours`, `theoryCreditHours`, `labCreditHours`, and `prerequisiteCourseId`.

