# Data Model: User Management

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Status**: Implemented on existing Identity persistence. The only migration, `AccountCredentialSetupTemplate`, seeds the setup template and its first revision; no account table or column was added. Session revocation gained the reason `AccountDeactivated` (stored as text, so no migration).

## Modeling Decisions

- Reuse ASP.NET Core Identity's `PortalUser`, role assignments, normalized-email index, and the existing `AccountStatus` and `DefaultRole` fields. Do not create a duplicate account/credential table.
- The account directory projects account identity and access data. Teacher-specific attributes remain on the existing `TeacherProfile`; Student academic/profile data and Coordinator assignments remain with their owning features.
- Keep audit history append-only and credential/setup proofs transient. Never put a password, setup proof, reset proof, or rendered security body in an account response, audit record, log, or permanent notification history.
- A version-controlled migration is expected to add the initial template/revision seed for `AccountCredentialSetup`; no account schema change is expected.

## Logical Entities

### Portal Account (existing Identity user)

Stable authentication identity. Reuse these existing attributes; do not introduce a parallel user entity.

| Attribute | Meaning and constraints |
|---|---|
| User ID | Stable Identity key referenced by sessions, role assignments, audit, and feature records. |
| Email / normalized email | Registered sign-in and setup-email destination; required and unique by the existing normalized-email constraint. Email changes use Identity's normalization and conflict handling. |
| Password hash | Identity-managed credential verifier; never returned or logged. A newly provisioned account has no password until secure setup completes. |
| Account status | Existing `PendingCredentialSetup`, `Active`, or `Deactivated`; only `Active` may authenticate. |
| Default role | Existing nullable value; when set it must be one of the assigned roles. |

### Role Assignment (existing Identity user-role relation)

- A Student account has only Student.
- An active Teacher retains Teacher. Admin may be added or removed through the Teacher Management role owner.
- Coordinator is derived from an active batch-coordinator assignment and is never directly assigned by User Management.
- Admin accounts use their approved Admin assignment. User Management cannot remove/deactivate the last active Admin.
- A Teacher's profile/role lifecycle stays with Teacher Management; User Management reads account-linked Teacher information without duplicating it.

### Credential Setup Operation (transient flow; no persisted entity)

- A setup proof is generated and verified by the configured Identity Data Protection token provider with a purpose distinct from password recovery.
- The proof is delivered through a security notification and is never stored in User Management tables, returned by the Admin API, or written to logs/audit.
- Reissue is allowed only while the account is pending and requested by an active Admin. Rotate the account security stamp before issuing the fresh token so earlier setup links no longer validate.
- Successful completion requires a valid purpose-specific proof and an account still in `PendingCredentialSetup`; Identity validates/adds the password, the account transitions to `Active`, the security stamp rotates, and a safe audit event commits in the same transaction.
- Failed/expired proof or delivery does not activate the account. The user can request an Admin-authorized fresh setup email while the account remains pending.

### Account Audit Event

Allowlisted account/access change fact containing stable actor and target IDs, action, UTC timestamp, outcome, and approved field-change summary. It never includes passwords, tokens, setup links, or raw provider messages.

## Ownership Boundaries

- Admin and Student generic identities may be provisioned by User Management with their single role and pending status.
- Teacher account/profile creation and Teacher-specific role/deactivation updates use the existing Teacher Management service and `TeacherProfile`; do not create a generic account that bypasses it.
- Coordinator membership is not a directly editable account role. Coordinator identity/role provisioning requires the batch-assignment owner to create or update the authoritative assignment and call the existing synchronization contract. When that owner provisions a new Coordinator account, it MUST request the common `AccountCredentialSetup` event after the account/assignment transaction commits; the account remains pending until setup completes. User Management does not create the Coordinator account or assignment.
- User Management lists accounts across all roles, but does not create TeacherProfile, StudentProfile, or batch-assignment data.

## State Transitions

1. **Provisioned**: Admin/Student generic identity is created in `PendingCredentialSetup`, with the assigned role and default role set to that role; Teacher provisioning remains owned by Teacher Management, and Coordinator account/assignment provisioning remains owned by the batch-assignment workflow.
2. **Setup email requested**: The account-setup event is durably requested after the account/audit transaction commits. Provider failure does not activate the account or fail the originating account operation.
3. **Setup link reissued**: Active Admin confirms reissue for a still-pending account. The security stamp is rotated and a new notification occurrence is requested; previous proofs become invalid.
4. **Active**: A valid setup proof is verified, the Identity password is set, status becomes `Active`, and audit commits atomically.
5. **Deactivated**: An authorized owner deactivates the account, revokes all active sessions, retains Identity/role/history records, and writes the required audit fact.

## Persistence and Migration Impact

Existing tables already contain `AspNetUsers.DefaultRole`, `AspNetUsers.AccountStatus`, Identity roles/user-role assignments, `AuthSessions`, notification records/templates, and audit history. User Management needs no new account column or table if the current schema matches the checked-in model.

Add an EF Core migration to seed the common `AccountCredentialSetup` event template and initial revision and update the model snapshot. If implementation adds a persistent setup-token table or account field, it requires a separate version-controlled migration and explicit security review; the chosen design avoids that persistence. Reuse the existing runtime grants for current tables. PostgreSQL tests must verify the seed migration and shared last-Admin lock behavior.
