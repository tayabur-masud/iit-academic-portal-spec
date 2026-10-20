# Research: Configuration

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Research date**: 2026-10-13

## Findings and Decisions

### Application architecture

- **Finding**: The service is an ASP.NET Core 10 layered monolith: Domain, Application, Infrastructure, and API projects. The frontend is an Angular 21 SPA in a separate submodule.
- **Decision**: Add a `Configuration` feature slice to those existing projects and `src/app/features/configuration/` in Angular. Keep controllers thin and use explicit request/response DTOs.
- **Rationale**: This matches the existing architecture and the constitution without a new service, project, or framework.
- **Alternatives considered**: A separate configuration service was rejected because the feature is reference data within the existing portal and the constitution requires a layered monolith. A generic key/value catalog was rejected because it would weaken typed relationships and entity-specific constraints.

### Catalog persistence and validation

- **Finding**: `PortalDbContext` owns EF mappings and migrations. The repository has no current academic catalog entities or catalog normalization examples.
- **Decision**: Model each of the nine independent catalogs (the ninth, Hall, was added on 2026-10-14 with the same shape as the coded catalogs plus an address), Batch, Division, District, and Upazila as typed entities. Name the academic-session entity/table `AcademicSession`/`AcademicSessions` to distinguish it from authentication sessions. Persist normalized comparison fields and express uniqueness/parent relationships in PostgreSQL indexes and foreign keys. Use restrictive delete behavior; lifecycle changes use `IsActive`.
- **Normalization rule**: For names, including Batch names, trim, collapse repeated whitespace, and compare using invariant case normalization. For optional codes, treat whitespace-only as absent, trim, and compare invariantly without changing the display value. Do not accent-fold or invent cross-catalog uniqueness.
- **Rationale**: This implements the spec's whitespace/case duplicate cases at both the application and database boundaries while preserving entered display text.
- **Alternatives considered**: Application-only duplicate checks were rejected because concurrent writes can race. Cascading deletes/deactivation were rejected by the lifecycle requirements. Exact field length caps are not specified; choose and enforce bounded API/database values together during implementation, informed by the owner seed values, without adding a new business constraint in this plan.

### Degree credit requirement

- **Finding**: `Degree` currently has only the shared name/active fields and optional code. It is registered through the generic `CodedResource<Degree>` flow; the shared request/response models and Angular resource descriptor have no numeric-field support. No existing Degree credit property or credit seed was found.
- **Decision**: Add `CreditsToBeCompleted` to Degree as a required positive decimal with up to two fractional digits, persisted as `numeric(8,2)`. Add a Degree-specific resource validator so missing, zero, negative, or over-precision values return a field error; expose JSON as `creditsToBeCompleted` only for Degree requests/responses; add a numeric editor to the existing Degree form. Other catalogs continue rejecting this field.
- **Existing data prerequisite**: A NOT NULL migration on an existing Degrees table requires an owner-approved positive credit-total mapping for every pre-existing Degree row. Do not backfill guessed values or a sentinel zero. Supply a mapping before generating/applying the migration; if there are no existing Degree rows, the mapping is unnecessary.
- **Rationale**: A positive numeric total supports fractional credit values while preserving Degree as an independent reference record. A Degree-specific extension keeps the existing generic catalog behavior for unrelated resources.
- **Alternatives considered**: Storing credits as free text, allowing null/zero totals, or adding the field to every catalog request were rejected because they weaken validation or pollute unrelated resources.

### Authorization and audit

- **Finding**: `PortalPolicies.AdminModule` authorizes only the server-validated session's active Admin role. `IAuditEventRecorder.Stage` adds a business event to the current unit of work without saving; the caller saves the change and event together through `PortalDbContext`.
- **Decision**: Apply the existing Admin policy to all configuration mutations and re-check it on every request. Stage one allowlisted business audit event for each successful catalog, batch, geographic, or bulk-seed write, then persist it in the same transaction. Reuse the existing `AuditTransactionTests` pattern for rollback on audit failure.
- **Rationale**: This preserves the portal's authorization boundary and its atomic-audit guarantee.
- **Alternatives considered**: UI-only authorization, client-supplied roles, a second audit implementation, and security-outbox recording for business changes were rejected.

### API and validation contracts

- **Finding**: Existing APIs use resource-oriented routes, explicit DTOs, active-session policies, safe Problem Details, and field-keyed `ValidationProblem` responses. Audit has a versioned OpenAPI 3.1 contract.
- **Decision**: Define REST collection/item routes under `/api/admin/configuration`, resource-specific request/response shapes, active/inactive filters, and bounded batch retrieval where collection size warrants it. The Angular Admin lists use cursor-based infinite loading and never show numbered page controls or page-size selectors; the API may retain numbered-page parameters for non-UI clients. Use the existing OpenAPI generation in the service and keep the planning contract in the feature's `contracts/` directory.
- **Rationale**: This is consistent with existing clients and error handling; duplicate and invalid-reference errors remain actionable at the field level.
- **Decision** (2026-10-14): Every entity records created by, created time, last modified by, and last modified time through a shared `AuditedEntity` base class, mapped automatically and stamped by a `SaveChanges` interceptor from the server clock and the request's signed-in account, so no feature can forget it and no client can supply it. A convention test fails when a new entity neither derives from the base nor is listed as exempt with a reason; Identity tables, `AuthSession`, `AuditEvent`, and the security outbox are the current exemptions because they record the same facts another way. Alternatives rejected: setting the values in each use case (forgettable), database triggers (no access to the signed-in account), and relying on audit events alone (they answer what changed, not who and when last, without a join).
- **Decision** (revised 2026-10-14): Every delete is a soft delete, and only for unreferenced records. The row is kept and marked (`IsDeleted`, `DeletedAtUtc`, `DeletedByUserId`), a global query filter hides it, and the unique indexes become filtered indexes over live rows so names and codes can be reused. Because nothing is removed, the foreign keys no longer reject a delete, so the application checks for live dependents and returns a conflict with deactivation as the safe alternative. Restrictive foreign keys remain as the guard against any physical delete. Audit each successful delete in the same unit of work.
- **Superseded**: the original decision exposed physical delete for unreferenced records.
- **Alternatives considered**: Cascading deletion and disabling all physical deletion were rejected: the former violates integrity/history, while the latter does not satisfy the Admin's requested delete operation. Restoring a deleted record in the portal, and keeping deleted names reserved, were offered and declined. A client-supplied role or one untyped catch-all DTO was also rejected.

### Geographic source and import

- **Finding**: No owner-provided JSON file or reusable geographic loader exists in the repositories. The feature spec assigns the approved official source and version to the data owner.
- **Decision**: Provide active-Admin-authorized validate and import API operations accepting the documented JSON body. Validation produces a report without writes. Import validates the complete hierarchy and uniqueness rules before opening one transaction; it writes nothing on any validation error, records the source/version and counts in an allowlisted audit event, and does not overwrite pre-existing Admin-managed rows. Re-running against populated rows reports conflicts rather than silently updating them.
- **Rationale**: This keeps a large external seed out of schema migrations, prevents partial hierarchy loads, preserves Admin edits, and applies the same active-role boundary to bulk and individual configuration writes.
- **Alternatives considered**: An operator-only CLI that bypasses the active Admin authorization boundary, invented or manually transcribed geography, partial per-row imports, silent upserts, and embedding an unapproved dataset in an EF migration were rejected.
- **External prerequisite**: The owner must supply the approved JSON and source/version citation before seed validation can pass release acceptance. The importer format and behavior can be designed and tested with synthetic fixtures, but production source fidelity cannot be claimed until then.

### Frontend and testing

- **Finding**: Angular uses typed `HttpClient` services, reactive forms, accessible state announcements, and Vitest. Backend API tests use xUnit, `WebApplicationFactory`, SQLite for pipeline coverage, and dedicated PostgreSQL 18 tests where provider behavior matters.
- **Decision**: Reuse these patterns for catalog lists/forms, loading/empty/error/inactive states, and Admin navigation. Label the Admin menu entry “Configuration” and use `/admin/configuration`; hide it for other active roles while keeping route/API guards authoritative. Add PostgreSQL integration coverage for indexes, foreign keys, deletes, and migrations; do not treat SQLite as proof of PostgreSQL-specific behavior.
- **Rationale**: This tests behavior at the existing boundaries without adding dependencies.
- **Alternatives considered**: New UI/test frameworks and SQLite-only database verification were rejected.

## Technical Goals and Open Inputs

- The feature specification sets no numeric latency or throughput SLO. Use bounded paging and measure representative API interactions; do not invent a target.
- No design clarification remains open. The approved geographic JSON is an owner-supplied release input, not a choice for implementation to guess.

### City Corporation geography extension (2026-10-20)

- **Finding**: The current service implements Division, District, and Upazila as separate `ConfigurationRecord` entities/resources. District and Upazila use the shared configuration CRUD, parent filters, normalized names, active-parent validation, restrictive foreign keys, soft deletion, and audit behavior. The District delete check currently detects Upazila dependents only. Geography import validation, result counts, target-empty checks, and the seed schema currently cover only the three existing levels. The Angular descriptor and generic list/form already support District-parent filters and the Division-to-District cascade.
- **Decision**: Add City Corporation as a fourth, code-free geographic entity with one required District parent. Its normalized name is unique within City Corporations in that District, following the separate-catalog uniqueness convention; names need not be unique across Upazila and City Corporation types. Reuse the shared Admin configuration CRUD, lifecycle, audit, list, and form patterns. New or reassigned City Corporations require an active District; an unchanged inactive District reference remains readable, and deactivation preserves existing children while blocking new assignments. District deletion is blocked while either Upazilas or City Corporations remain.
- **Rationale**: The user specified a City Corporation as another geography configuration under a District, like Upazila. Reusing the existing typed resource architecture preserves consistent validation, authorization, soft-delete, audit, and parent selection without coupling it to student addresses or adding a service or package.
- **Import decision**: Extend the owner-provided seed schema with an optional `cityCorporations` array on a District; each entry contains a required name only. Treat an omitted array as no City Corporations for that District, validate duplicates within that District's City Corporation list, include the count in reports/audit metadata, and include the CityCorporations table in target-empty checks and atomic writes. Do not invent a SQL seed or claim production source fidelity; owner-approved City Corporation records and citation/version remain a release input.
- **API/UI decision**: Add `/api/admin/configuration/geography/city-corporations` with the existing District parent filter (`districtId`). Reuse the generic list/form and extend its resource descriptor with a required District selector filtered by the selected Division. No new request field, framework, or UI module is needed beyond the existing parent-reference support.
- **Database decision**: Add a `CityCorporations` table with shared configuration/audit columns, required `DistrictId`, restrictive FK, parent index, and partial unique index on `(DistrictId, NormalizedName)` for non-deleted records. The migration is additive and does not alter existing geographic rows.
- **Validation decisions**: Extend `GeographySeedValidator`, `GeographySeed`, import DTO/report counts, `GeographyImportService`, and geography routes/allowlist. API and import tests must cover parent validation, scoped duplicates, inactive Districts, District delete protection, all-or-nothing import, and City Corporation counts. Add PostgreSQL migration/constraint coverage and Angular list/form filtering tests.
- **Alternatives considered**: A separate City Corporation module would duplicate catalog behavior. Adding a code field or inserting invented City Corporation seed rows has no basis in the request and is rejected. Requiring the optional seed array for every District would reject owner files that contain no City Corporations, so it remains optional.

## Degree Minimum CGPA for Completion (2026-10-20)

- **Decision**: Add `MinimumCgpaForCompletion` to Degree as a required decimal on the 4.00 scale, greater than 0 and at most 4.00, with up to
  two fractional digits, persisted as `numeric(3,2)` with the check `CK_Degrees_MinimumCgpaForCompletion_Range`. It is exposed as
  `minimumCgpaForCompletion` on Degree requests and responses only and is refused on other catalogs.
- **Existing rows**: Unlike the credit total, the project owner chose a backfill: every Degree that exists when the migration runs gets 2.50.
  The migration then drops the column default, so a new Degree must always state its own value. Admins can change the backfilled value.
- **Eligibility**: Not used by graduation eligibility yet (owner decision); the portal does not calculate a student's CGPA. Feature 007's
  check still compares credits only.
- **One migration**: At the owner's request it ships in the same migration as the City Corporation table,
  `CityCorporationsDegreeMinimumCgpaAndGrades`.
- **Alternatives considered**: An optional value (a missing minimum would have to mean "no CGPA rule"), and a 5.00 scale; both were declined.

## Grade Configuration (2026-10-20)

- **Finding**: No Grade catalog, domain entity, persistence mapping, API route, or Angular editor exists. The generic configuration flow supports individual records and per-record audit, but the approved Grade requirements also require the active set to remain one complete, non-overlapping 0–100 partition.
- **Decision**: Add a global Grade scale shared by all Degrees. Use one record per band with the letter grade as the normalized unique catalog name, a lower mark bound (inclusive), upper mark bound, an upper-bound-included flag, a numeric grade (`numeric(3,2)` within 0.00–4.00), comment, and the shared active/audit/soft-delete fields. Percentage bounds are decimal values from 0 to 100; do not round marks during range matching.
- **Initial values**: Seed the ten bands and comments from the screenshot with the seed script `database/postgresql/seed-grades.sql`, like the geography data (owner decision; the Grade table itself is in the shared `CityCorporationsDegreeMinimumCgpaAndGrades` migration, which holds no seed). Lower bounds are inclusive; upper bounds are exclusive except the A+ band, whose 100 upper bound is inclusive. Because the active bands must cover 0-100 exactly once, the script seeds only an empty table and never adds beside or overwrites later Admin changes.
- **Scale maintenance**: Provide list/detail reads and a full-scale replacement workflow. The editor submits the desired complete set, including active/inactive bands; omitted, unreferenced bands are soft-deleted. A PostgreSQL transaction-scoped advisory lock serializes replacement writes. In the locked transaction, validate the resulting active set covers every value in 0–100 exactly once, then save all row changes and audit events atomically. Reject invalid replacement attempts without partial writes; valid concurrent replacements serialize and each committed scale satisfies the invariant.
- **Rationale**: The continuous coverage requirement is cross-record: editing, deactivating, or deleting one band independently can leave a gap or overlap. Validating and replacing the whole set atomically supports the Admin-approved maintenance model while preserving the range invariant.
- **Alternatives considered**: Reusing per-row generic CRUD was rejected because a single row mutation cannot safely reshape neighboring thresholds while preserving full coverage. Degree-specific Grade copies were rejected by the clarification that one rubric is shared across Degrees. Assigning grades to students, converting marks, calculating results, or calculating CGPA remains out of scope.
- **Validation**: Add domain/service tests for all ten defaults, inclusive/exclusive boundaries, percentages 0 and 100, values outside the range, numeric-grade precision/range, duplicate letters, gaps/overlaps, and atomic rejection. Add PostgreSQL coverage for migration seeding and concurrent replacement serialization, plus Angular tests for whole-scale editing and validation feedback.

