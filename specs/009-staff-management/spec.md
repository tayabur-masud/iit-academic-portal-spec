# Feature Specification: Staff Management

**Feature Branch**: `feature/009-staff-management`

**Created**: 2026-10-21

**Status**: Ready for planning.

**Input**: User description: Admins create staff as they create teachers. Each staff account receives the Admin role and follows the existing account activation process. Staff information includes Name, Mobile number, and Designation; an Admin manually verifies it. My Profile is excluded.

## Clarifications

### Session 2026-10-21

- Q: Which portal accounts are considered staff, and what role must a staff account have? → A: Staff Management creates a new linked portal account and StaffProfile for a new email; the new account receives the existing Admin role as its default role. Do not add a separate Staff role.
- Q: What should Staff Management require to evidence manual verification before a profile change is saved? → A: The saving Admin explicitly confirms that the changed information was manually verified. Record the confirmation with the audit event; do not upload or retain source documents.
- Q: If the email entered for a new staff member already belongs to a portal account, what should Staff Management do? → A: If it belongs to an active Admin account with no StaffProfile, attach a StaffProfile to that account without changing credentials/status or requesting setup again. Reject a deactivated or non-Admin account and reject an Admin account that already has a StaffProfile; never create a duplicate account.
- Q: Should Staff Management support Teacher Management-style deactivation, including disabling the linked Admin account and revoking its sessions, or should account/profile deactivation remain outside this feature? → A: Deactivate the StaffProfile and linked Admin account together, revoke active sessions, preserve profile/audit history, and retain the existing safeguards against self-deactivation or deactivating the last active Admin.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Maintain Staff Profile Details (Priority: P1)

As an active Admin, I need to create a staff member with an Admin portal account and maintain their manually verified Name, Mobile number, and Designation so the staff directory and access are accurate and traceable.

**Why this priority**: Staff need a linked portal account with the correct role and verified profile details before they can access Admin functions.

**Independent Test**: As an active Admin, create a staff member with a new email and verify one linked Admin portal account and StaffProfile are created atomically and the account remains pending until secure setup completes. Also attach a profile to an existing active Admin account with no profile, preserving that account's credentials/status and not requesting setup again. Verify manual-verification confirmation is audited and My Profile remains view-only.

**Acceptance Scenarios**:

1. **Given** an active Admin has manually verified a new staff member's Name, Mobile number, and Designation, **When** the Admin creates the staff member with a unique email and explicit verification confirmation, **Then** one StaffProfile and one linked portal account are created atomically, the account receives the Admin role as its default, and it remains pending until the existing secure credential-setup process activates it.
2. **Given** an existing staff member's information has been manually verified, **When** an active Admin updates the Name, Mobile number, or Designation and confirms verification, **Then** the profile change and verification audit are saved atomically without changing the linked account identity or roles.
3. **Given** an active Admin updates their own staff information, **When** the update is entered through Staff Management after manual verification and the Admin confirms verification, **Then** it follows the same validation and audit behavior as an update to another staff member; no write action is offered from My Profile.
4. **Given** an Admin selects a Designation, **When** the selected catalog entry is active and exists, **Then** the staff profile references that Designation and displays its current catalog name.
5. **Given** a non-Admin or anonymous user attempts to list, view, create, or change staff information, **When** the server authorizes the request, **Then** access is denied without exposing staff personal information.
6. **Given** an Admin submits an invalid, missing, or inactive Designation or omits a required profile field, **When** the change is validated, **Then** the request is rejected with a field-level explanation and no profile or audit change is committed.
7. **Given** a staff profile change is committed, **When** an Admin reviews the audit history, **Then** the responsible Admin, target staff account, time, and changed fields are recorded.
8. **Given** an Admin submits a profile change without confirming manual verification, **When** the change is validated, **Then** the save is rejected and neither the profile nor its audit history changes.
9. **Given** the email belongs to an existing active Admin account with no StaffProfile, **When** an Admin creates a staff profile for that email, **Then** a profile is attached to the existing account without changing its role, credentials, status, or setup-email state.
10. **Given** an active Admin deactivates a staff member, **When** the deactivation is allowed, **Then** the StaffProfile is marked inactive, the linked account is disabled, active sessions are revoked, and profile/audit history is preserved. A request that would deactivate the acting Admin or remove the last active Admin is rejected without changes.

### Edge Cases

- An Admin submits an email belonging to an existing Admin account with no StaffProfile; attach the profile without creating a duplicate account. Reject an email belonging to a non-Admin or deactivated account, and reject an account that already has a StaffProfile.
- An Admin attempts to create a second staff profile for a portal account that already has one.
- Account, StaffProfile, or required audit persistence fails during initial provisioning; none of the account/profile changes may be left partially committed.
- Credential setup is not delivered, expires, or fails; the staff account remains pending and cannot sign in until the existing setup flow succeeds.
- An Admin attempts to deactivate their own staff account or the last active Admin; reject the request without deactivating the profile/account or revoking its sessions.
- An Admin enters a blank Name or Mobile number, or submits a Mobile number exceeding the supported field limit.
- An Admin selects a Designation that does not exist, has been deleted, or is inactive.
- The target portal account is missing, deactivated, or no longer has the Admin role when a staff profile change is submitted.
- An Admin updates their own staff information; the change must use the same Admin workflow, validation, and audit as updates to others.
- An Admin changes staff information without manually verifying the source; the system must not imply that the information was verified by the application or store an unverified proof.
- An Admin attempts to save a staff profile change without explicitly confirming manual verification; reject the change without updating the profile or adding a successful-change audit event.
- Audit persistence fails during a staff profile update; the profile change must not partially commit.
- An unauthenticated or non-Admin user directly calls a staff management operation.
- A staff member attempts to change their information through the view-only My Profile area.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The portal MUST allow an active Admin to view and search staff profile records linked to portal accounts. Results MUST be bounded and MUST NOT expose passwords, credential-setup proofs, or unrelated account secrets.
- **FR-002**: Each staff profile MUST contain a required Name, required Mobile number, and required Designation reference. Mobile numbers are retained as contact information and MUST NOT be represented as verified by the portal unless a separate approved verification feature exists.
- **FR-003**: Designation MUST reference an active entry in the existing IIT Designations catalog. The profile MUST NOT duplicate designation text as an independently maintained value; the displayed designation name comes from the referenced catalog entry.
- **FR-004**: Each staff profile MUST link to exactly one portal account, and each portal account MUST have at most one staff profile. When an Admin creates staff with a new email, Staff Management MUST create the linked portal account and profile atomically, assign only the Admin role, and set Admin as the default role. When the email belongs to an existing active Admin account with no StaffProfile, Staff Management MUST attach a profile to that account without changing its role, credentials, status, or setup-email state. A deactivated/non-Admin account or an account with an existing StaffProfile MUST be rejected without partial changes.
- **FR-005**: Staff Management MUST NOT create a Staff role or provide role editing. Each newly created staff account receives the existing Admin role and default role; Designation values MUST NOT grant permissions. User Management remains the owner of later account role changes.
- **FR-006**: Only a session actively using the Admin role MUST be allowed to list, view, create, or change staff profile data. The server MUST enforce this policy on every request regardless of interface visibility.
- **FR-007**: Before a staff profile create or update is committed, the acting Admin MUST explicitly confirm that the submitted or changed information was manually verified against the institution's accepted source. The server MUST reject a change without this confirmation. The confirmation MUST be recorded with the audit event; the feature MUST NOT claim independent application verification or require storage of sensitive source documents.
- **FR-008**: An Admin MUST be able to update their own staff record through the same Staff Management workflow used to update other staff records, subject to the same manual verification, validation, authorization, and audit rules. The My Profile menu/page MUST remain view-only and MUST NOT provide profile editing in this feature.
- **FR-009**: Successful staff profile creates and updates MUST be audited with the acting Admin, target account/profile, action, UTC time, outcome, allowlisted changed fields, and the Admin's manual-verification confirmation. For a new email, portal account creation, StaffProfile creation, and required audit records MUST commit atomically. Attaching a profile to an existing Admin account MUST atomically create only the profile and its audit event. Failed validation, missing confirmation, or audit persistence MUST leave no partial account/profile change.
- **FR-010**: A newly created staff account MUST have no password and MUST start pending; after the account/profile transaction commits, Staff Management MUST request the existing secure credential-setup flow. The account MUST remain unable to sign in until setup succeeds, and setup or delivery failure MUST leave it pending. Attaching a profile to an existing Admin account MUST NOT change credentials/status or request setup again. Staff Management MUST NOT set or reveal credentials or edit roles. Except for staff deactivation under FR-012, User Management remains the owner of account status and lifecycle.
- **FR-011**: Staff Management MUST NOT edit StudentProfile or TeacherProfile data; Student Management and Teacher Management remain the owners of those role-specific profiles.
- **FR-012**: An active Admin MUST be able to deactivate a StaffProfile. Deactivation MUST mark the profile inactive, deactivate its linked Admin account, revoke that account's active sessions, preserve the profile and audit history, and commit the profile/account/audit changes atomically. The portal MUST reject an Admin's attempt to deactivate their own account or to deactivate the last active Admin account without changing profile, account, or session state.

### Key Entities *(include if feature involves data)*

- **Staff profile**: A staff member's Name, Mobile number, Designation reference, and active/inactive state, linked one-to-one to a portal account created or linked during Staff Management provisioning and maintained by Admins after manual verification. Deactivation disables the linked account and preserves profile/audit history.
- **Portal account**: The authentication identity linked to the StaffProfile. Staff Management creates a new account as pending with the Admin role/default role; User Management remains the owner of credentials, activation, status, and later role changes.
- **Designation**: An existing IIT Designations catalog entry referenced by the staff profile.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of successful staff creations produce exactly one StaffProfile linked to an Admin-role account, with a non-empty Name, Mobile number, and valid active Designation reference; a new email creates exactly one pending account, while an existing eligible Admin account is reused without account or credential changes.
- **SC-002**: 100% of repeated StaffProfile creation attempts for one account, and attempts to use an existing non-Admin or deactivated account, are rejected without duplicate accounts or partial profiles.
- **SC-003**: 100% of non-Admin direct staff-management requests are denied without disclosing staff personal information.
- **SC-004**: 100% of successful staff profile updates require an explicit manual-verification confirmation and record the acting Admin, target staff account, timestamp, confirmation, and allowlisted field changes atomically.
- **SC-005**: 100% of invalid or inactive Designation references and missing required profile values are rejected without changing staff profile or audit data.
- **SC-006**: The My Profile menu/page provides no staff-profile edit capability; an Admin's own profile can be changed only through the same verified, audited Staff Management workflow used for other staff.
- **SC-007**: 100% of newly provisioned staff accounts remain pending and unable to sign in until successful completion of the existing secure credential-setup flow; setup delivery failure does not activate the account or leave a partial profile.
- **SC-008**: 100% of permitted staff deactivations disable the linked account, revoke active sessions, and preserve profile/audit history; self-deactivation and deactivation of the last active Admin are rejected without changes.

## Assumptions

- Staff Management creates a new Admin-role portal account and StaffProfile together, following the Teacher Management provisioning pattern. User Management owns account identity, roles, credentials, and ordinary lifecycle operations; Staff Management invokes the established deactivation safeguards when deactivating staff.
- Name, Mobile number, and Designation are required staff profile values; mobile format is validated using the portal's existing conventions, with no new phone-verification flow.
- Designation values are maintained in Configuration; no separate designation editor or new designation list is created here.
- Manual verification is performed by an Admin before save. The Admin explicitly confirms that verification in the workflow; the application records this confirmation with the change audit but does not store identity documents or claim that it independently verified the source.
- Admins can update their own staff record through Staff Management after manual verification; My Profile remains view-only and out of scope.
- Student and Teacher profile data remain managed by Student Management and Teacher Management respectively.
- Every staff profile represents an Admin-role staff account. Newly created accounts use Admin as their default role. No Super Admin role or separate Staff role is introduced.

## Out of Scope

- My Profile self-service editing or profile display page.
- Changing account email after provisioning or assigning/removing portal roles.
- Student or Teacher profile fields and lifecycle.
- Designation catalog creation or maintenance.
- Uploading or retaining identity documents as verification evidence.
