# Implementation Plan: Configuration

**Branch**: `feature/003-configuration` | **Date**: 2026-10-20 | **Spec**: [spec.md](spec.md)

**Input**: [Configuration feature specification](spec.md)

## Summary

Deliver the existing Admin configuration catalogs plus City Corporation as a fourth geographic entity beside Division, District, and Upazila, and add a global Admin-maintained Grade scale shared by all Degrees. A City Corporation has a required District parent, no code, and a normalized name unique within its District. Extend the existing geography import contract to accept optional City Corporation lists within Districts and include their counts and validation in the existing all-or-nothing import. Grade stores marks bands, letter grades, numeric grades, and comments; seed the ten screenshot bands and preserve a continuous, non-overlapping active 0–100 scale through atomic scale replacement. Academic Sessions retain globally unique normalized labels; every Batch requires one Session, and Batch names are unique within their Degree. Each Degree stores a required positive `CreditsToBeCompleted` total and a required `MinimumCgpaForCompletion` (`numeric(3,2)`, greater than 0 and at most 4.00); Configuration does not calculate earned credits, CGPA, or graduation eligibility. The City Corporation table and Degree CGPA column ship in `CityCorporationsDegreeMinimumCgpaAndGrades`; Grade uses a separate versioned migration. Reuse the existing Angular + ASP.NET Core layered monolith, authorization, audit, and lifecycle conventions. Do not invent City Corporation seed data or add runtime dependencies.

## Technical Context

**Language/Version**: TypeScript 5.9 / Angular 21; C# 14 / .NET 10.

**Primary Dependencies**: Existing Angular HttpClient, Router, and reactive forms; Tailwind CSS 4; ASP.NET Core 10, EF Core 10, and Npgsql 10.x. No new package is planned.

**Storage**: PostgreSQL 18 through the existing `PortalDbContext`. One migration, `CityCorporationsDegreeMinimumCgpaAndGrades`, carries the City Corporation table, the Degree minimum CGPA, and (at the project owner's request, instead of a separate Grade migration) the `Grades` table with its scale-range checks, unique live letter-grade index, and the ten initial bands from the screenshot; persist percentage thresholds without imposing a new decimal scale, and persist numeric grades as `numeric(3,2)`.

**Testing**: xUnit API/integration tests using the real ASP.NET pipeline and SQLite fixture, plus PostgreSQL 18 integration coverage for the City Corporation and Grade migrations, restrictive District foreign key, parent-scoped unique indexes, and concurrent Grade-scale replacement; Angular Vitest component/service tests with `HttpTestingController` for District-filtered City Corporation and the Grade scale editor.

**Target Platform**: Browser-based Angular SPA and ASP.NET Core 10 API; local development uses Node.js 24 LTS, .NET 10 SDK, and PostgreSQL 18.

**Project Type**: Layered monolith with separate Angular frontend and ASP.NET Core REST API submodules.

**Performance Goals**: The specification sets no numeric latency or throughput target. Keep small catalogs searchable and return bounded pages where volume warrants it; avoid unbounded geographic or batch results. Establish numeric SLOs only through an approved requirement.

**Constraints**: Only the session's active Admin role authorizes writes. Reuse existing REST, Problem Details, anti-forgery, audit, and design-system conventions. Preserve references when Division or District is deactivated; block new assignments to inactive parents; use restrictive foreign keys. City Corporation names are normalized and unique within the parent District; geographic records have no codes. The owner-approved geographic JSON may include `cityCorporations` for a District; do not invent or require seed rows without an approved source. Grade is one global scale, with active bands continuously and non-overlappingly covering 0–100. All Grade-set changes must validate and save atomically so concurrent Admin writes cannot create a gap or overlap. Preserve the existing Degree credit/CGPA rules and keep student/teacher workflows, new roles, services, frameworks, and audit systems out of scope.

**Scale/Scope**: Fifteen configuration resources: existing academic catalogs, Batch, four geographic entity types (Division, District, Upazila, City Corporation), and one global Grade scale. City Corporation volume is determined by a future owner-approved dataset and is not assumed here; the Grade scale begins with ten bands.

## Constitution Check

*Gate before Phase 0: PASS. Re-checked after Phase 1 design: PASS.*

| Principle | Result | Evidence |
|---|---|---|
| IIT identity and usability | PASS | Reuse the shared design system and IIT tokens; provide responsive loading, empty, validation, and error states. |
| Layered architecture | PASS | Place domain rules, use cases, persistence, and thin API endpoints in their existing projects. |
| REST contracts | PASS | Define resource endpoints and DTOs in the feature OpenAPI contract; use HTTP semantics and safe Problem Details. |
| Security and integrity | PASS | Enforce `AdminModule` from the active session on every write; constrain uniqueness and relationships in PostgreSQL; make required audit atomic. |
| Reliability and observability | PASS | Test CRUD, invalid input, role denial, PostgreSQL constraints, audit rollback, and importer all-or-nothing behavior. |

No constitution exception or new dependency is required. The owner-provided geographic seed file is a release prerequisite, not a design ambiguity.

## Implementation Approach

1. Add separate domain entities and EF mappings for catalogs, batches, and geography; persist normalized comparison values; add unique indexes and restrictive relationships in one reviewed migration. Extend `Degree` with the required positive credit total and backfill existing Degree rows only from an owner-approved mapping before enforcing the required database value. Name the academic catalog entity/table `AcademicSession`/`AcademicSessions` to distinguish it from authentication `AuthSession`/`AuthSessions`.
2. Add application use cases for normalization, validation, search/filter/page, lifecycle changes, and relationship checks. Add Degree-specific validation for a required positive credit total with at most two fractional digits. Stage a feature-owned business audit event through `IAuditEventRecorder` and save it with each successful write in the same `PortalDbContext` unit of work.
3. Add active-Admin-only REST resources, request/response DTOs, field-keyed validation errors, and Admin-authorized geography validate/import endpoints. Degree DTOs expose `creditsToBeCompleted` and `minimumCgpaForCompletion` only for Degree records and reject them for other resources. The importer validates source/version, normalized names, and all parent links before one transaction; invalid input writes nothing and produces a validation report.
4. Add an Angular configuration feature under the existing feature tree at `/admin/configuration`; label its Admin module menu entry “Configuration” and guard the route by the active Admin role. Include typed API services, reusable catalog list/form patterns, a Degree-only required credits input, geographic parent selectors, and accessible request states.
5. Verify API authorization and audit behavior through the real test host, database constraints against PostgreSQL, Angular interactions through Vitest, and the manual end-to-end scenarios in [quickstart.md](quickstart.md).
6. Extend the geography slice with a `CityCorporation` entity/resource, `city-corporations` route, District-scoped filter and uniqueness, and a District foreign key. Update District dependent checks so City Corporations block parent deletion, and validate active District references on create or reassignment while preserving unchanged inactive historical references.
7. Extend the nested geographic seed schema and import pipeline with optional `cityCorporations` arrays under Districts. Validate names and counts, include CityCorporations in the empty-target check and atomic insert/audit, but add no fabricated SQL seed; production seed acceptance requires owner-provided source/version data.
8. Add the City Corporation descriptor and Administration UX to the existing Geography group. Reuse the Division-to-District cascade, then use the selected District for City Corporation filtering and parent selection; keep current Admin guards and accessibility states.
9. Add `Grade.cs` in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Configuration/`, a dedicated `GradeScaleService.cs` in the Application layer, and a `GradeScaleController.cs` under the API Configuration controllers. Use a full-scale replacement request rather than independent row writes: acquire a PostgreSQL transaction-scoped advisory lock, validate the complete proposed active range set for gap-free, non-overlapping 0–100 coverage, then write creates/updates/deactivations/soft deletes and allowlisted audit events atomically. Add a custom Grade scale editor under `iit-academic-portal/src/app/features/configuration/` instead of forcing cross-band validation into the generic one-record form. Seed the ten screenshot bands in the shared `CityCorporationsDegreeMinimumCgpaAndGrades` migration (owner decision; originally a separate migration); preserve Grade rows after Admin edits by keeping the migration seed insert-only.

## Project Structure

### Documentation (this feature)

```text
specs/003-configuration/
├── plan.md
├── research.md
├── data-model.md
├── contracts/
│   ├── configuration-api.md
│   └── geographic-seed.schema.json
└── quickstart.md
```

`tasks.md` is a later Phase 2 artifact and is not created by this planning workflow.

### Source Code (existing projects)

```text
iit-academic-portal-service/
├── src/IitAcademicPortal.Domain/Configuration/
├── src/IitAcademicPortal.Application/Configuration/
├── src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs
├── src/IitAcademicPortal.Infrastructure/Persistence/ConfigurationStore.cs, GradeScaleStore.cs
├── src/IitAcademicPortal.Infrastructure/Persistence/Migrations/
├── src/IitAcademicPortal.Api/Controllers/Configuration/
├── src/IitAcademicPortal.Api/Contracts/
└── tests/IitAcademicPortal.Api.Tests/Configuration/

iit-academic-portal/src/app/features/configuration/
├── configuration-api.ts       (typed client for every resource)
├── configuration.models.ts    (item types and the descriptor of each resource)
├── configuration-home.*       (landing page grouping the resources)
├── catalogs/                  (the generic list and form that serve the fifteen record-by-record resources)
└── grades/                    (the Grade scale editor: the whole scale in one table, saved together)
```

**Structure Decision**: Extend the existing Domain → Application → Infrastructure → API split and Angular feature layout. Keep EF mappings in the current `PortalDbContext` convention, use the existing migrations folder and xUnit/Vitest projects, and do not add a project or package solely for this feature.
