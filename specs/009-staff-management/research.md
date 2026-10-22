# Research: Staff Management

**Feature**: [Staff Management](spec.md)  
**Plan**: [Implementation plan](plan.md)  
**Research date**: 2026-10-21

## Findings and Decisions

### Staff identity and account ownership

- **Finding**: `PortalUser` owns account identity, login email, account status, and assigned roles. User Management manages generic account identity, credentials, and subsequent role changes. `TeacherProvisioningService` creates a pending PortalUser and TeacherProfile in one transaction, then requests secure credential setup after commit; `TeacherLifecycleService` deactivates profile/account together, revokes sessions, and protects the last active Admin. `UserProvisioningService` commits an account-only transaction, so it must not be called separately from StaffProfile creation if atomicity is required.
- **Decision**: Staff Management creates one `StaffProfile` for a new staff email, creating a linked PortalUser in the same transaction with only the Admin role and Admin as default. The account starts `PendingCredentialSetup` with no password; after commit, request the existing secure setup flow. If the email matches an existing active Admin account with no StaffProfile, attach a profile without changing role, credentials, status, or setup-email state. Reject an existing non-Admin, deactivated account, or account already linked to a StaffProfile. Never create duplicate portal accounts. User Management remains the owner of later email and role changes; Staff Management owns the staff provisioning operation and its coupled profile.
- **Rationale**: This follows the user-approved Teacher-style provisioning and activation flow while assigning the required Admin role. Existing Admin accounts remain reusable without credential resets. Creating the account and profile in one unit of work prevents partial provisioning; setup notification after commit keeps delivery failure from rolling back the account/profile or activating it prematurely.
- **Alternatives considered**: Creating a generic account first through User Management and attaching a profile later was rejected because it creates an orphanable partial workflow. Rejecting every existing email was rejected because an existing active Admin account without a profile must be linkable. A separate Staff role was rejected by the user. Reusing `TeacherProfile` was rejected because Teacher Management owns teacher data and lifecycle.

### Designation catalog

- **Finding**: Configuration already provides an independent Designations catalog; TeacherProfile optionally references it with restrictive deletion behavior.
- **Decision**: `StaffProfile.DesignationId` is required and references the existing active Designations entry. Validate that a newly selected designation exists and is active. Display the current catalog name from the reference; do not copy editable designation text into StaffProfile. A designation grants no permissions.
- **Rationale**: One catalog prevents divergent designation names across staff and teachers and follows existing Configuration ownership.
- **Alternatives considered**: Free-text or copied designation names were rejected because Designations are already centrally maintained. Creating a Staff-specific designation list was rejected as duplication.

### Manual verification and audit

- **Decision**: Before every staff profile create or update, require the acting Admin to explicitly confirm that the submitted/changed information was manually verified against the institution's accepted source. Persist the confirmation with the corresponding allowlisted business audit event in the same transaction as the profile change. Do not store an enduring `IsVerified` state, scan, document, or proof.
- **Rationale**: Verification is a human action; a confirmation records the Admin's attestation without claiming that the application verified the source. Atomic audit provides an accountable record of who made which change and confirmed verification.
- **Alternatives considered**: Automatically marking information verified from field presence was rejected because it misrepresents a human check. Requiring a second Admin approval was not requested. Uploading evidence was rejected because the user specified manual verification but did not request document retention; the physical source remains outside the application.

### Admin self-update and My Profile

- **Decision**: An Admin may update their own StaffProfile through the same Staff Management workflow, after the same manual-verification confirmation and with the same validation/audit behavior as updates to other staff. The My Profile menu/page remains view-only and is outside this feature.
- **Rationale**: The user explicitly permits Admin self-edit but requires it to use the ordinary Admin management flow rather than a self-service profile page.
- **Alternatives considered**: A separate self-service edit path was rejected because it would bypass the specified verified management workflow. Blocking self-update was rejected by the user's explicit direction.

### Staff deactivation

- **Decision**: Deactivate a StaffProfile and its linked Admin account together. Mark the profile inactive, mark the account deactivated, revoke active sessions, and preserve both records and their audit history in one transaction. Reuse the Teacher/User Management last-active-Admin lock and reject deactivation of the acting Admin. A repeated deactivation is a no-op; reactivation is not offered by Staff Management.
- **Rationale**: The user approved Teacher Management-style deactivation. Applying the existing last-Admin and self-deactivation rules prevents Staff Management from bypassing portal-wide account safety.
- **Alternatives considered**: Deactivating only the profile while leaving its Admin account enabled would allow a former staff member to retain privileged access. Physical deletion or reactivation was not requested and would undermine the existing lifecycle/history model.

### Storage, validation, and UI conventions

- **Decision**: Keep the already-applied `StaffProfiles` migration intact and add a follow-up migration for `StaffProfile.IsActive`; retain unique restrictive `UserId` and required restrictive `DesignationId`. Require Name (maximum 200), Mobile number (maximum 50), active Designation on create or reassignment, and a true `manualVerificationConfirmed` on every create/update. Reuse the existing Angular Admin feature tree, REST DTOs, active-Admin authorization, field-keyed validation, audit, account-provisioning, notification, and session-revocation patterns. Store Mobile on StaffProfile, not `PortalUser.PhoneNumber`.
- **Rationale**: A distinct StaffProfile isolates role-specific profile data while existing Identity and Configuration records remain authoritative for account and Designation data. A follow-up migration avoids rewriting a migration already applied in Development. Established Teacher and User Management services supply transaction, credential setup, deactivation, and last-Admin patterns.
- **Alternatives considered**: Changing the applied `StaffProfiles` migration was rejected. Synchronizing Mobile to Identity `PhoneNumber` was rejected because that field has generic authentication/contact semantics. A new service, package, or frontend framework is not justified.

## Repository Findings

- `PortalUser` extends ASP.NET Core Identity and contains the account's `DefaultRole` and `AccountStatus`; Identity supplies the account email and phone field.
- User Management owns generic account identity, roles, and credentials. Staff Management must provision new Admin accounts together with profiles and may attach an existing active Admin account with no StaffProfile.
- `TeacherProfile` is an audited one-to-one account profile with DisplayName, optional Phone, and optional DesignationId; Teacher Management owns its lifecycle and role rules.
- The shared Configuration `Designation` entity already exists and is used by TeacherProfile.
- No StaffProfile or Admin-specific profile entity was found in the current service source.
- The existing Admin shell has a disabled My Profile action; implementing a My Profile page is out of scope here.
- Existing audit recording, active-Admin authorization, EF migrations, Angular reactive forms, xUnit/Vitest tests, secure credential setup, session revocation, and last-Admin protection are available for reuse.

## External Inputs

No external service or owner is required. The Admin is responsible for manual verification before saving and attests to that check in the workflow. Source documents remain outside the application.
