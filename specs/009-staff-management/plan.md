# Implementation Plan: Staff Management

**Branch**: `feature/009-staff-management` | **Date**: 2026-10-21 | **Spec**: [spec.md](spec.md)

**Input**: [Staff Management specification](spec.md)

## Summary

Provide an Admin-managed StaffProfile with required Name, Mobile number, and Designation. For a new email, create a pending PortalUser with the Admin role/default role and StaffProfile atomically, then request the existing credential-setup flow after commit. For an existing active Admin account with no profile, attach a StaffProfile without changing credentials/status or requesting setup; reject deactivated, non-Admin, or already-profiled accounts. Admins confirm manual verification on every profile create/update, and the confirmation is recorded in the atomic audit event. Teacher-style deactivation marks the StaffProfile inactive, disables the linked Admin account, revokes sessions, and preserves history with last-Admin/self-deactivation protections. My Profile stays view-only and out of scope.

## Technical Context

**Language/Version**: TypeScript 5.9 / Angular 21; C# 14 / .NET 10.

**Primary Dependencies**: Existing Angular HttpClient, Router, reactive forms, ASP.NET Core, EF Core, Identity, audit, and Configuration services. No new package is planned.

**Storage**: PostgreSQL 18 through the existing `PortalDbContext`; preserve the applied `StaffProfiles` migration and add a follow-up versioned migration for the required `IsActive` profile state. Keep unique restrictive `UserId` and required restrictive `DesignationId` foreign keys. The profile stores Name, Mobile number, Designation reference, and active/inactive state; manual-verification confirmation is audit data, not a verified-state column.

**Testing**: xUnit API/integration tests with the existing test host and SQLite fixture; PostgreSQL 18 tests for profile uniqueness, restrictive account/Designation references, lifecycle migration, provisioning/attachment/deactivation integrity, session revocation, and atomic audit; Angular Vitest tests for email-based create/link, profile updates, deactivation, and view-only My Profile behavior.

**Target Platform**: Browser-based Angular SPA and ASP.NET Core 10 API; Node.js 24 LTS, .NET 10 SDK, and PostgreSQL 18 for local development.

**Project Type**: Existing layered monolith with separate Angular frontend and ASP.NET Core REST API submodules.

**Performance Goals**: No numeric SLO is specified. Keep Staff directory reads bounded and searchable by Name, account email, Mobile number, and Designation.

**Constraints**: Only callers actively using Admin may use Staff Management. New accounts are created with Admin as default role and remain pending until existing credential setup completes; existing active Admin accounts without profiles may be linked without account changes. User Management remains the owner of account identity, role changes, credentials, and ordinary lifecycle; Staff Management deactivation follows Teacher lifecycle and existing self/last-Admin protections. Designation references the existing active Configuration catalog. Every profile create/update requires explicit verification confirmation in the atomic audit event. Do not store verification documents, add self-service My Profile writes, create a Staff/Super Admin role, or modify StudentProfile/TeacherProfile.

**Scale/Scope**: At most one StaffProfile per Admin-role portal account. Profiles can be created with new Admin accounts or attached to existing eligible Admin accounts, and can be deactivated with their linked account. Each profile has required Name, Mobile number, and Designation reference; no employee identifier, Staff role, or My Profile feature is included.

## Constitution Check

*GATE: PASS before Phase 0 research; rechecked PASS after Phase 1 design.*

| Principle | Result | Evidence |
|---|---|---|
| IIT Visual Identity and Academic Usability | PASS | Reuse the existing Admin shell, shared components, and accessible form/error patterns; My Profile remains view-only. |
| Maintainable Layered Architecture | PASS | Keep staff profile rules in Domain/Application, persistence in `PortalDbContext`, and UI under the existing staff-management feature; no new project or service. |
| RESTful API Contracts | PASS | Use resource-oriented Admin staff collection/item operations with explicit DTOs and field-keyed errors. |
| Security and Data Integrity | PASS | Enforce active-Admin access; create Admin accounts only through the Staff workflow's atomic provisioning path; retain User Management role ownership; enforce designation/account constraints and last-Admin protections. |
| Reliable, Simple, and Observable Software | PASS | Test provisioning/attachment, credential setup, verification confirmation, audit atomicity, deactivation/session revocation, role boundaries, and PostgreSQL constraints. |

No constitution exception, new framework, or dependency is required.

## Implementation Approach

1. Extend `StaffProfile` with required Name, Mobile number, and Designation plus an active/inactive lifecycle state. Keep a unique restrictive link to one PortalUser and a required restrictive Designation reference.
2. Preserve the applied `StaffProfiles` migration and add a versioned follow-up migration for the StaffProfile lifecycle column; do not rewrite deployed migration history. Keep the UserId and Designation foreign keys and unique account index.
3. Add Staff create logic keyed by email. For a new email, create one `PortalUser` with Admin as its role/default role, `PendingCredentialSetup`, and no password, plus StaffProfile and account/profile audit events in one transaction. If the email matches an existing active Admin with no profile, attach StaffProfile and its audit without changing account role/status/credentials. Reject deactivated/non-Admin accounts and duplicate profiles. Do not call generic UserProvisioningService in a way that commits the account separately from the profile.
4. After a new account/profile transaction commits, request the existing credential-setup flow. A delivery failure leaves the account pending without rolling back committed records; do not send another setup request when attaching to an existing Admin account. User Management retains credential setup and later role administration.
5. Update profile fields through the existing Staff management use case. Validate Name/Mobile limits and active Designation; require `manualVerificationConfirmed: true` on create and update. Stage allowlisted staff profile audit events with the attestation in the same transaction; do not store verification documents or a permanent verified state.
6. Add Teacher-style deactivation: in one transaction set StaffProfile inactive, set the linked PortalUser to Deactivated, revoke all active sessions, and record profile/account audit. Reuse the existing last-Admin lock and reject self-deactivation or deactivation of the last active Admin. Preserve history; treat repeat deactivation as a no-op and do not offer reactivation.
7. Expose active-Admin-authorized collection/detail, create/link-by-email, update, and deactivation endpoints with explicit DTOs and field-keyed errors. Remove the obsolete eligible-account-picker route/UI; the create form accepts email and the server performs safe account resolution.
8. Add Staff list/form UI under the existing Admin feature area. Admins, including the current Admin, use the same Staff Management workflow; My Profile remains view-only. Reuse Designation options and existing accessible loading, empty, validation, success, and error patterns.
9. Validate provisioning versus attachment, pending-account/setup behavior, audit atomicity, designation/account constraints, deactivation/session revocation and last-Admin safeguards through SQLite/API/PostgreSQL tests and Angular tests; document runnable scenarios in the quickstart.

## Project Structure

### Documentation (this feature)

```text
specs/009-staff-management/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── staff-management-api.md
├── checklists/
│   └── requirements.md
└── tasks.md
```

### Source Code (repository root)
```text
iit-academic-portal-service/
├── src/IitAcademicPortal.Domain/Staff/StaffProfile.cs
├── src/IitAcademicPortal.Application/Staff/StaffProfileService.cs
├── src/IitAcademicPortal.Application/Staff/StaffManagementModels.cs
├── src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs
├── src/IitAcademicPortal.Infrastructure/Persistence/Migrations/
├── src/IitAcademicPortal.Api/Controllers/Staff/StaffController.cs
├── src/IitAcademicPortal.Api/Contracts/StaffManagementContracts.cs
└── tests/IitAcademicPortal.Api.Tests/Staff/StaffManagementTests.cs

iit-academic-portal/src/app/features/staff/
├── staff-api.ts
├── staff.models.ts
├── staff-list.ts / staff-list.html
└── staff-form.ts / staff-form.html

iit-academic-portal/src/app/app.routes.ts
iit-academic-portal/src/app/layout/shell/shell.ts
```

**Structure Decision**: Extend the existing Domain → Application → Infrastructure → API layering and Angular Admin feature tree. Keep accounts and role assignments in User Management; StaffProfile is a role-specific profile linked to an existing PortalUser and Configuration Designation. Reuse existing authorization, audit, validation, and UI components; add no new project or package.

## Complexity Tracking

No constitution exception, new framework, or dependency is required.
