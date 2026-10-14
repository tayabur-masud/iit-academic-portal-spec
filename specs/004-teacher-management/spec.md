# Feature Specification: Teacher Management

**Feature Branch**: `feature/004-teacher-management`

**Created**: 2026-10-14

**Status**: Implemented (email delivery and the batch-coordinator assignment producer remain deferred, as specified)

**Input**: User description: Admin can add, update, and delete teacher information. Each teacher becomes a user with the Teacher role. Admin can modify the role.

## Clarifications

### Session 2026-10-14

- Q: Which portal roles may an Admin assign to or remove from a teacher account, and may Admin be granted through this workflow? → A: Teacher remains assigned while the profile is active; Admin may assign or remove Admin, Student is never assigned, and Coordinator is synchronized automatically from batch-coordinator assignments. The default role must be among assigned roles.
- Q: What should Delete do to the teacher profile and linked user account? → A: Deactivate the profile and disable the linked account, preserve profile/audit history, and revoke all active sessions; physical deletion is not part of this feature.
- Q: How should a newly created teacher receive initial credentials? → A: Once the email mechanism is ready, the portal emails password-setup instructions using the existing secure credential setup/recovery flow; the Admin does not set or receive the password.
- Q: How should teacher account creation behave if the password-setup email cannot be delivered? → A: During this feature, no delivery is attempted and the profile/account remains pending and unable to sign in; delivery failure and retry behavior are deferred until the email mechanism is ready.
- Q: What is in scope while the email mechanism is unavailable? → A: Create the teacher profile and linked account in a pending, non-sign-in state without attempting email delivery; defer email integration and its delivery tests until the email mechanism is ready.
- Q: Should the system block a role change or teacher deactivation that would leave the portal without any active Admin accounts? → A: Yes. Preserve at least one active Admin; reject a role change or deactivation that would remove the last active Admin.
- Q: Which teacher details, beyond display name and linked email, should Admins manage in this feature? → A: Employee/staff ID, phone/contact, and Designation are optional; a supplied employee/staff ID is unique across teacher profiles.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Add a Teacher (Priority: P1)

As an Admin, I need to add a teacher and provision a corresponding portal user so the teacher can access the portal with the Teacher role.

**Why this priority**: Teacher records and an authorized account are prerequisites for teacher-facing portal features.

**Independent Test**: As an active Admin, create a teacher with valid required profile information; verify one linked user account with the Teacher role exists, duplicate identity information is rejected, and the account remains pending without sign-in access until credential setup is available. Email delivery and email-specific tests are deferred until the shared email mechanism is ready.

**Acceptance Scenarios**:

Scenarios 4-5 define portal-wide behavior. This feature delivers and tests the role synchronizer, but the assignment-trigger integration is a completion gate for the future batch-assignment owner because no producer currently exists.

1. **Given** an Admin submits valid teacher information with an email not already assigned to another user, **When** the teacher is created, **Then** one teacher profile and one linked user account are created, the user receives the Teacher role, and Teacher is the default role.
2. **Given** an Admin submits an email already used by a portal account, **When** the request is validated, **Then** creation is rejected with a field-level message and no duplicate profile or account is created.
3. **Given** an Admin creates a teacher account while the email mechanism is unavailable, **When** provisioning completes, **Then** the linked account remains pending and unable to sign in; this feature does not attempt email delivery or provide an alternate password handoff.

---

### User Story 2 - Maintain Teacher Information and Roles (Priority: P1)

As an Admin, I need to view and update teacher information and manage permitted role assignments so that the directory and access remain accurate.

**Why this priority**: Teacher contact and account information must stay current, and access changes must be administered safely.

**Independent Test**: As an active Admin, view, search, and update teacher profile fields, then modify role assignments within the approved policy; verify validation, active-role behavior, and audit history.

**Acceptance Scenarios**:

1. **Given** an Admin updates a teacher's profile, **When** all values are valid, **Then** the change is saved and the linked user identity remains the same.
2. **Given** an Admin changes a teacher's role assignment, **When** the change is allowed by the approved role policy, **Then** only the permitted assignments and default role are changed and existing authorization rules apply on subsequent requests.
3. **Given** a role change would violate the approved role policy or leave an invalid default role, **When** it is submitted, **Then** the change is rejected without altering the current assignments.
4. **Given** an active teacher is assigned as coordinator of a batch, **When** the batch-coordinator assignment becomes active, **Then** the Coordinator role is added automatically; it is removed when the teacher has no remaining active batch-coordinator assignments.
5. **Given** a teacher is removed as coordinator from one batch, **When** the assignment ends, **Then** Coordinator remains assigned if the teacher coordinates another active batch and is revoked if no other active batch-coordinator assignment remains.
6. **Given** a teacher account is the last active Admin account, **When** an Admin tries to remove its Admin role or deactivate it, **Then** the operation is rejected and the active Admin account remains available.

---

### User Story 3 - Find and Deactivate a Teacher (Priority: P1)

As an Admin, I need to find and deactivate a teacher when appropriate so that the directory reflects current staff while preserving profile and audit history.

**Why this priority**: Administrators need to maintain an accurate directory and prevent former teachers from retaining inappropriate access.

**Independent Test**: Search and filter teachers, deactivate a teacher, then verify directory status, disabled account access, revoked sessions, dependent references, and audit history.

**Acceptance Scenarios**:

1. **Given** an Admin searches or filters the teacher directory, **When** matching teachers exist, **Then** results are clear, paginated when needed, and support viewing each teacher's current profile and roles.
2. **Given** an Admin deactivates a teacher, **When** the operation succeeds, **Then** the linked account is disabled, active sessions are revoked, and the profile and audit history are preserved.
3. **Given** a non-Admin or unauthenticated user attempts teacher-management operations, **When** the service authorizes the request, **Then** the operation is denied regardless of menu visibility.

### Edge Cases

- A teacher email matches an existing portal account that is not currently linked to a teacher profile.
- Two Admins create or edit the same teacher concurrently.
- An optional employee/staff ID is omitted, blank, or duplicates another teacher's ID after trimming and case normalization.
- An Admin tries to remove the required Teacher role while the teacher profile remains active.
- An Admin attempts to assign Student to a teacher; the operation is rejected.
- A teacher has multiple batch-coordinator assignments; removing one must not remove Coordinator while another remains.
- A teacher's final active batch-coordinator assignment ends; Coordinator is removed automatically.
- The default role is removed from the account's allowed assignments.
- A role change or deactivation would remove the last active Admin account; reject it without changing the assignment or account status.
- A teacher is referenced by records in future teaching, course, or assessment features when deactivation is requested.
- Account creation succeeds but profile persistence or a required audit event fails.
- The email mechanism is unavailable during this feature; the created account remains pending and cannot sign in until credential setup is implemented in follow-up email work.
- A teacher account has active sessions when its role assignment or status changes.

## Requirements *(mandatory)*

### Business Rules

- Teacher management is an Admin capability. Every operation is authorized server-side against the session's active Admin role; assigned roles or visible UI do not grant access.
- Creating a teacher creates one linked portal user account, assigns the Teacher role, and sets Teacher as the default role. A teacher profile is linked to exactly one user; a user may have at most one teacher profile.
- A teacher uses the portal's existing email/password authentication. During this feature, the linked account is created pending and cannot sign in until credential setup becomes available. The Admin does not set or receive the password, and secrets are never written to logs/audit data.
- Email transport, setup-instruction delivery, retry/resend behavior, and email-specific end-to-end tests are deferred until the shared email mechanism is ready; this feature does not attempt delivery or provide an alternate credential handoff.
- The linked user identity is stable across teacher profile updates. Email changes follow the existing account identity rules and remain unique across users.
- Teacher remains assigned while the teacher profile is active. Admin may assign or remove Admin; Student is never assigned to a teacher. Coordinator is system-managed: it is present while the teacher has at least one active batch-coordinator assignment and removed when the last such assignment ends. Batch assignment workflows remain outside Teacher Management but MUST trigger this role synchronization. The default role must always be among assigned roles.
- The portal MUST always retain at least one active Admin account; a role change or deactivation that would remove the last active Admin is rejected.
- Deactivation disables the linked account and revokes all active sessions while preserving the teacher profile and audit history. Physical deletion is out of scope.
- Successful create, update, role-change, and deactivation operations use the existing audit system. Required audit and business changes commit atomically; failures do not leave partial profile/account changes.
- Teacher profiles may use the existing Designation catalog. This feature does not add course assignment, assessment, student, or academic workflow data.

### Functional Requirements

- **FR-001**: The portal MUST allow an active Admin to create a teacher profile linked to one portal user account, assign the Teacher role, and set Teacher as the default role.
- **FR-002**: A teacher profile MUST include a required display name and the linked user's required, unique email. Employee/staff ID, phone/contact details, and Designation are optional. A supplied employee/staff ID MUST be unique across teacher profiles after trimming and case-insensitive normalization.
- **FR-003**: The portal MUST allow active Admins to view, list, search, and filter teacher profiles and their permitted role assignments.
- **FR-004**: The portal MUST allow active Admins to update teacher profile information while preserving the linked user identity and enforcing required-field, email-uniqueness, and optional employee/staff-ID uniqueness rules.
- **FR-005**: The portal MUST keep Teacher assigned while a teacher profile is active, allow Admins to assign or remove the Admin role, and MUST NOT assign Student to a teacher. The account's default role MUST remain one of its assigned roles.
- **FR-006**: The portal MUST allow Admins to deactivate a teacher profile. Deactivation MUST disable the linked account and revoke all active sessions while preserving the profile and audit history; physical deletion MUST NOT be offered.
- **FR-007**: All teacher-management operations MUST be denied to sessions not actively using the Admin role, including direct requests that bypass the user interface.
- **FR-008**: Successful teacher creation, profile changes, role changes, and deactivation MUST produce allowlisted audit events recording actor, action, teacher/user identifiers, UTC time, outcome, and approved change summary.
- **FR-009**: Teacher identity/profile changes and required audit events MUST be atomic; failed validation or audit persistence MUST NOT leave partially created or changed accounts/profiles.
- **FR-010**: The Admin module MUST provide a Teacher Management entry and accessible directory/profile workflows with loading, empty, error, validation, and inactive states.
- **FR-011**: Teacher creation MUST create the linked user account in a pending state that cannot sign in until secure credential setup is available. The Admin MUST NOT set or receive the teacher's password. The eventual setup flow MUST enforce the existing minimum password policy (at least 8 characters, including at least one letter and one number).
- **FR-012**: Email transport, password-setup instruction delivery, retry/resend behavior, and email-related end-to-end tests MUST be deferred until the shared email mechanism is ready. Teacher creation in this feature MUST NOT attempt email delivery or provide an alternate password handoff.
- **FR-013**: When another portal workflow creates, changes, or ends a batch-coordinator assignment for a teacher, the portal MUST synchronize the Coordinator role to whether at least one active assignment remains. Admins MUST NOT manage this derived role independently of those assignments. The assignment-owning workflow MUST call the internal synchronization contract in the same unit of work as each assignment change and verify the producer integration before claiming this portal-wide requirement is complete.
- **FR-014**: The portal MUST reject an Admin-role removal or teacher deactivation that would leave no active Admin accounts; the assignment and account status MUST remain unchanged.

### Acceptance Criteria

- **AC-001 - Provisioning**: Every successful teacher creation produces exactly one linked user account with the Teacher role as its default; duplicate email does not create a second user or profile. (FR-001–FR-002)
- **AC-002 - Profile maintenance**: Admins can find and update teacher profile information while preserving identity linkage and enforcing uniqueness. (FR-002–FR-004)
- **AC-003 - Role safety**: Teacher remains assigned while active; Admin may assign/remove Admin; Student cannot be assigned; Coordinator is synchronized to active batch-coordinator assignments; the default role remains assigned. (FR-005, FR-012)
- **AC-004 - Deactivation safety**: Deactivation disables account access, revokes active sessions, preserves profile/audit history, and does not physically delete the user. (FR-006, FR-008–FR-009)
- **AC-005 - Authorization and audit**: 100% of non-Admin or anonymous direct management requests are denied; successful changes are audited atomically. (FR-007–FR-009)
- **AC-006 - Admin usability**: Admins can find and maintain teacher records through accessible, responsive directory and profile workflows. (FR-003, FR-010)
- **AC-007 - Initial credentials**: Every newly created teacher account remains pending and unable to sign in until secure credential setup is available; no password is set or received by the Admin. (FR-011)
- **AC-008 - Coordinator synchronization**: Given authoritative post-change assignment state, the synchronizer ensures Coordinator is assigned when at least one active assignment remains and removes it when none remain. The assignment-owning workflow's end-to-end trigger is accepted only after it proves the call and assignment mutation share one unit of work. (FR-013)
- **AC-009 - Last Admin protection**: Attempts to remove the Admin role from or deactivate the last active Admin account are denied without changing role assignments or account status. (FR-014)

## Key Entities *(include if feature involves data)*

- **Teacher profile**: Required display name and linked email, optional employee/staff ID unique across teacher profiles when supplied, optional phone/contact and Designation, and one-to-one link with a portal user.
- **Portal user account**: Authentication identity with unique email, assigned roles, default role, and account status.
- **Role assignment**: Relationship between a portal user and a supported portal role. Teacher is required for an active teacher profile; Admin is Admin-managed, Student is prohibited, and Coordinator is derived from batch-coordinator assignments.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of valid teacher creates produce exactly one linked account with Teacher role; 100% of duplicate-email attempts are rejected without duplicate records.
- **SC-002**: At least 95% of representative Admin users can find and update a specified teacher within two minutes without assistance.
- **SC-003**: 100% of unauthorized direct teacher-management requests are denied without exposing teacher or account data.
- **SC-004**: 100% of successful create, update, role-change, and deactivation operations produce the required audit history atomically.
- **SC-005**: 100% of role changes keep Teacher assigned while active, never assign Student, allow only Admin-authorized Admin changes, and preserve a valid default role.
- **SC-006**: 100% of newly created teacher accounts remain pending and unable to sign in until secure credential setup is available; no password is exposed to the Admin.
- **SC-007**: 100% of calls to the Coordinator synchronizer reconcile the role to the supplied authoritative state: add/retain Coordinator when an active assignment exists and remove it otherwise. The assignment-producer integration is separately gated on the owning feature's end-to-end transaction tests.
- **SC-008**: 100% of role changes or deactivations that would leave no active Admin accounts are rejected without changing account access.

## Assumptions

- Existing portal authentication, supported role names, active-role checks, email delivery, and audit facilities are reused.
- Teacher Management is accessible only from the Admin module and does not create course/assessment assignments.
- Teacher profile fields include required display name and linked email; employee/staff ID, contact number, and Designation are optional. A supplied employee/staff ID is unique across teacher profiles after trimming and case-insensitive normalization.
- The teacher email mechanism is not ready; secure invitation delivery and its tests will be completed as follow-up work. Until then, created accounts remain pending and cannot sign in.
- No batch-coordinator assignment producer exists in the current service. Teacher Management supplies and tests the synchronizer only; the future assignment-owning feature must invoke it before committing every assignment mutation and prove the end-to-end integration before the portal-wide behavior in FR-013 is considered complete.
