# Data Model: Staff Management

**Status**: `StaffProfiles` is implemented by `20261021174956_StaffProfiles`; `20261022043821_StaffProfileLifecycle` adds `IsActive` (default true).

**Feature**: [Staff Management](spec.md)  
**Research decisions**: [research.md](research.md)

## Shared Conventions

- Use server-generated `Guid` identifiers and derive `StaffProfile` from the shared `AuditedEntity` so create/update actor and UTC timestamps are server-stamped.
- Do not expose normalized comparison fields, audit actor fields, or account role assignments as writable profile fields.
- The Staff Management profile is eligible only while its linked `PortalUser` has the existing Admin role. The role check is enforced in the application; User Management remains authoritative for assignments.
- A new-email StaffProfile and its PortalUser are created atomically; an Admin account that is not deactivated (active or pending setup) and has no profile may be linked without changing its status, credentials, or roles.
- Staff deactivation marks the profile inactive, deactivates its linked account, revokes sessions, and preserves both records and audit history. Self-deactivation and deactivation of the last active Admin are rejected. A deactivated profile is not reactivated through Staff Management.

## StaffProfile

One Admin staff member's directory/profile data, linked one-to-one to a newly provisioned or eligible existing portal account.

| Field | Type | Required | Rules |
|---|---|---:|---|
| Id | UUID | Yes | Stable profile identifier. |
| UserId | Portal user identifier | Yes | Restrictive foreign key to `PortalUser`; unique so one account has at most one StaffProfile. |
| Name | String | Yes | Required display name; trim surrounding whitespace; maximum 200 characters following the existing staff/profile name convention. |
| MobileNumber | String | Yes | Required contact number; trim surrounding whitespace; maximum 50 characters following the existing phone convention. It is not represented as phone-verified. Stored on StaffProfile, not synchronized to Identity `PhoneNumber`. |
| DesignationId | UUID | Yes | Restrictive foreign key to an existing active `Designation`; display name is resolved from Configuration. Designation is descriptive and does not grant authorization. |
| IsActive | Boolean | Yes | Defaults true. Deactivation sets false and deactivates the linked Admin account; the profile/history is retained and cannot be reactivated through Staff Management. |
| Audit fields | Existing `AuditedEntity` fields | Yes | Server-generated created/modified actor and UTC timestamps. |

### Verification Confirmation

The create/update request requires `manualVerificationConfirmed: true`. It is request/audit data, not a StaffProfile column or an enduring verified state. The allowlisted audit event records that the acting Admin confirmed verification, together with the profile target, action, UTC time, and changed fields. If confirmation is absent/false or audit persistence fails, the profile change does not commit. No verification document or proof is stored.

## Relationships and Integrity

- One StaffProfile references exactly one `PortalUser`; a PortalUser can have at most one StaffProfile. A new account has Admin as its sole role/default role. An existing account is linkable only when it is not deactivated, has Admin, and has no StaffProfile.
- StaffProfile has one required Designation reference. A Designation may be referenced by many StaffProfiles and TeacherProfiles. Physical deletion of a referenced Designation is prevented by restrictive foreign keys; Configuration's existing dependent-record checks should report the StaffProfile use as a delete conflict.
- The Admin-role requirement is an application-level account eligibility rule, not a database role foreign key. User Management owns subsequent role changes; the StaffProfile remains stored if its Admin role is later removed.
- StaffProfile has no credentials, login email, employee identifier, verification-state flag, or self-service edit state. Read email/account status from its linked PortalUser.
- New PortalUser, StaffProfile, `account.created` and `staff.profile.created` events commit atomically; request credential setup only after commit. Attaching to an existing Admin account creates only StaffProfile and its event and does not request setup again.
- Deactivation updates StaffProfile `IsActive`, PortalUser `AccountStatus`, session revocations, and audit events atomically. Roles and profile are retained; a repeat deactivation is a no-op.

## Migration

Preserve the applied `20261021174956_StaffProfiles` migration. Add a follow-up versioned migration that adds required `IsActive` with a true default for existing profiles. Keep the unique `UserId` index and restrictive PortalUser/Designation foreign keys. Apply through the existing schema-owner workflow; no new StaffProfile rows or account backfill are required.
