# Implementation Plan: My Profile

**Branch**: `010-my-profile` | **Date**: 2026-10-23 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/010-my-profile/spec.md`

## Summary

Add a read-only My Profile page for every supported active portal role. The authenticated account ID and active role come from the validated session; a purpose-built application read service selects the corresponding Staff, Student, or Teacher profile and returns only the fields approved for that role. Coordinator uses its linked Teacher profile. A dedicated API response omits account/profile identifiers, audit metadata, credential data, and Student admission summaries. No new persisted data or database migration is required.

## Technical Context

**Language/Version**: TypeScript 5.9 / Angular 21; C# 14 / ASP.NET Core 10.

**Primary Dependencies**: Existing Angular Router, HttpClient, reactive forms/components; ASP.NET Core MVC, Identity session authentication, EF Core stores, PostgreSQL provider. No new package is planned.

**Storage**: Existing PostgreSQL 18 `PortalDbContext` entities. Read-only access to PortalUser, StaffProfile, StudentProfile/StudentAcademicHistory, TeacherProfile, and Designation. No schema change or migration.

**Testing**: xUnit API integration tests using the existing test host/SQLite fixtures; Angular Vitest tests for role-specific rendering, navigation, error/empty states, and view-only behavior. PostgreSQL migration tests are not applicable because the schema does not change.

**Target Platform**: Authenticated browser SPA and ASP.NET Core 10 API; Node.js 24 LTS and .NET 10 SDK for development.

**Project Type**: Existing layered monolith with Angular frontend and ASP.NET Core REST API submodules.

**Performance Goals**: No separate latency SLO is specified. Return one signed-in user's role-specific profile and do not load Student admissions or academic-result records as part of this view.

**Constraints**: Support active Admin, Student, Teacher, and Coordinator roles. Derive user identity and active role from server session claims; accept no user ID or role selector from the client. Coordinator maps to TeacherProfile. Student view includes all approved personal profile and prior-education details but excludes internal IDs, audit metadata, credentials, and admissions/results. Reuse existing profile data and management ownership; provide no writes.

**Scale/Scope**: One request returns one current account's profile for its active role. Multi-role accounts see the matching profile for the active role; the account identity remains constant across role changes. No profile search, bulk reads, exports, or new persistence.

## Constitution Check

*GATE: PASS before Phase 0 research; rechecked PASS after Phase 1 design.*

| Principle | Result | Evidence |
|---|---|---|
| IIT Visual Identity and Academic Usability | PASS | Reuse the account shell and design-system tokens; provide accessible navigation and loading, empty, and error states for all supported roles. |
| Maintainable Layered Architecture | PASS | Keep role resolution in an application service, database projections in existing profile stores, a thin API controller, and presentation in an Angular profile feature. |
| RESTful API Contracts | PASS | Use a read-only current-user profile resource with an explicit allowlisted response DTO and consistent error behavior. |
| Security and Data Integrity | PASS | Scope by authenticated user ID and active-role claims; accept no target identity from the client, omit secrets/internal metadata, and disable caching through existing API middleware. |
| Reliable, Simple, and Observable Software | PASS | Test every active role, cross-account denial, missing profiles, role switching, sensitive-field exclusions, and view-only behavior; no schema or external dependency changes. |

No exception, new framework, package, or database migration is required.

## Implementation Approach

1. Add a role-aware `MyProfileService` and view models in the Application layer, register the service in `iit-academic-portal-service/src/IitAcademicPortal.Application/DependencyInjection.cs`, and receive the authenticated user ID and active role from the controller. Resolve account email and dispatch to the matching read-only profile query; do not accept an account ID or role from the request.
2. Extend profile store read methods by user ID for Staff and Teacher. Add a Student self-view projection that returns StudentProfile fields, resolved addresses, and prior academic history without loading account audit metadata or admission summaries. Do not reuse the existing full Student profile detail response directly because it includes every admission.
3. Map Admin to StaffProfile, Student to StudentProfile, Teacher to TeacherProfile, and Coordinator to the same account's TeacherProfile. Return a profile-unavailable state when a role-linked profile is missing; do not fall back to another role's profile.
4. Add an authenticated `GET /api/profile` endpoint and explicit role-discriminated response contracts. The service uses the validated session's NameIdentifier and active role claims. Return only the profile fields described in the data model; do not expose internal identifiers, audit fields, account status, roles, credentials, or setup proofs.
5. Add a lazy-loaded My Profile page under the authenticated shell, enable the account-menu destination for all roles, and render the profile variant matching the active role. Keep the page view-only and refresh data after navigation/role changes; reuse accessible loading, empty, and recoverable error patterns.
6. Add API and Angular tests for all four roles, Student sensitive-field allowlisting, Coordinator mapping, multi-role role switching, missing profiles, anonymous/cross-account denial, no-store responses, and absence of write actions. Update the quickstart with runnable checks; no PostgreSQL migration test is needed.

## Project Structure

### Documentation (this feature)

```text
specs/010-my-profile/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── my-profile-api.md
├── checklists/
│   └── requirements.md
└── tasks.md                 # Generated later by /speckit-tasks
```

### Source Code (repository root)

```text
iit-academic-portal-service/
├── src/IitAcademicPortal.Application/MyProfile/
│   ├── MyProfileService.cs
│   └── MyProfileModels.cs
├── src/IitAcademicPortal.Application/DependencyInjection.cs
├── src/IitAcademicPortal.Application/Abstractions/
│   ├── IStaffStore.cs
│   ├── IStudentStore.cs
│   └── ITeacherStore.cs
├── src/IitAcademicPortal.Infrastructure/Persistence/
│   ├── StaffStore.cs
│   ├── StudentStore.cs
│   └── TeacherStore.cs
├── src/IitAcademicPortal.Api/Controllers/Profile/MyProfileController.cs
├── src/IitAcademicPortal.Api/Contracts/MyProfileContracts.cs
└── tests/IitAcademicPortal.Api.Tests/Profiles/MyProfileTests.cs

iit-academic-portal/
└── src/app/
    ├── features/profile/my-profile.ts
    ├── features/profile/my-profile.html
    ├── features/profile/my-profile-api.ts
    ├── features/profile/my-profile.models.ts
    ├── features/profile/my-profile.spec.ts
    ├── app.routes.ts
    └── layout/shell/shell.html
```

**Structure Decision**: Extend the existing Application → Infrastructure → API layers and the Angular authenticated shell. Reuse the Student, Teacher, and Staff stores with user-scoped reads; do not create a new project, persistence table, or generic profile framework.

## Complexity Tracking

No constitution violations or justified complexity exceptions.
