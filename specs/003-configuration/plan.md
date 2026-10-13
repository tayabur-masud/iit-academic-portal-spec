# Implementation Plan: Configuration

**Branch**: `feature/003-configuration` | **Date**: 2026-10-13 | **Spec**: [spec.md](spec.md)

**Input**: [Configuration feature specification](spec.md)

## Summary

Deliver twelve independently modeled reference entities for Admin maintenance: eight academic catalogs, Batches, and the Division/District/Upazila hierarchy. Academic Sessions have globally unique normalized labels, every Batch requires one Session reference, and Batch names are unique within their Degree. Keep the implementation in the existing Angular + ASP.NET Core layered monolith. Enforce normalized uniqueness, parent/reference integrity, active-role authorization, non-cascading deactivation, and same-unit-of-work business audit. Load the owner-approved geographic JSON through a controlled all-or-nothing importer; do not invent seed data or introduce new runtime dependencies.

## Technical Context

**Language/Version**: TypeScript 5.9 / Angular 21; C# 14 / .NET 10.

**Primary Dependencies**: Existing Angular HttpClient, Router, and reactive forms; Tailwind CSS 4; ASP.NET Core 10, EF Core 10, and Npgsql 10.x. No new package is planned.

**Storage**: PostgreSQL 18 through the existing `PortalDbContext`; schema changes require a versioned EF Core migration.

**Testing**: xUnit API/integration tests using the real ASP.NET pipeline and SQLite fixture, plus PostgreSQL 18 integration coverage for provider-specific constraints/migrations; Angular Vitest component/service tests with `HttpTestingController`.

**Target Platform**: Browser-based Angular SPA and ASP.NET Core 10 API; local development uses Node.js 24 LTS, .NET 10 SDK, and PostgreSQL 18.

**Project Type**: Layered monolith with separate Angular frontend and ASP.NET Core REST API submodules.

**Performance Goals**: The specification sets no numeric latency or throughput target. Keep small catalogs searchable and return bounded pages where volume warrants it; avoid unbounded geographic or batch results. Establish numeric SLOs only through an approved requirement.

**Constraints**: Only the session's active Admin role authorizes writes. Use existing REST, Problem Details, anti-forgery, audit, and design-system conventions. Preserve references when records are deactivated; use restrictive foreign keys. Geographic data must come from an owner-supplied JSON file citing the approved source/version; validate the complete hierarchy before committing any seed rows. No student/teacher workflows, new role, service, framework, or audit system.

**Scale/Scope**: Eight independent catalogs (institution, board, group, examination type, degree, semester, academic session, designation), one batch resource, and three geographic levels. Geographic row counts are determined by the approved owner dataset and are not assumed here.

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

1. Add separate domain entities and EF mappings for catalogs, batches, and geography; persist normalized comparison values; add unique indexes and restrictive relationships in one reviewed migration. Name the academic catalog entity/table `AcademicSession`/`AcademicSessions` to distinguish it from authentication `AuthSession`/`AuthSessions`.
2. Add application use cases for normalization, validation, search/filter/page, lifecycle changes, and relationship checks. Stage a feature-owned business audit event through `IAuditEventRecorder` and save it with each successful write in the same `PortalDbContext` unit of work.
3. Add active-Admin-only REST resources, request/response DTOs, field-keyed validation errors, and Admin-authorized geography validate/import endpoints. The importer validates source/version, normalized names, and all parent links before one transaction; invalid input writes nothing and produces a validation report.
4. Add an Angular configuration feature under the existing feature tree at `/admin/configuration`; label its Admin module menu entry “Configuration” and guard the route by the active Admin role. Include typed API services, reusable catalog list/form patterns, geographic parent selectors, and accessible request states.
5. Verify API authorization and audit behavior through the real test host, database constraints against PostgreSQL, Angular interactions through Vitest, and the manual end-to-end scenarios in [quickstart.md](quickstart.md).

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
├── src/IitAcademicPortal.Infrastructure/Persistence/Migrations/
├── src/IitAcademicPortal.Api/Controllers/Configuration/
├── src/IitAcademicPortal.Api/Contracts/
└── tests/IitAcademicPortal.Api.Tests/Configuration/

iit-academic-portal/src/app/features/configuration/
├── configuration-api.ts       (typed client for every resource)
├── configuration.models.ts    (item types and the descriptor of each resource)
├── configuration-home.*       (landing page grouping the resources)
└── catalogs/                  (the generic list and form that serve all twelve resources)
```

**Structure Decision**: Extend the existing Domain → Application → Infrastructure → API split and Angular feature layout. Keep EF mappings in the current `PortalDbContext` convention, use the existing migrations folder and xUnit/Vitest projects, and do not add a project or package solely for this feature.
