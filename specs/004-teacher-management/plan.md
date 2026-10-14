# Implementation Plan: Teacher Management

**Branch**: `feature/004-teacher-management` | **Date**: 2026-10-14 | **Spec**: [spec.md](spec.md)

**Input**: [Teacher Management feature specification](spec.md)

## Summary

Allow active Admins to manage Teacher profiles linked one-to-one to Identity users, preserve role invariants, and deactivate teachers without deleting history. Persist account status so new accounts remain non-sign-in pending while email setup is deferred and deactivated users are denied at both sign-in and session validation. Reuse Identity, Admin policy, audit unit of work, and the existing layered monolith. Provide an internal Coordinator-role synchronization contract for the future batch-coordinator assignment owner; do not invent or add that assignment workflow here.

## Technical Context

**Language/Version**: TypeScript 5.9 / Angular 21; C# 14 / .NET 10.

**Primary Dependencies**: Existing ASP.NET Core Identity 10, EF Core 10, Npgsql 10.x, Angular HttpClient/reactive forms/router, Tailwind CSS 4, and Vitest. No new dependency or service is planned.

**Storage**: PostgreSQL 18 through `PortalDbContext`. Add a TeacherProfiles table, an account-status column on `AspNetUsers`, foreign keys/indexes, and a versioned EF Core migration. Existing users default to Active during migration.

**Testing**: xUnit API/integration tests using `WebApplicationFactory` and SQLite for request/validation/audit behavior; PostgreSQL 18 coverage for migrations, indexes, uniqueness, transactions, and last-Admin concurrency; Angular Vitest tests with `HttpTestingController`.

**Target Platform**: Browser-based Angular SPA and ASP.NET Core 10 API; local development uses Node.js 24 LTS, .NET 10 SDK, and PostgreSQL 18.

**Project Type**: Existing layered monolith with separate Angular frontend and ASP.NET Core API submodules.

**Performance Goals**: No feature-specific numeric latency target is set. Use bounded pagination for the teacher directory and avoid loading unnecessary user or role data.

**Constraints**: Only an active Admin may manage teacher profiles or role assignments. Teacher remains assigned for active profiles; Admin may be assigned/removed, Student is prohibited, and Coordinator is system-derived. Keep one active Admin account at all times. Teacher creation creates a pending account with no email send or alternate password handoff in this feature; email integration/delivery tests are deferred. Deactivation disables the account, revokes every active session, and preserves records/history. Batch-coordinator assignment storage/workflow is outside this feature; its future owner must call the defined role-sync integration. Use existing audit and design-system conventions.

**Scale/Scope**: One TeacherProfile per PortalUser; required display name and unique email, optional employee/staff ID (unique across all teacher profiles when present), phone/contact, and Designation. Identity role assignments remain in ASP.NET Identity tables. No courses, assessments, batch assignment UI, or email transport work is included.

## Constitution Check

*Gate before Phase 0: PASS. Re-checked after Phase 1 design: PASS.*

| Principle | Result | Evidence |
|---|---|---|
| IIT identity and usability | PASS | Admin directory/forms reuse `specs/design-system.md` tokens and accessible loading, empty, validation, and error states. |
| Layered architecture | PASS | Teacher profile/domain rules, application provisioning/role logic, EF persistence, thin API controllers, and Angular feature UI remain in existing layers. |
| REST contracts | PASS | Define resource-oriented Admin teacher routes, DTOs, field-keyed errors, and semantically appropriate status codes. |
| Security and integrity | PASS | Enforce active Admin on every operation, block Student role assignment, preserve one Admin, reject sign-in/session use for pending/deactivated accounts, and use unique constraints/foreign keys. |
| Reliability and observability | PASS | Account/profile/audit writes are atomic; role and deactivation actions are tested; secrets are never logged; email delivery tests are deferred by scope. |

No constitution exception or new dependency is required. Email delivery and its tests are deferred until the shared email mechanism is ready. The future batch-coordinator assignment feature is an explicit integration dependency for end-to-end Coordinator synchronization.

## Implementation Approach

1. Add a one-to-one TeacherProfile model, user account status (`PendingCredentialSetup`, `Active`, `Deactivated`), normalized optional staff identifier, and restrictive optional Designation reference. Add a migration with existing users defaulted to Active.
2. Add a teacher application service using Identity `UserManager` to create the user and profile with a Teacher role/default, without invoking email. Because Identity manager calls persist internally, wrap those calls, profile persistence, audit staging, and related updates in one explicit `PortalDbContext` transaction. Keep account status pending until a future credential-setup flow activates it.
3. Enforce account status in `PortalAuthenticationService.SignInAsync` and the validated-session read path so pending/deactivated users cannot authenticate or reuse existing sessions. Add a bulk active-session revocation method and a specific teacher-deactivation reason.
4. Add Admin-only list/read/create/update/role-update/deactivate API operations in `Controllers/Teachers/TeachersController.cs` and `Controllers/Teachers/TeacherDirectoryController.cs`. Enforce role policy (Teacher required while active, Admin assignable/removable, Student prohibited), resolve the default role from remaining assignments, and serialize last-Admin checks with role/deactivation writes.
5. Stage allowlisted `teacher.*` business audit events in the same `PortalDbContext` unit of work as identity/profile/role changes. On deactivation, update profile/account status and revoke all sessions atomically.
6. Add an internal `TeacherCoordinatorRoleSynchronizer` contract that the future batch-coordinator assignment owner calls with the authoritative post-change assignment state. It adds Coordinator when any active assignment exists and removes it after the last one ends; it does not expose a manual Coordinator-role endpoint.
7. Add the Admin Teacher Management directory/profile/role screens and route using current Angular navigation, forms, and problem-detail patterns. Display PendingSetup and Deactivated account states; do not send email or build an alternate credential handoff.
8. Verify API role boundaries, pending/deactivated sign-in denial, last-Admin protection, session revocation, audit atomicity, PostgreSQL constraints/migration, and Angular workflows. Test the Coordinator synchronizer directly. Do not claim assignment-trigger synchronization is complete until the future batch-assignment owner calls it in the assignment unit of work and adds end-to-end producer tests.

## Project Structure

### Documentation (this feature)

```text
specs/004-teacher-management/
├── plan.md
├── research.md
├── data-model.md
├── contracts/
│   ├── teacher-management-api.md
│   └── batch-coordinator-role-sync.md
└── quickstart.md
```

`tasks.md` is a later Phase 2 artifact and is not created by this planning workflow.

### Source Code (existing projects)

```text
iit-academic-portal-service/
├── src/IitAcademicPortal.Domain/Teachers/TeacherProfile.cs
├── src/IitAcademicPortal.Domain/Identity/PortalUser.cs
├── src/IitAcademicPortal.Application/Teachers/
├── src/IitAcademicPortal.Application/Abstractions/IAuthSessionRepository.cs
├── src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs
├── src/IitAcademicPortal.Infrastructure/Persistence/AuthSessionRepository.cs
├── src/IitAcademicPortal.Infrastructure/Persistence/Migrations/
├── src/IitAcademicPortal.Api/Controllers/Teachers/TeachersController.cs
├── src/IitAcademicPortal.Api/Controllers/Teachers/TeacherDirectoryController.cs
├── src/IitAcademicPortal.Api/Contracts/TeacherContracts.cs
└── tests/IitAcademicPortal.Api.Tests/TeacherManagement/

iit-academic-portal/src/app/features/teachers/
├── teacher-list.ts / teacher-list.html
├── teacher-form.ts / teacher-form.html
└── *.spec.ts
```

**Structure Decision**: Extend existing Domain → Application → Infrastructure → API projects and the Angular feature tree. Use `PortalDbContext`, existing Identity roles, the active `AdminModule` policy, the existing audit recorder, and the current test projects. Add no separate service or package.
