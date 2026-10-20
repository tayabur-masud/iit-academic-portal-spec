# Data Model: Configuration

**Feature**: [spec.md](spec.md)  
**Research decisions**: [research.md](research.md)

## Shared Conventions

- Use server-generated `Guid` identifiers, following existing portal records.
- Every entity derives from the shared `AuditedEntity`, so it also carries `CreatedAtUtc`, `CreatedByUserId`, `LastModifiedAtUtc`, and `LastModifiedByUserId` (times required and UTC, defaulting to the database clock for rows added by SQL; account ids optional, at most 450 characters). The database context stamps them on every save from the server clock and the signed-in account; on insert the last-modified pair equals the created pair; the account is null for a background change or a seed script, and rows that existed before the columns show the migration time and no account.
- Every entity also carries `IsDeleted` (default false), nullable `DeletedAtUtc` (UTC), and nullable `DeletedByUserId` (the stable account id). A delete sets these and keeps the row; a global query filter hides deleted rows from every query, and the unique indexes below apply only where `IsDeleted` is false.
- Persist `IsActive` on every catalog, Batch, and geographic entity; new records default to active.
- Persist `NormalizedName` alongside the display `Name`. Normalize by trimming, collapsing repeated whitespace, and invariant case normalization. Preserve the display name as entered after trimming.
- Where a code is optional, store `NormalizedCode` as `NULL` when omitted or whitespace-only; otherwise trim and invariant-case-normalize it while preserving the display `Code`.
- Division, District, Upazila, and CityCorporation have no code fields; geographic uniqueness is based on normalized names and parent scope.
- Do not expose normalized comparison fields as client-editable DTO fields.
- Exact maximum string lengths are not specified by the feature. Set consistent API/domain and database limits during implementation, based on approved seed values where applicable.

## Entities

| Entity | Fields | Relationships and constraints |
|---|---|---|
| `EducationalInstitution` | `Id`, `Name`, `NormalizedName`, `Type` (`School`, `College`, `University`), nullable `Code`/`NormalizedCode`, nullable `Address`, `IsActive` | Independent catalog. Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type. |
| `EducationBoard` | `Id`, `Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, `IsActive` | Independent catalog. Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type. |
| `EducationGroup` | `Id`, `Name`, `NormalizedName`, `IsActive` | Independent catalog. Unique `NormalizedName`. |
| `ExaminationType` | `Id`, `Name`, `NormalizedName`, `IsActive` | Independent previous-education category. Unique `NormalizedName`; not an IIT degree or student-history record. |
| `Degree` | `Id`, `Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, required `CreditsToBeCompleted` (`numeric(8,2)`), required `MinimumCgpaForCompletion` (`numeric(3,2)`), `IsActive` | Independent IIT program catalog. Credits must be greater than zero, with at most two fractional digits. The minimum CGPA must be greater than 0 and at most 4.00, with at most two fractional digits (check `CK_Degrees_MinimumCgpaForCompletion_Range`). Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type. Student Management uses the value as the degree's required-credit total; this catalog does not store earned credits. |
| `Semester` | `Id`, `Name` (display label), `NormalizedName`, `IsActive` | Independent catalog. Unique `NormalizedName`. No order field is needed by an existing domain rule; sort consistently by normalized name unless an approved requirement establishes ordering. |
| `AcademicSession` | `Id`, `Name` (display label), `NormalizedName`, `IsActive` | Independent academic-session catalog, distinct from authentication `AuthSession`. Unique `NormalizedName` globally. |
| `Designation` | `Id`, `Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, `IsActive` | Independent catalog. Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type. |
| `Hall` | `Id`, `Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, nullable `Address`, `IsActive` | Independent university residential hall catalog. Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type; address at most 500 characters. No relationships. |
| `Batch` | `Id`, `Name`, `NormalizedName`, `DegreeId`, `StartingSemesterId`, required `AcademicSessionId`, `IsActive` | Exactly one `Degree`, one starting `Semester`, and one `AcademicSession`. Unique (`DegreeId`, `NormalizedName`); a different Degree may reuse the name. All references use restrictive delete behavior. |
| `Division` | `Id`, `Name`, `NormalizedName`, `IsActive` | Top geographic level. Unique `NormalizedName` globally. |
| `District` | `Id`, `DivisionId`, `Name`, `NormalizedName`, `IsActive` | Exactly one Division. Unique (`DivisionId`, `NormalizedName`). Restrict physical parent deletion. |
| `Upazila` | `Id`, `DistrictId`, `Name`, `NormalizedName`, `IsActive` | Exactly one District. Unique (`DistrictId`, `NormalizedName`). Restrict physical parent deletion. |
| `CityCorporation` | `Id`, `DistrictId`, `Name`, `NormalizedName`, `IsActive` | Exactly one District. Unique (`DistrictId`, `NormalizedName`) among City Corporations; no code. Restrict physical parent deletion. Added by the `CityCorporationsDegreeMinimumCgpaAndGrades` migration (table `CityCorporations`, unique index `IX_CityCorporations_DistrictId_NormalizedName` over live rows, which also serves District lookups, and a restrictive key to `Districts`). |
| `Grade` | `Id`, `Name` (letter grade), `NormalizedName`, `MarksLowerBound`, `MarksUpperBound`, `UpperBoundInclusive`, `NumericGrade` (`numeric(3,2)`), `Comment`, `IsActive` | One global scale shared across Degrees. Unique normalized letter-grade name among live rows. Active bands form a continuous, non-overlapping partition of 0–100; lower bounds are inclusive, upper bounds exclusive except the top bound 100. No Degree/student/course relationship. |

### Degree Credit Migration Prerequisite

The Degree credit column is required for new and updated records. Before applying a migration that makes it non-null, provide an owner-approved positive credit total for every existing Degree row. The migration MUST NOT assign guessed values, zero, or a universal default. If existing degrees have no approved mapping, keep the migration unapplied until the mapping is supplied.

## Seed Data

`Hall` has an owner-supplied seed: `iit-academic-portal-service/database/postgresql/seed-halls.sql` loads 23 University of Dhaka halls and
hostels with fixed version 5 UUIDs derived from the normalized name, the default address "University of Dhaka, Dhaka-1000, Bangladesh",
and no code. It uses `ON CONFLICT DO NOTHING`, so it can be run again on any database without duplicating or overwriting rows, and it is
not audited (it does not go through the application). A seeded row an Admin deleted stays deleted (its row and id remain), the created and
last-modified times take the database default and the "by" columns stay empty (shown as System).

Divisions, Districts, Upazilas, and City Corporations also have owner-supplied seeds: `seed-divisions.sql` (8), `seed-districts.sql` (64), `seed-upazilas.sql` (492),
and `seed-city-corporations.sql` (13, added 2026-10-21 from the owner's list; Dhaka district has two) in the same folder, run in that order. Their ids are fixed version 5 UUIDs derived from the level and the normalized names; a child's parent is found by normalized name; they use
`ON CONFLICT DO NOTHING` and are not audited. The source files carried no official citation, and two exact repeats in Natore were left out.

## Validation and Lifecycle

- Normalized names are unique inside their own catalog. Equal names in different catalogs are allowed.
- Optional normalized codes are unique inside their own academic catalog when non-null. Geographic entities have no code fields.
- `Degree.CreditsToBeCompleted` is required, positive, and limited to two fractional decimal places. Create/update validation returns a field error for missing, zero, negative, or over-precision values.
- `Degree.MinimumCgpaForCompletion` is required, greater than 0, at most 4.00, and limited to two fractional decimal places. Create/update validation returns a field error for a missing, zero, negative, over-4.00, or over-precision value; an update must send it again. The `CityCorporationsDegreeMinimumCgpaAndGrades` migration sets 2.50 on every existing Degree and leaves no column default. Graduation eligibility does not use it yet.
- Division normalized names are unique globally; District, Upazila, and CityCorporation normalized names are unique within their parent and their own entity type.
- Academic Session normalized names are unique globally. Multiple Batches may reference one Session.
- A Batch normalized name is unique only within its Degree; its required Session does not alter that scope.
- A District may be created or reassigned only under an existing active Division. An Upazila or CityCorporation may be created or reassigned only under an existing active District. Batch writes require active Degrees, Semesters, and Academic Sessions.
- Deactivation preserves identifiers and existing relationships. Existing-reference views may show inactive status; new assignments to inactive records are rejected. Deactivation never cascades to children or dependents.
- Foreign keys use restrictive/no-action delete behavior, so the database still refuses to physically delete a referenced row. The application never deletes a row: an Admin soft deletes a record (sets `IsDeleted`, `DeletedAtUtc`, `DeletedByUserId`) only when no live record references it (Degree, Semester, and Academic Session are referenced by Batches; Division by Districts; District by Upazilas and CityCorporations); otherwise the delete is rejected and the record can be deactivated instead. A deactivated dependent still blocks the delete; a deleted one does not.
- A deleted record is hidden everywhere: it is not listed or found by search, cannot be opened, edited, or deleted again (404), cannot be chosen as a parent or reference, and does not count toward uniqueness, so its name and code can be used again. It cannot be restored from the portal.
- Successful configuration creates, updates, deactivations, deletions, and their allowlisted business audit events are persisted in one database transaction. A failed audit persistence must roll back the configuration write.

## Geographic Seed Input

The seed document contains a source citation/version and nested Divisions → Districts → Upazilas, with an optional `cityCorporations` name list under each District. Its structural contract is [geographic-seed.schema.json](contracts/geographic-seed.schema.json). Semantic validation additionally enforces normalized uniqueness, parent links, and the approved source/version. The full document is validated before a single import transaction; no partial records are committed. No City Corporation rows are invented when the approved owner source omits them.

## Grade Scale

- Store one Grade row per band. `Name` is the letter grade and its `NormalizedName` is unique among live Grade rows. `MarksLowerBound` is inclusive; `MarksUpperBound` is exclusive except for the uppermost band, which includes 100 through `UpperBoundInclusive`. Percentage bounds are decimal values between 0 and 100; no extra decimal-rounding rule is imposed by the feature. `NumericGrade` is a decimal between 0.00 and 4.00 with at most two decimal places; `Comment` is required.
- The active Grade rows form one continuous, non-overlapping partition covering every percentage from 0 through 100 exactly once. Inactive Grade rows are excluded from this partition but remain identifiable. A mutation that would violate this invariant is rejected without changing the scale.
- Seed the ten screenshot bands (A+ 80–100, A 75–80, A- 70–75, B+ 65–70, B 60–65, B- 55–60, C+ 50–55, C 45–50, D 40–45, F 0–40) and their supplied numeric grades/comments with the seed script `iit-academic-portal-service/database/postgresql/seed-grades.sql` (owner decision 2026-10-21: a seed script like the geography data, not a migration), using stable identifiers. The script seeds only an empty `Grades` table, so it never updates, adds beside, or restores rows an Admin has changed or deleted; the migration creates the table and its checks only.
- Grade has no relationship to Degree, student, or course in this feature. Soft-delete only unreferenced bands; changes that deactivate/delete a band must also provide the replacement active bands needed to preserve complete coverage.
- Apply the entire desired Grade set atomically through the Grade scale write workflow. Serialize writes in PostgreSQL, validate the resulting active partition while holding the serialization guard, and save all row and audit changes in the same transaction.

### Grade storage as implemented (2026-10-21)

- Table `Grades` with the shared configuration columns; `Name` is the letter grade (at most 20 characters by validation).
  `MarksLowerBound` and `MarksUpperBound` are unconstrained `numeric` (no rounding); `NumericGrade` is `numeric(3,2)`; `Comment` is
  required, at most 500 characters.
- Checks: `CK_Grades_Marks_Range` (lower >= 0, upper <= 100, lower < upper), `CK_Grades_UpperBoundInclusive_At100` (only a band ending at
  100 may include it), `CK_Grades_NumericGrade_Range` (0 to 4), and `CK_Grades_Comment_NotBlank`. The unique index
  `IX_Grades_NormalizedName` covers live rows only. There are no foreign keys.
- Coverage of 0-100 by the active bands is a rule across rows, so it is enforced by `GradeScaleService` under the advisory lock, not by a
  constraint.
- Seed ids are fixed version 5 UUIDs of `grade|LETTER` under the geography seeds' namespace (for example A+ is
  `0ebd496e-c1c2-572e-9973-00adb99eee78`), written out in `seed-grades.sql`.

