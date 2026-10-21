# Implementation Plan: Student Management

**Branch**: `feature/007-student-management` | **Date**: 2026-10-19 | **Spec**: [spec.md](spec.md)

**Input**: [Student Management specification](spec.md)

## Summary

Deliver one Admin-managed Student profile and a separate admission for each Degree/Batch enrollment. The first admission creates the linked Student account; later admissions for the same person reuse that account and profile. Store prior academic-history entries on the profile so they are shared across admissions. Support later Hall and registration-number assignment, and calculate eligibility per admission from the Degree's required credits and minimum CGPA, that admission's passed-course credits and current cumulative CGPA from published results, and its Hall and registration number. Reuse the existing pending-account credential setup only for newly created accounts. Address locality may be one Upazila or City Corporation under the selected District. Do not store scanned documents, course registrations/results, copied result totals, or automatic cross-admission credit/CGPA transfers.

## Technical Context

**Language/Version**: TypeScript 5.9 / Angular 21; C# 14 / .NET 10.

**Primary Dependencies**: Existing Angular HttpClient, Router, reactive forms, ASP.NET Core, EF Core, ASP.NET Core Identity, notification, audit, and configuration services. No new package is planned.

**Storage**: PostgreSQL 18 through the existing `PortalDbContext`; preserve the existing Student Management schema and add a versioned migration for nullable mailing/permanent `CityCorporationId` references, restrictive foreign keys, indexes, and check constraints preventing both locality foreign keys from being set together. Application validation requires exactly one locality when an address is supplied. Existing Upazila references remain unchanged; no address backfill is required. CGPA remains read-only result-provider data and is not persisted by Student Management.

**Testing**: xUnit API/integration tests using the existing test host and SQLite fixture, PostgreSQL 18 tests for migration, restrictive geographic foreign keys and address-locality constraints, and Angular Vitest tests using the existing HttpClient test support. Cover CGPA below/equal/above the Degree minimum, unavailable Degree/result values, current cumulative CGPA per admission, no cross-admission carryover, both locality choices filtered by District, invalid locality references, and result-publication changes to eligibility.

**Target Platform**: Browser-based Angular SPA and ASP.NET Core 10 API; local development uses Node.js 24 LTS, .NET 10 SDK, and PostgreSQL 18.

**Project Type**: Layered monolith with separate Angular frontend and ASP.NET Core REST API repositories.

**Performance Goals**: The specification sets no numeric API latency or throughput target. Student lists must use bounded pagination and searchable/filterable queries consistent with existing Admin directories.

**Constraints**: Active Admin authorization is enforced server-side. Initial account/profile/admission creation plus audit must commit atomically; request credential setup only after creating a new account and committing. Reuse a normalized-email match only when it identifies a Student account; never merge a non-Student account. One profile has multiple program admissions; same Student/Degree/Batch duplicates are rejected. Email remains canonical on the portal account and shared mobile/personal data belongs to the profile. Mailing and permanent addresses each use free-text details plus exactly one Upazila or City Corporation reference; both locality lists are scoped to the selected District, and District/Division are derived from the selected locality. Academic-history Category uses six fixed values; school/college, university, institution, board, and group are chosen from the Configuration catalogs when listed, storing the record reference plus a name snapshot that later catalog changes never rewrite; a value not in a catalog is typed and kept as text; all other applicable source-form values are optional text snapshots. Do not store attachments, credentials, course registrations/results, copied credits/CGPA, or automatic cross-admission transfers. Degree, Batch, Hall, Upazila, and City Corporation relationships use existing Configuration records. Read current cumulative CGPA and passed-course credits through a read-only published-results boundary keyed by admission; missing source data must never produce eligibility. Compare the published admission CGPA against the selected Degree's minimum CGPA, using the configured Grade numeric-value system as owned by academic results.

**Scale/Scope**: One Student profile per portal account, zero or more profile-level academic-history entries from six categories, and zero or more program admissions per profile. Each admission has optional later Hall and registration number and its own computed graduation eligibility using admission-specific published results. Each of two addresses may reference exactly one Upazila or City Corporation. No record-volume estimate is specified; use bounded student/admission paging and District-filtered geographic lookups.

## Constitution Check

*GATE: PASS before Phase 0 research and rechecked after Phase 1 design.*

| Principle | Result | Evidence |
|---|---|---|
| IIT identity and academic usability | PASS | Reuse shared IIT UI tokens and components; provide accessible form, validation, loading, empty, success, and error states. |
| Maintainable layered architecture | PASS | Put admission and eligibility rules in the domain/application layers; keep controllers and Angular components focused on transport and presentation. |
| RESTful API contracts | PASS | Expose resource-oriented student routes with explicit DTOs, field errors, and documented account-creation semantics. |
| Security and data integrity | PASS | Enforce active-Admin access on each request, protect personal data, and constrain account, Degree, Batch, Hall, address-locality, and registration-number relationships in the database. |
| Reliable, simple, observable software | PASS | Test atomic account/admission creation, setup failure behavior, authorization, registration uniqueness, published-result unavailability, locality integrity, and credit/CGPA eligibility outcomes. |

No constitution exception, new framework, or dependency is required.

## Addendum Implementation Approach

1. Extend the admission-keyed read-only academic-results boundary to expose completed credits from passed course results and the current cumulative CGPA reported for that admission. Keep an explicit unavailable result when the owner source is absent; do not calculate, cache, or persist result totals in Student Management.
2. Extend the selected-Degree reference projection with `MinimumCgpaForCompletion`. Evaluate a `MinimumCgpa` eligibility requirement against that value and the published current cumulative CGPA; missing values are unavailable and never eligible. Include the CGPA and requirement status in the existing eligibility response.
3. Preserve `MailingUpazilaId` and `PermanentUpazilaId`, add nullable corresponding City Corporation references, and enforce that each supplied address selects exactly one locality. Add restrictive City Corporation foreign keys, locality indexes, and a versioned migration without changing existing Upazila selections.
4. Update address validation and DTOs to accept `districtId` as validation context and exactly one of `upazilaId` and `cityCorporationId`; verify the locality belongs to that District, derive Division from Configuration, and persist only the locality reference. Reject deleted, missing, inactive-new, or cross-District locality data. Protect referenced City Corporations from deletion under the existing Configuration dependent-record rule.
5. Update the Admin address form to load both Upazilas and City Corporations for the selected District and present one locality choice. Reset the locality when Division or District changes and retain separate address details and cell numbers.
6. Verify migration constraints and historical Upazila rows on PostgreSQL, eligibility thresholds and unavailable-source behavior through API tests, and both locality selection paths through Angular tests and the updated quickstart.

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
