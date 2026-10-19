# Implementation Notes: Course Management

**Feature**: [spec.md](spec.md) | **Tasks**: [tasks.md](tasks.md) | **Validation**: [quickstart.md](quickstart.md)

## Shared catalog changes

- `ConfigurationResource<T>` gained three per-resource hooks, all off by default so other catalogs behave as before: `SupportsDelete`
  (Course answers `DELETE` with 405 and an `Allow` header), `NamesAreUnique` (Course names may repeat; a duplicate-index race is then
  reported on `code`), and `UpdateConflictAsync` (a 409 decided after validation, before anything is staged or saved).
- The list's parent filter now also accepts `degreeId`, mapped to Course's Degree.
- Lists of typed catalogs filter by `type`: a `TypeFilter` hook (Course and Institution override it; the default refuses with a
  `type` error), combined with the parent filter into one database expression. The list screen shows a Type filter for any resource with
  a type.
- Lists can sort by a property other than the name: `SortProperty` and `SortKey` (the value the cursor carries), passed through
  `IConfigurationStore.ListAsync`. Course sorts by normalized code then ID, in both cursor and numbered paging; other catalogs still sort
  by name.
- `ConfigurationAuditEventDefinitions.ApprovedFields` adds the Course fields, and also `creditsToBeCompleted`, which the Degree audit
  snapshot already wrote but was being dropped.
- A Degree that has courses can no longer be deleted.

## Course rules

- Validation lives in `CourseResource`. The prerequisite cycle check walks the chain one link at a time through
  `IConfigurationStore.PrerequisiteOfAsync`, never loading the catalog; `HasActiveDependentCoursesAsync` answers both the
  deactivation and the Degree-change questions.
- A Degree change is checked both ways: this course's prerequisite must be in the new Degree (reported on `prerequisiteCourseId`), and no
  active course that depends on it may be left in the old Degree (reported on `degreeId`).
- Reactivating a course requires its prerequisite to be active; an unchanged prerequisite may otherwise stay as it is.
- The database repeats the credit rules as check constraints, forbids a self-prerequisite, keeps code uniqueness over live rows (inactive
  courses included), and uses restrictive keys for Degree and prerequisite.

## Frontend

- The generic catalog descriptor gained `codeRequired`, `typeOptions`, `isCourse`, `canDelete`, and the `degreeId` parent filter. The
  form shows the credit fields and a prerequisite selector loaded from the active courses of the chosen Degree (never the course itself;
  a saved prerequisite stays selectable), and hides the delete section for courses. The list adds a Degree filter and a credit-hours
  column; for a resource with a required code (courses), Code is the first column and links to the record.
- Adding the course screens pushed the initial bundle to 492.72 kB of 500 kB, so the Configuration routes are now lazy-loaded like the
  other Admin screens; the initial bundle is 454.63 kB.

## Not done

- Manual quickstart scenarios (the migration is applied to the Development database).
- The SC-006 usability study (T017).
