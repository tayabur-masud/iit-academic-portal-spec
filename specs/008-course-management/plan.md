# Implementation Plan: Course Management

**Branch**: `feature/008-course-management` | **Date**: 2026-10-19 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/008-course-management/spec.md`

## Summary

Add Admin-managed Course records to the existing configuration catalog. Reuse the current resource registry, list/read/create/update pipeline, audit transaction, Admin policy, problem responses, and Angular configuration screens, extending their contracts and descriptors for course-specific credits, type, Degree, and optional prerequisite. Course-specific application validation will enforce global code uniqueness, each supplied theory/lab value being between zero and total credit hours inclusive, matching sums when both components are supplied, same-Degree active prerequisites, acyclic prerequisite chains, rejection of Degree changes that invalidate any affected prerequisite without modifying affected Courses, and protection against deactivating a prerequisite used by an active course. The schema change is versioned as an EF Core migration.

## Technical Context

**Language/Version**: TypeScript 5.9 / Angular 21 / Node.js 24; C# 14 / ASP.NET Core 10

**Primary Dependencies**: Existing Angular forms/router and design system; ASP.NET Core Identity 10, EF Core 10, Npgsql, and PostgreSQL 18. No new package is planned.

**Storage**: PostgreSQL 18 via the existing `PortalDbContext` and configuration persistence layer

**Testing**: Angular `npm test` and `npm run build`; service `dotnet test` (API pipeline tests use SQLite, with optional PostgreSQL-specific integration coverage)

**Target Platform**: Admin web application and ASP.NET Core API

**Project Type**: Layered monolith with Angular frontend and REST API

**Performance Goals**: Use existing catalog search and cursor pagination (default 25, maximum 100); the feature defines no additional latency or throughput SLO.

**Constraints**: Active Admin authorization on every request; transactional business audit; positive total credit hours at up to two decimal places; globally unique course code; theory/lab values independently optional, each between zero and total inclusive, and summing to total when both are set; prerequisite must be active, same-Degree, non-self, and acyclic; reject a Degree change that invalidates any affected prerequisite without modifying affected Courses; reject deactivation while an active course depends on the course. Do not change enrollment, teaching, or result data. Use existing platform stack and conventions.

**Scale/Scope**: One Course catalog resource with list/search/read/create/update/deactivate flows, Degree and optional self-prerequisite relationships, audit events, and one additive schema migration. No numeric catalog-size target is specified.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **I. IIT Visual Identity and Academic Usability: PASS**. Extend the existing responsive configuration list/form and reuse design-system controls, validation, loading, empty, success, and error states. Keep deactivation explicit and auditable.
- **II. Maintainable Layered Architecture: PASS**. Keep Course rules in the application/domain boundary; keep the controller thin; persistence remains behind the existing configuration store and EF Core context.
- **III. RESTful API Contracts: PASS**. Add a resource under the existing `/api/admin/configuration` route, use the shared list/read/create/update semantics, validate at the API boundary, and preserve field-keyed errors and paging conventions.
- **IV. Security and Data Integrity: PASS**. Require the active Admin role and anti-forgery token for state changes; validate all course and prerequisite rules server-side; add restrictive FKs, indexes, checks, and a versioned migration; preserve audit atomicity.
- **V. Reliable, Simple, and Observable Software: PASS**. Cover business rules, authorization, audit rollback, and PostgreSQL constraints with focused automated tests; retain existing logging and error handling without a new dependency.

**Pre-research gate: PASS.** No constitution violation or unresolved spec clarification blocks design. Feature-specific latency, availability, and volume targets were not stated; existing paged catalog behavior is the chosen baseline.

## Project Structure

### Documentation (this feature)

```text
specs/008-course-management/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
iit-academic-portal/
└── src/app/features/configuration/
    ├── configuration.models.ts
    ├── configuration-api.ts
    └── catalogs/ (list, form, and focused tests)
iit-academic-portal-service/
├── src/IitAcademicPortal.Domain/Configuration/Course.cs
├── src/IitAcademicPortal.Application/Configuration/ (Course resource and registry)
├── src/IitAcademicPortal.Api/ (shared configuration contracts and catalog route)
├── src/IitAcademicPortal.Infrastructure/Persistence/ (Course mapping and migration)
└── tests/IitAcademicPortal.Api.Tests/Configuration/ (validation, API, auth, audit, and persistence tests)
```

**Structure Decision**: Keep Course in the existing configuration-catalog vertical slice rather than creating parallel frontend and backend modules. Extend shared catalog DTOs/descriptors only for optional Course fields; add a focused Course resource for domain validation. The existing generic delete operation is not part of the approved Course scope: expose only list, read, create, and update, with deactivation performed by updating `isActive` to `false`.

## Post-Design Constitution Check

- **I. IIT Visual Identity and Academic Usability: PASS**. The design reuses existing Admin catalog screens and design-system states, adds accessible optional-reference and credit fields, and keeps deactivation explicit.
- **II. Maintainable Layered Architecture: PASS**. Course-specific rules stay in the application resource/store boundary; the API remains thin; EF details remain in Infrastructure.
- **III. RESTful API Contracts: PASS**. The contract uses the existing resource path, verbs, paging, anti-forgery, and Problem Details conventions. A per-resource opt-out prevents exposing an unrequested delete operation.
- **IV. Security and Data Integrity: PASS**. Active Admin authorization, audit atomicity, restrictive FKs, code uniqueness, credit checks, prerequisite validation, and a versioned migration are included.
- **V. Reliable, Simple, and Observable Software: PASS**. Validation, authorization, cycle/dependent behavior, audit rollback, and PostgreSQL constraints receive focused automated coverage; no new dependency is introduced.

**Final design gate: PASS.** No constitution violations or unresolved technical decisions remain. The schema change is limited to Course and does not alter enrollment or result records.

## Complexity Tracking

No constitution violations require an exception. The Course-specific validation and resource-level name-uniqueness opt-out reuse the shared catalog without introducing rules absent from the specification.
