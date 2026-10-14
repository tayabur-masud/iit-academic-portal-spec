# Data Model: Teacher Management

**Feature**: [spec.md](spec.md)  
**Research decisions**: [research.md](research.md)

## Entities

| Entity | Fields | Relationships and constraints |
|---|---|---|
| `TeacherProfile` | `Id` (server-generated UUID), `UserId`, `DisplayName`, nullable `EmployeeStaffId`, nullable `NormalizedEmployeeStaffId`, nullable `Phone`, nullable `DesignationId`, `IsActive`, inherited audit fields | Exactly one `PortalUser`; `UserId` is unique and restrictive. A supplied normalized employee/staff ID is globally unique across teacher profiles, including inactive profiles. `DesignationId` optionally references the existing Designation catalog with restrictive delete behavior. |
| `PortalUser` (existing Identity user, extended) | Existing Identity fields, `DefaultRole`, `AccountStatus` | Account status values: `PendingCredentialSetup`, `Active`, `Deactivated`. Existing accounts default to `Active` in the migration. Teacher provisioning creates PendingCredentialSetup; only a later approved credential-setup flow may transition it to Active. Identity `NormalizedEmail` remains globally unique. |
| Identity role assignment (existing) | `UserId`, `RoleId` | Teacher is required while its profile is active. Student is prohibited for teacher accounts. Admin may be assigned/removed by active Admins. Coordinator is derived from active batch-coordinator assignments and is never edited through the Teacher Management role endpoint. `DefaultRole` must be one of the current assignments. |

## Validation and Lifecycle

- `DisplayName` and Identity email are required. Email uniqueness follows the existing Identity normalized-email index.
- `EmployeeStaffId`, `Phone`, and `DesignationId` are optional. When supplied, `EmployeeStaffId` is trimmed and normalized using the existing name-normalization convention and must be unique across all teacher profiles.
- Maximum string lengths are bounded consistently with existing Identity/configuration conventions during implementation; the feature spec supplies no new business-specific length limits.
- A teacher profile is one-to-one with a PortalUser. A user cannot receive a second TeacherProfile.
- New teacher accounts have `AccountStatus = PendingCredentialSetup`; pending users cannot sign in or authenticate an existing session. Email delivery is not performed by this feature.
- Deactivation sets `TeacherProfile.IsActive = false` and `PortalUser.AccountStatus = Deactivated`, preserves Identity roles/profile/history, and revokes all active AuthSessions. Deactivated users cannot sign in or reuse existing sessions.
- Role changes preserve Teacher for active profiles, reject Student, permit Admin changes subject to last-Admin protection, and derive Coordinator from authoritative batch-coordinator assignment state. The default role remains assigned; if it becomes invalid, use existing `PortalRoles.ResolveDefault` behavior.
- The system must retain at least one active Admin account. The check and Admin-role removal/deactivation commit atomically under concurrency.
- Successful profile, account-status, role, and session changes commit with their allowlisted business audit event in one unit of work. No physical delete occurs.

## Coordinator Assignment Integration

Batch-coordinator assignments are not modeled in this feature. The future assignment owner calls the application synchronization contract with the teacher user ID and authoritative `hasActiveCoordinatorAssignments` state after each assignment change. Coordinator is added when the state is true and removed when false. The future assignment workflow must call this synchronizer in the same unit of work as its assignment update; end-to-end producer verification is deferred until that workflow exists.
