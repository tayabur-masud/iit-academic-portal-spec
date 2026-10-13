# Data Model: Configuration

**Feature**: [spec.md](spec.md)  
**Research decisions**: [research.md](research.md)

## Shared Conventions

- Use server-generated `Guid` identifiers, following existing portal records.
- Persist `IsActive` on every catalog, Batch, and geographic entity; new records default to active.
- Persist `NormalizedName` alongside the display `Name`. Normalize by trimming, collapsing repeated whitespace, and invariant case normalization. Preserve the display name as entered after trimming.
- Where a code is optional, store `NormalizedCode` as `NULL` when omitted or whitespace-only; otherwise trim and invariant-case-normalize it while preserving the display `Code`.
- Division, District, and Upazila have no code fields; geographic uniqueness is based on normalized names and parent scope.
- Do not expose normalized comparison fields as client-editable DTO fields.
- Exact maximum string lengths are not specified by the feature. Set consistent API/domain and database limits during implementation, based on approved seed values where applicable.

## Entities

| Entity | Fields | Relationships and constraints |
|---|---|---|
| `EducationalInstitution` | `Id`, `Name`, `NormalizedName`, `Type` (`School`, `College`, `University`), nullable `Code`/`NormalizedCode`, nullable `Address`, `IsActive` | Independent catalog. Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type. |
| `EducationBoard` | `Id`, `Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, `IsActive` | Independent catalog. Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type. |
| `EducationGroup` | `Id`, `Name`, `NormalizedName`, `IsActive` | Independent catalog. Unique `NormalizedName`. |
| `ExaminationType` | `Id`, `Name`, `NormalizedName`, `IsActive` | Independent previous-education category. Unique `NormalizedName`; not an IIT degree or student-history record. |
| `Degree` | `Id`, `Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, `IsActive` | Independent IIT program catalog. Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type. |
| `Semester` | `Id`, `Name` (display label), `NormalizedName`, `IsActive` | Independent catalog. Unique `NormalizedName`. No order field is needed by an existing domain rule; sort consistently by normalized name unless an approved requirement establishes ordering. |
| `AcademicSession` | `Id`, `Name` (display label), `NormalizedName`, `IsActive` | Independent academic-session catalog, distinct from authentication `AuthSession`. Unique `NormalizedName` globally. |
| `Designation` | `Id`, `Name`, `NormalizedName`, nullable `Code`/`NormalizedCode`, `IsActive` | Independent catalog. Unique `NormalizedName`; unique non-null `NormalizedCode` within this entity type. |
| `Batch` | `Id`, `Name`, `NormalizedName`, `DegreeId`, `StartingSemesterId`, required `AcademicSessionId`, `IsActive` | Exactly one `Degree`, one starting `Semester`, and one `AcademicSession`. Unique (`DegreeId`, `NormalizedName`); a different Degree may reuse the name. All references use restrictive delete behavior. |
| `Division` | `Id`, `Name`, `NormalizedName`, `IsActive` | Top geographic level. Unique `NormalizedName` globally. |
| `District` | `Id`, `DivisionId`, `Name`, `NormalizedName`, `IsActive` | Exactly one Division. Unique (`DivisionId`, `NormalizedName`). Restrict physical parent deletion. |
| `Upazila` | `Id`, `DistrictId`, `Name`, `NormalizedName`, `IsActive` | Exactly one District. Unique (`DistrictId`, `NormalizedName`). Restrict physical parent deletion. |

## Validation and Lifecycle

- Normalized names are unique inside their own catalog. Equal names in different catalogs are allowed.
- Optional normalized codes are unique inside their own academic catalog when non-null. Geographic entities have no code fields.
- Division normalized names are unique globally; District and Upazila normalized names are unique within their parent.
- Academic Session normalized names are unique globally. Multiple Batches may reference one Session.
- A Batch normalized name is unique only within its Degree; its required Session does not alter that scope.
- A District may be created or reassigned only under an existing active Division. An Upazila may be created or reassigned only under an existing active District. Batch writes require active Degrees, Semesters, and Academic Sessions.
- Deactivation preserves identifiers and existing relationships. Existing-reference views may show inactive status; new assignments to inactive records are rejected. Deactivation never cascades to children or dependents.
- Foreign keys use restrictive/no-action delete behavior. Admin may physically delete a record only when no dependents reference it; otherwise the delete is rejected and the record can be deactivated instead.
- Successful configuration creates, updates, deactivations, deletions, and their allowlisted business audit events are persisted in one database transaction. A failed audit persistence must roll back the configuration write.

## Geographic Seed Input

The seed document contains a source citation/version and nested Divisions → Districts → Upazilas. Its structural contract is [geographic-seed.schema.json](contracts/geographic-seed.schema.json). Semantic validation additionally enforces normalized uniqueness, parent links, and the approved source/version. The full document is validated before a single import transaction; no partial records are committed.
