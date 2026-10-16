# Quickstart: Teacher Management Validation

**Feature**: [Teacher Management](spec.md)  
**Plan**: [Implementation plan](plan.md)  
**Contracts**: [Teacher API](contracts/teacher-management-api.md), [Coordinator role sync](contracts/batch-coordinator-role-sync.md)  
**Data model**: [data-model.md](data-model.md)

## Prerequisites

- Node.js 24 LTS and .NET 10 SDK.
- PostgreSQL 18 with the existing portal database configured and the Teacher Management migration applied through the schema-owner workflow.
- Development Admin account (`admin@iit.test`) configured with `DevelopmentSeed:Password`.
- Email delivery is not a prerequisite for this feature. Newly created teacher accounts remain PendingCredentialSetup and cannot sign in; do not configure a temporary-password or manual credential bypass.

## Build and Automated Tests

Run from the repository root after implementation:

```powershell
Push-Location iit-academic-portal-service
dotnet tool restore
dotnet ef database update -p src/IitAcademicPortal.Infrastructure -s src/IitAcademicPortal.Api
dotnet test
Pop-Location

Push-Location iit-academic-portal
npm ci
npm test -- --watch=false
npm run build
Pop-Location
```

Service tests use the existing xUnit API test host and SQLite pipeline fixture (`tests/IitAcademicPortal.Api.Tests/TeacherManagement/`). Set `AUDIT_TEST_POSTGRES` (for example `Host=127.0.0.1;Username=postgres;Password=postgres`) to also run the PostgreSQL 18 tests in `TeacherManagementPostgresTests`: the migration backfill (existing accounts become Active), the one-to-one profile link, the unique normalized staff ID (inactive profiles included, blanks may repeat), restrictive foreign keys, runtime-role grants, and the last-Admin lock under concurrency. Email delivery and email-specific end-to-end tests are deferred until the shared email mechanism is ready.

## End-to-End Scenarios

1. Start PostgreSQL and the service with its HTTPS profile; start Angular and sign in as the active Admin. Open the Teacher Management entry in the Admin module.
2. Create a teacher with display name and a unique email; optionally provide employee/staff ID, phone, and Designation. Verify exactly one linked portal user and Teacher role/default are created, the account is PendingCredentialSetup, and no email is sent.
3. Attempt to create another teacher with the same email or supplied employee/staff ID; verify field-level conflict errors and no partial profile/user creation.
4. Attempt sign-in with a pending account and an existing session for a non-active account; verify protected access is denied. Verify Admin can list/view pending profiles but cannot obtain or set the teacher's password.
5. Update teacher profile data. Add Admin to a teacher account and set a valid default; attempt to add Student and verify rejection. Verify Teacher remains assigned while the profile is active.
6. Add Admin to a teacher only when another active Admin exists. Attempt to remove/deactivate the last active Admin and verify conflict with no changes.
7. Call the internal Coordinator role synchronizer with active assignments true/false. Verify Coordinator is added for the first assignment, retained while another assignment remains, and removed after the last ends; verify Teacher and the default-role invariant remain valid. End-to-end batch assignment producer testing waits for that owning feature.
8. Deactivate a teacher. Verify profile/account status is Deactivated, all active sessions are revoked, the profile and audit history remain, and sign-in plus existing-session requests are denied.
9. Switch to Student/Teacher/Coordinator roles and call teacher-management routes directly; verify 403. Repeat anonymously for 401. Verify each successful management action has a safe, allowlisted audit event and audit failure rolls back the profile/account/role change.

## Where Things Are

- API: `POST/GET /api/admin/teachers`, `GET/PUT /api/admin/teachers/{id}`, `PUT /api/admin/teachers/{id}/roles`, `POST /api/admin/teachers/{id}/deactivation` (see the [API contract](contracts/teacher-management-api.md)).
- UI (Admin module): **Teachers** in the menu, `/admin/teachers` (directory), `/admin/teachers/new`, `/admin/teachers/:id` (profile, roles, status, deactivation, record history).
- Internal Coordinator contract: `ITeacherCoordinatorRoleSynchronizer` (see [the contract](contracts/batch-coordinator-role-sync.md)).
- Migration: `TeacherManagement` (adds `AspNetUsers.AccountStatus`, default `Active`, and `TeacherProfiles`). Apply it with the schema-owner workflow before using the feature on a database.

## Verification Results (2026-10-14)

- Latest `dotnet test` run (isolated artifacts, without `AUDIT_TEST_POSTGRES`): **576 passed, 55 skipped, 0 failed**. The skipped cases require PostgreSQL; the application-service concurrency test compiles and is discovered but has not run against PostgreSQL in this pass.
- `npm test -- --watch=false`: **125 passed** across 15 files. `npm run build`: succeeds.
- Covered by tests: provisioning (one pending account and profile, Teacher role and default, no password, no email, duplicate email and staff ID, designation rules, atomic rollback when the audit event fails), pending and deactivated accounts refused at sign-in and for existing sessions, role policy (Student and Coordinator refused, Teacher required, default role, held Coordinator kept, deactivated teacher conflict), last-Admin behavior, deactivation (sessions revoked, nothing deleted, repeat is a no-op, audited once), the directory (search, status filter, bounded infinite loading, validation), the Coordinator synchronizer (add, keep, remove, idempotent, default-role correction, rejections, commits only with the caller's save), the 401/403 matrix (including unchanged business state and expected access-denied events), and profile-update rollback when audit staging fails.
- The PostgreSQL concurrency test now exercises the actual role-removal and deactivation services. Run it with `AUDIT_TEST_POSTGRES` set before treating concurrent last-Admin behavior as verified.
- The default .NET build output was locked by the running API process; validation succeeded using the isolated `--artifacts-path` output directory.
- **Not run in this pass**: a live browser check (headless Chrome with axe-core) and applying the migration to the Development database; both remain to be done before relying on the feature there.
- **Still out of scope, as specified**: email delivery and credential setup (so a created teacher cannot sign in yet and cannot be reactivated), and the batch-coordinator assignment producer. The synchronizer is tested directly only; the producer's owner must call it in the assignment's unit of work and prove that with their own tests.

## Deferred Email Verification

Update (feature 005): the notification capability now exists and provisioning requests an invitation, but the secure setup link and activation are still not built, so delivery stays blocked and the scenarios below remain deferred.

When the shared email mechanism is ready, add follow-up scenarios for setup-message delivery, secure proof expiry/single use, retry/resend, pending-to-active transition, and delivery failure. Those tests are not part of this feature's current verification gate.
