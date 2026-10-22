# Quickstart: Staff Management Validation

**Feature**: [Staff Management](spec.md)  
**Plan**: [Implementation plan](plan.md)  
**Data model**: [data-model.md](data-model.md)  
**API contract**: [staff-management-api.md](contracts/staff-management-api.md)

## Prerequisites

- Node.js 24 LTS, .NET 10 SDK, and PostgreSQL 18 configured per the service setup documentation.
- An active Admin session, secure credential-setup notification configuration, and at least one existing portal account with the Admin role for the account-attachment scenario.
- At least one active Designation in Configuration.
- An institution-approved source from which the Admin can manually verify each staff Name, Mobile number, and Designation before saving.
- The `StaffProfiles` migration and follow-up StaffProfile lifecycle migration applied through the existing schema-owner workflow before running against PostgreSQL.

## Automated Checks

Run from the workspace root:

```powershell
Push-Location iit-academic-portal-service
dotnet test tests/IitAcademicPortal.Api.Tests/IitAcademicPortal.Api.Tests.csproj
Pop-Location

Push-Location iit-academic-portal
npm test -- --watch=false
npm run build
Pop-Location
```

Run focused PostgreSQL integration tests with `AUDIT_TEST_POSTGRES` configured to verify the StaffProfiles migration, unique portal-account relationship, restrictive Designation foreign key, and audit transaction behavior.

## End-to-End Scenarios

1. Sign in as an active Admin. Create staff with a new email, Name, Mobile number, active Designation, and manual-verification confirmation. Verify exactly one PortalUser and StaffProfile are committed together, the account has Admin as its role/default, has no password, is pending setup, and the setup email is requested only after commit. Verify account/profile creation and audit events are present.
2. Create a staff profile using an email that belongs to an existing active Admin account with no StaffProfile. Verify the existing account ID, credentials, role, and status are retained and no setup email is requested again. Verify a deactivated or non-Admin account, or an account already linked to a StaffProfile, is rejected without partial changes.
3. Open another active staff profile and update Name, Mobile number, or Designation after manual verification. Confirm the verification attestation is required and recorded in the same audit transaction as the profile change.
4. Update the signed-in Admin's own StaffProfile through the same Staff Management flow as for another person. Confirm the same validation, verification, and audit behavior; the My Profile menu remains view-only.
5. Submit create/update requests without verification confirmation, with an inactive/nonexistent Designation, or with missing/blank/over-limit Name or Mobile number. Verify field-keyed rejection, no profile/account changes, and no successful-change audit event.
6. Try creating another StaffProfile for an account that already has one. Verify conflict and no duplicate profile/account.
7. Deactivate a different staff member. Verify StaffProfile becomes inactive, the linked Admin account is deactivated, active sessions are revoked, and profile/audit history remains. Verify self-deactivation and deactivation of the last active Admin are refused without changes; repeating deactivation is a no-op.
8. Call list/detail/create/update/deactivation anonymously and with a non-Admin active role. Verify 401/403 responses and no staff personal information disclosure.
9. Open the account dropdown's My Profile menu/page. Verify it remains view-only and exposes no staff-profile update control or write request.
10. Verify Staff Management does not assign/change roles, update email/password, edit StudentProfile/TeacherProfile, grant permissions through Designation, or upload/store verification documents.

## Database Migration

The applied `20261021174956_StaffProfiles` migration creates the base table, unique restrictive `UserId` foreign key, required restrictive `DesignationId` foreign key, and inherited audit columns. A follow-up versioned migration adds the required `IsActive` profile lifecycle state (existing rows default active); do not rewrite the migration already applied to Development. Apply the follow-up through the existing schema-owner workflow after PostgreSQL tests are reviewed. Do not alter TeacherProfile.

## Verification Record (2026-10-22, provisioning and deactivation)

Run on Windows 11 with .NET 10 SDK, Node 24, and a local PostgreSQL 18 server (`AUDIT_TEST_POSTGRES` set to its administrator connection
string).

| Check | Result |
|---|---|
| `dotnet build -c Release` (service) | Succeeded |
| `dotnet test -c Release` with `AUDIT_TEST_POSTGRES` | 1220 passed, 1 skipped (the feature 006 Coordinator release-gate test) |
| Staff tests | `StaffManagementTests` 29, `StaffProvisioningTests` 4 (notification test host), `StaffManagementPostgresTests` 4: all passed |
| `npm test -- --watch=false` | 352 passed (30 files); `staff-management.spec.ts` and `shell.spec.ts` 28 of them |
| `npm run build` | Succeeded; initial bundle 457.11 kB (budget 500 kB) |
| Migration `20261022043821_StaffProfileLifecycle` | Adds `IsActive` (default true); verified on PostgreSQL by migrating to `StaffProfiles`, inserting a profile, then migrating fully and reading it back active. Applied to the disposable test databases, and on 2026-10-22 to the Development database at the project owner's request (column `boolean NOT NULL DEFAULT true`; no shared or Production database changed) |

Automated coverage: scenario 1 (pending Admin, no password, one setup request after commit, both audit events, sign-in after setup,
setup-request failure, rollback when the audit fails) in `StaffProvisioningTests` and `StaffManagementPostgresTests`; 2 and 6 (link, refused
accounts, duplicate) and 3-5 and 7-10 in `StaffManagementTests` (deactivation revokes sessions and a revoked client gets 401, repeat no-op,
self and last-Admin refusals, no delete/reactivation/`me` routes, eligible-accounts gone); deactivation atomicity on PostgreSQL; the form,
create outcomes, in-row deactivation, and inactive state in `staff-management.spec.ts`; My Profile view-only in `shell.spec.ts`.

**Not done:** the hand-run browser walkthrough of scenarios 1-10, including a real setup email.

## Verification Record (2026-10-21, previous implementation)

Run on Windows 11 with .NET 10 SDK, Node 24, and a local PostgreSQL 18 server (`AUDIT_TEST_POSTGRES` pointing at it; each PostgreSQL
test class uses its own throwaway database built with the real migrations and the runtime-role grants).

| Check | Result |
|---|---|
| `dotnet test` (Release) with `AUDIT_TEST_POSTGRES` | 1210 passed, 1 skipped (the feature 006 Coordinator release-gate test) |
| Staff tests | `StaffManagementTests` (25, SQLite API host) and `StaffManagementPostgresTests` (2) all passed within the run above |
| `npm test -- --watch=false` | 345 passed (30 files) |
| `npm run build` | Succeeded; initial bundle 457.04 kB (budget 500 kB); the staff list and form are lazy chunks |
| Migration `20261021174956_StaffProfiles` | Creates only `StaffProfiles` with its unique `UserId` index, `DesignationId` index, and two restrictive keys; applied to the disposable PostgreSQL test databases, and on 2026-10-22 to the Development database at the project owner's request (no shared or Production database changed) |

Automated coverage by scenario:
1-3 (create, update another Admin, update one's own profile through the same route, audit with the confirmation) and 4 (missing
confirmation, invalid fields and Designations) and 5 (second profile) and 6 (anonymous 401, non-Admin 403, anti-forgery) and 7 (no
delete, no `/me`, no My Profile write) and 8 (account, roles, status, phone, and teacher profile unchanged; no document field) in
`StaffManagementTests`; the unique account, restrictive keys, column limits, no verification column, and profile/audit atomicity on
PostgreSQL in `StaffManagementPostgresTests`; the directory, form, confirmation, and refused saves in `staff-management.spec.ts`; and the
Staff navigation entry and the disabled, view-only My Profile action in `shell.spec.ts`.

**Evidence scope:** The verification record above covers the original profile-only Staff Management implementation and its eligible-account picker. It does not validate the newly specified account provisioning/attachment or deactivation behavior.

