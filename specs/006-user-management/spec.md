# Feature Specification: User Management

**Feature Branch**: `feature/006-user-management`

**Created**: 2026-10-18

**Status**: Implemented, except the Coordinator setup-email producer (blocked on the batch-assignment owner) and the SC-002 usability study (not yet run); see tasks.md T011 and T031

**Input**: User description: "user management"

## Clarifications

### Session 2026-10-18

- Q: Which account types are in scope, given Teacher Management already owns teacher profiles? A: All supported portal roles are in scope: Admin, Student, Teacher, and Coordinator. User Management handles generic account identity/access; Teacher Management remains the owner of teacher-specific profile data and lifecycle.
- Q: What role changes should Admins be allowed to make across supported account types? A: A Student account has only the Student role. An active Teacher keeps Teacher; an Admin may assign or remove Admin for a Teacher. Coordinator remains derived from active batch-coordinator assignments and is not directly managed here. The default role must remain assigned.
- Q: How should a newly created Teacher set a password, and what happens if the setup email is not received? A: Keep the linked account pending until the Teacher sets a password using the secure setup email; an active Admin may request a fresh setup email while the account remains pending. Never expose the password or reuse an old setup link.
- Q: How should newly provisioned Admin, Student, and Coordinator accounts receive credentials? A: Use the same secure setup-email flow for all supported roles. Every new account remains pending until its user sets a password; an active Admin may request a fresh setup email while the account remains pending. The batch-coordinator assignment owner provisions Coordinator accounts and MUST request the common setup event after its account/assignment transaction commits; User Management does not create Coordinator accounts or assignments directly.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Find and Review Portal Accounts (Priority: P1)

As an active Admin, I need to find and review portal accounts so I can understand who has access and their current account state.

**Why this priority**: Account administration needs a reliable, authorized view of existing identities before access or lifecycle changes can be made.

**Independent Test**: As an active Admin, search a fixture set of portal accounts by email and inspect their current status and assigned roles; verify non-Admin and anonymous requests reveal no account data.

**Acceptance Scenarios**:

1. **Given** portal accounts exist, **When** an active Admin searches or filters the account list, **Then** only matching accounts appear in a bounded, usable result set.
2. **Given** an active Admin opens an account, **When** its detail is displayed, **Then** the account's stable identity, current status, and assigned roles are shown without credentials or secrets.
3. **Given** a non-Admin or anonymous visitor requests account data directly, **When** the request is authorized, **Then** access is denied without disclosing account information.

---

### User Story 2 - Provision a Portal Account (Priority: P1)

As an active Admin, I need to provision an account for an eligible person so the person can receive only the approved portal access.

**Why this priority**: Portal users need a controlled account before they can use role-specific features.

**Independent Test**: As an active Admin, create an account with a unique email and approved role, then verify one account is created, duplicate identity is rejected without partial changes, and credentials are not exposed to the Admin. Verify Teacher provisioning requests the common setup event through Teacher Management and a newly provisioned Coordinator requests it through the batch-assignment owner after its transaction commits; neither owner bypasses pending setup.

**Acceptance Scenarios**:

1. **Given** an active Admin submits valid account information for an in-scope account type, **When** provisioning succeeds, **Then** exactly one portal account is created with the approved role state.
2. **Given** the email already identifies a portal account, **When** an Admin attempts to create a duplicate, **Then** the request is rejected without creating a second account or partial role changes.
3. **Given** an account of any supported role is provisioned through its owning workflow, **When** provisioning commits, **Then** it remains pending and the same secure password-setup email flow is requested for its registered address; the Admin does not receive or set the password. The batch-assignment owner requests the event for a newly provisioned Coordinator account.
4. **Given** a pending account follows a valid setup email and sets a password that meets the existing password policy, **When** setup completes, **Then** the linked account becomes active and the user can sign in under its assigned roles.
5. **Given** a user reports not receiving the setup email while their account remains pending, **When** an active Admin requests reissue, **Then** the owning secure setup workflow creates a fresh link, the request is audited, and no password or proof is disclosed.

---

### User Story 3 - Maintain Account Access and Lifecycle (Priority: P1)

As an active Admin, I need to maintain permitted access and deactivate accounts when needed so account privileges reflect current authorization.

**Why this priority**: Incorrect access can expose protected portal data, while deactivated users must not retain active sessions.

**Independent Test**: As an active Admin, change an account's permitted access and deactivate a test account; verify role invariants, session revocation, audit history, and direct authorization behavior.

**Acceptance Scenarios**:

1. **Given** an active Admin makes an allowed access change, **When** it is saved, **Then** only permitted role assignments and a valid default role are stored and subsequent authorization reflects the change.
2. **Given** an Admin deactivates an account, **When** the change succeeds, **Then** future sign-in and existing sessions are denied while the account and audit history remain reviewable.
3. **Given** an access change would violate role policy, remove the last active Admin, or deactivate the caller's own account, **When** it is submitted, **Then** it is rejected without changing account state or assignments.
4. **Given** an account has related feature-specific profile data, **When** generic account access is changed, **Then** the owning feature's profile and business records remain intact.
5. **Given** an Admin changes a Teacher's roles, **When** the change is saved, **Then** Teacher remains assigned while the profile is active, Admin may be added or removed, and Coordinator changes only when the batch-coordinator assignment owner reports its authoritative state. A Student account remains Student-only.

### Edge Cases

- Two Admins change the same account or role assignments concurrently.
- An account email matches another portal identity after normalization.
- Self-deactivation by an Admin is rejected even when another active Admin exists; deactivation of the last active Admin is also rejected.
- A user has multiple assigned roles and an Admin changes or removes the current default role.
- A deactivated account has active sessions or references in audit and feature-specific history.
- Account creation succeeds but a required account or audit write fails.
- A requested role is unsupported or conflicts with a role derived by another feature.
- An account is in pending credential setup when an Admin attempts to change access or deactivate it.
- A setup email for any supported account role fails, expires, or is reissued while the account is pending; each reissue is Admin-only and uses a fresh owner-generated setup link.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The portal MUST allow sessions actively using the Admin role to list, search, filter, and inspect in-scope portal accounts. Search MUST support the registered email and stable account identifier; results MUST be bounded.
- **FR-002**: User Management MUST cover generic portal accounts with any supported role: Admin, Student, Teacher, and Coordinator. It MUST manage account identity and access without duplicating or independently changing feature-specific profile data; Teacher Management remains the owner of TeacherProfile fields and teacher-specific lifecycle.
- **FR-003**: For an in-scope account type, the portal MUST allow an active Admin to create one account using a unique normalized email and the minimum required identity data; duplicate identity MUST be rejected without partial account or role changes.
- **FR-004**: A Student account MUST have only the Student role. Teacher MUST remain assigned while a Teacher profile is active. An active Admin MAY assign or remove Admin from a Teacher account. Coordinator MUST be derived from active batch-coordinator assignments and MUST NOT be directly changed through User Management. Other feature-derived role assignments remain with their owning workflows. Every default role MUST remain among the account's assigned roles, and the last active Admin MUST NOT be removable or deactivatable.
- **FR-005**: The portal MUST allow an active Admin to update the registered email for an in-scope account without replacing its stable account identifier or violating normalized-email uniqueness. Email is the only generic account identity field editable through User Management; feature-specific profile fields remain with their owning management feature.
- **FR-006**: The portal MUST allow an active Admin to deactivate an in-scope account without physically deleting it, except that an Admin MUST NOT deactivate their own account. Deactivation MUST block sign-in, revoke all active sessions, and preserve account, audit, and feature history.
- **FR-007**: Every User Management operation MUST be authorized server-side against the session's active Admin role. Hiding controls in the user interface MUST NOT grant or enforce access.
- **FR-008**: Successful account creation, identity changes, role changes, and deactivation MUST produce allowlisted audit records; required account changes and audit records MUST commit atomically.
- **FR-009**: Every account provisioned for an in-scope role MUST remain pending until its user sets a password through the same owning secure setup-email flow; only then may the account become active. Each account-type owner MUST request the common setup event after its account transaction commits, including the batch-assignment owner for a newly provisioned Coordinator account. If a user reports not receiving the setup email while the account remains pending, only an active Admin MAY request a reissue through that workflow; each reissue MUST use a fresh setup link and produce a safe audit event. Delivery failure MUST leave the account pending. Admins MUST NOT set or receive user passwords, and credentials/proofs MUST NOT appear in audit records or logs. Profile-specific provisioning details remain with each account type's owning feature.
- **FR-010**: User Management MUST coexist with Teacher Management and other role-specific features without changing their profile ownership, derived-role synchronization, or business records.

### Key Entities *(include if feature involves data)*

- **Portal account**: A stable account identity with normalized email, account status, assigned roles, and default role.
- **Role assignment**: A supported access assignment attached to a portal account; some assignments may be required or derived by an owning feature.
- **Account credential setup**: One secure email flow that allows a pending account of any supported role to set a password and activates the linked account; an active Admin can request a fresh setup email while the account remains pending.
- **Account audit history**: Immutable actor, action, target, outcome, and approved change facts that do not contain passwords or proofs.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of duplicate normalized-email creation attempts are rejected without producing a second account or partial role assignment.
- **SC-002**: At least 9 of 10 representative Admins can find a specified account and complete an allowed maintenance action within two minutes without developer assistance.
- **SC-003**: 100% of anonymous and non-Admin direct account-management requests are denied without exposing account data.
- **SC-004**: 100% of successful account/access changes have corresponding audit history, and simulated audit failure leaves no partial business change.
- **SC-005**: 100% of deactivated account tests deny both new sign-in and requests using existing sessions while retaining account and audit history.
- **SC-006**: 100% of account setup tests verify that an Admin never receives or sets the user's password and that account activation follows the approved setup policy.

## Assumptions

- User Management is an active-Admin capability and reuses the existing portal identity, active-role authorization, and audit systems.
- The term "user" means a portal account with one or more supported roles, not every feature-specific profile or academic record.
- Teacher profile fields and teacher-specific lifecycle remain in Teacher Management; User Management works with the linked portal account only.
- Deactivation, rather than physical deletion, is the conservative default because account and audit references must remain reviewable.
- Student accounts remain Student-only; active Teacher accounts retain Teacher; Admin may be added or removed for a Teacher; Coordinator remains derived from batch-coordinator assignments. Other feature-derived roles remain with their owners.
- All supported roles use the same secure setup-email and password-activation flow. Admin reissue is limited to pending accounts and delegates fresh link creation to the owning credential workflow.
