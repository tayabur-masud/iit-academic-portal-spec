# Implementation Plan: Student Management

**Branch**: `feature/007-student-management` | **Date**: 2026-10-19 | **Spec**: [spec.md](spec.md)

**Input**: [Student Management specification](spec.md)

## Summary

Deliver one Admin-managed Student profile and a separate admission for each degree/batch enrollment. The first admission creates the linked Student account; later admissions for the same person reuse that account and profile. Store prior academic-history entries on the profile so they are shared across program admissions. Support later Hall and registration-number assignment, and calculate eligibility separately for each admission from its Degree's configured requirement and its passed-course credits. Reuse the existing pending-account credential setup only for newly created accounts; do not store scanned documents, course registrations/results, duplicate credit totals, or automatic cross-program credit transfers.

## Technical Context

**Language/Version**: TypeScript 5.9 / Angular 21; C# 14 / .NET 10.

**Primary Dependencies**: Existing Angular HttpClient, Router, reactive forms, ASP.NET Core, EF Core, ASP.NET Core Identity, notification, audit, and configuration services. No new package is planned.

**Storage**: PostgreSQL 18 through the existing `PortalDbContext`; add Student profile, program admission, and academic-history tables, foreign keys, and uniqueness constraints through versioned EF migrations.

**Testing**: xUnit API/integration tests using the existing test host and SQLite fixture, PostgreSQL 18 tests for provider-specific constraints and migrations, and Angular Vitest tests using the existing HttpClient test support.

**Target Platform**: Browser-based Angular SPA and ASP.NET Core 10 API; local development uses Node.js 24 LTS, .NET 10 SDK, and PostgreSQL 18.

**Project Type**: Layered monolith with separate Angular frontend and ASP.NET Core REST API repositories.

**Performance Goals**: The specification sets no numeric API latency or throughput target. Student lists must use bounded pagination and searchable/filterable queries consistent with existing Admin directories.

**Constraints**: Active Admin authorization is enforced server-side. Initial account/profile/admission creation plus audit must commit atomically; request credential setup only after creating a new account and committing. Reuse a normalized-email match only when it identifies a Student account; never merge a non-Student account. One profile has multiple program admissions; same Student/Degree/Batch duplicates are rejected. Email remains canonical on the portal account and shared mobile/personal data belongs to the profile. Mailing and permanent addresses each use free-text details plus an Upazila reference; derive District/Division through existing geographic Configuration and reject invalid references. Academic-history Category uses six fixed values; all other applicable source-form values are optional text snapshots, not mutable Configuration references. Do not store attachments, credentials, course registrations/results, completed-credit copies, or automatic credit transfers between admissions. Degree, Batch, and Hall relationships use existing Configuration records. Course-result credits are read through an admission-keyed application boundary; unavailable source data must never produce eligibility.

**Scale/Scope**: One Student profile per portal account, zero or more profile-level academic-history entries from six categories, and zero or more program admissions per profile. Each admission has optional later Hall and registration number and its own computed graduation eligibility. No record-volume estimate is specified; use bounded student/admission paging.

## Constitution Check

*GATE: PASS before Phase 0 research and rechecked after Phase 1 design.*

| Principle | Result | Evidence |
|---|---|---|
| IIT identity and academic usability | PASS | Reuse shared IIT UI tokens and components; provide accessible form, validation, loading, empty, success, and error states. |
| Maintainable layered architecture | PASS | Put admission and eligibility rules in the domain/application layers; keep controllers and Angular components focused on transport and presentation. |
| RESTful API contracts | PASS | Expose resource-oriented student routes with explicit DTOs, field errors, and documented account-creation semantics. |
| Security and data integrity | PASS | Enforce active-Admin access on each request, protect personal data, and constrain account, Degree, Batch, Hall, and registration-number relationships in the database. |
| Reliable, simple, observable software | PASS | Test atomic account/admission creation, setup failure behavior, authorization, registration uniqueness, credit-source unavailability, and eligibility outcomes. |

No constitution exception, new framework, or dependency is required.

## Project Structure

### Documentation (this feature)

```text
specs/007-student-management/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── student-management-api.md
└── tasks.md             # Created by /speckit-tasks
```

### Source Code (repository root)
```text
iit-academic-portal-service/
├── src/IitAcademicPortal.Domain/Students/
├── src/IitAcademicPortal.Application/Students/
├── src/IitAcademicPortal.Infrastructure/Persistence/
├── src/IitAcademicPortal.Api/Controllers/
├── src/IitAcademicPortal.Api/Contracts/
└── tests/IitAcademicPortal.Api.Tests/Students/

iit-academic-portal/src/app/features/students/
├── student-api.ts
├── student.models.ts
├── student-profile.*
├── student-list.*
├── student-form.*
└── graduation-eligibility.*
```

**Structure Decision**: Extend the existing Domain → Application → Infrastructure → API split and Angular feature tree. Keep persistence in `PortalDbContext`, migrations in the existing migrations folder, and tests in the current xUnit and Vitest projects. Reuse existing account, audit, notification, and Configuration abstractions; add no project or package.
