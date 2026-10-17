# Quickstart: User Management Validation

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Contracts**: [User Management API](contracts/user-management-api.md), [credential setup](contracts/account-credential-setup.md)  
**Data model**: [data-model.md](data-model.md)  
**Implementation notes**: [implementation-notes.md](implementation-notes.md)

## Prerequisites

- Node.js 24 LTS and .NET 10 SDK.
- PostgreSQL 18 development database with the User Management notification-template seed migration applied through the schema-owner workflow.
- Active Admin test account; non-production Admin/Student accounts and a Teacher linked to a TeacherProfile.
- Development email provider configured with a controlled test mailbox allowlist. Keep provider keys in user secrets/environment configuration; never send real credentials or data.
- Production-like tests use a durable/shared protected Data Protection key ring, runtime grants, and operational alert/recovery ownership.
- The first active Admin is established by the deployment/bootstrap owner; the Admin-only UI cannot create its own initial authority.
- Student profile and batch-coordinator assignment owners remain external dependencies for profile/Coordinator end-to-end tests.

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

Run PostgreSQL-gated tests with `AUDIT_TEST_POSTGRES` set to verify the setup-template seed, the shared last-Admin lock, migration behavior, and runtime grants. SQLite tests do not prove PostgreSQL-specific locks or grants.

## Usability Validation

Observe 10 representative Admins completing a standardized account-search task followed by an allowed maintenance action. Start timing when the task is presented and stop when the action's success state is visible. SC-002 passes when at least 9 of 10 participants complete both steps within two minutes without developer assistance. Record the task prompt, participant count, completion count, and elapsed times with the validation results.

## End-to-End Scenarios

1. Sign in as an active Admin. Search accounts by email/ID with role and status filters, inspect one account, and verify bounded loading and no secrets in the response.
2. Create an Admin and a Student with unique emails. Verify both start `PendingCredentialSetup`, have no password, get their one allowed role/default, and cause one setup-email request after the account/audit transaction commits.
3. Attempt to create a duplicate normalized email and invalid account type. Verify a safe conflict/validation result and no partial account, role, or audit state.
4. Create a Teacher through the existing Teacher Management flow and verify it uses the common setup-token purpose without creating a second account or TeacherProfile. Update Teacher profile data only through Teacher Management.
5. As an Admin, assign and remove Admin on a Teacher while preserving Teacher, Coordinator-derived state, and the default-role invariant. Verify Student-only accounts reject other roles and Coordinator cannot be directly assigned.
6. Complete account setup for Admin, Student, and Teacher using the emailed proof. Verify the account becomes Active only after a valid password is set and that the proof cannot be replayed or used as a password-reset proof.
7. Reissue a setup email for a pending account as an active Admin. Verify a fresh link is issued, prior link is invalidated, the operation is audited, and the API does not return proof material. Reject reissue for Active/Deactivated accounts and deny it to non-Admins.
8. Simulate setup-email or notification persistence failure. Verify the account remains pending, the failure signal is emitted, and the same public sign-in/recovery behavior does not disclose account status.
9. Deactivate an Admin/Student account and verify new sign-in and existing sessions fail while account and audit history remain. Verify an Admin cannot deactivate their own account. Deactivate a Teacher through Teacher Management and verify its profile/roles stay consistent.
10. Run concurrent last-Admin removals/deactivations through both generic and Teacher endpoints against PostgreSQL. Verify the shared transaction lock leaves at least one active Admin.
11. Verify account/setup audit records contain actor, target, action, outcome, and safe change summary only; no password, token, setup link, or rendered security body appears in audit, logs, or permanent notification history.
12. Confirm Coordinator provisioning remains blocked to direct User Management writes until the batch-assignment owner creates the account/authoritative assignment and calls the synchronizer. For a newly provisioned Coordinator account, verify the owner requests `AccountCredentialSetup` after commit and the account remains pending until password setup; do not mark that integration complete before its owner tests pass.

## Where Things Are

- Admin account UI: `/admin/users`; Teacher-specific editing continues at `/admin/teachers`.
- Admin API and public setup completion: see [user-management-api.md](contracts/user-management-api.md).
- Credential event/proof rules: see [account-credential-setup.md](contracts/account-credential-setup.md).
- Existing account and role state: `PortalUser`, Identity roles/user-role assignments, and `AuthSessions` in `PortalDbContext`.
- One EF migration is expected for the `AccountCredentialSetup` notification template/revision seed. No new account column/table is expected unless implementation discovers a missing schema requirement.

## Validation Results (2026-10-18)

Run on Windows 11 with .NET 10 SDK, Node 24, and a local PostgreSQL 18 server.

| Check | Result |
|---|---|
| `dotnet ef database update` (Development database) | Applied `20261017200942_AccountCredentialSetupTemplate`; it inserts only the setup template and its first revision |
| `dotnet test` without `AUDIT_TEST_POSTGRES` | 875 passed, 70 skipped (69 PostgreSQL-gated tests, plus the Coordinator release-gate test) |
| `dotnet test` with `AUDIT_TEST_POSTGRES` | 944 passed, 1 skipped (the Coordinator release-gate test) |
| `npm test -- --watch=false` | 251 passed (24 files) |
| `npm run build` | Succeeded; initial bundle 473.40 kB (budget 500 kB); the account pages and the setup page are lazy chunks |

New automated coverage, by scenario above: directory and authorization (1) in `UserDirectoryTests`; provisioning and duplicates (2, 3) in
`UserProvisioningTests`; Teacher owner path (4) and the Coordinator gate (12) in `AccountCredentialSetupOwnerTests`; roles (5), email,
deactivation, and self-deactivation (9) in `UserAccessAndLifecycleTests`; setup completion, purpose separation, replay, expiry, reissue, and
delivery failure (6, 7, 8) in `AccountCredentialSetupTests`; shared last-Admin lock and seed (10) in `UserManagementPostgresTests`; privacy (11)
in `AccountCredentialPrivacyTests`; and the Angular specs under `src/app/features/users/`.

### Not done

- **SC-002 usability study.** No Admin participants were observed, so SC-002 is unverified and T031 stays open. Record the task prompt,
  participant count, completion count, and elapsed times here when it is run.
- **Coordinator setup email (scenario 12).** No batch-assignment owner exists. The producer test is skipped with the gate as its reason;
  Coordinator provisioning is not complete.
- **Manual end-to-end with a real mailbox.** Not run here. With the Development Brevo allowlist, create an account, open the emailed link at
  `/account-setup`, set a password, and sign in.
- **Operational gates** before Production: the public https setup URL (`AccountCredentialSetup:SetupPageUrl` or `Notifications:PortalUrl`), a
  persisted and shared Data Protection key ring, the first-Admin bootstrap, and a Production email provider whose retention is verified.

