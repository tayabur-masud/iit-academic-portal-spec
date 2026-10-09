# Feature Specification: Role-Based Authentication and Access Control

**Feature Branch**: `feature/001-role-based-auth` (all three repositories)

**Created**: 2026-10-08

**Status**: Draft

**Input**: User description: Define login, logout, password recovery, role-based access control, multiple assigned roles, and role switching for the IIT Academic Portal.

## Clarifications

### Session 2026-10-09

- Q: Do you mean a 24-hour inactivity timeout or a 24-hour absolute maximum from sign-in? → A: Authenticated sessions expire after three hours without authenticated activity; each authenticated request refreshes the inactivity window.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sign In and Sign Out (Priority: P1)

A Student, Teacher, Coordinator, or Admin signs in to the portal using an existing account and reaches the experience for an assigned role. When finished, the user signs out and protected information is no longer available in that session.

**Why this priority**: Sign-in and sign-out establish the authenticated boundary for every protected portal experience.

**Independent Test**: Verify successful and unsuccessful sign-in, loading and validation feedback, sign-out, and rejection of protected access after sign-out.

**Acceptance Scenarios**:

1. **Given** an eligible account with one assigned role, **When** the user submits valid credentials, **Then** the portal establishes an authenticated session and shows the experience for that role.
2. **Given** an eligible account with multiple assigned roles, **When** the user submits valid credentials, **Then** the portal offers only that user's assigned roles and requires an active role before showing role-specific content.
3. **Given** invalid credentials, **When** the user submits the sign-in form, **Then** the portal denies authentication, creates no authenticated session, and shows a generic actionable error.
4. **Given** an authenticated user, **When** the user signs out, **Then** the current session ends, the user is returned to the unauthenticated experience, and protected information cannot be accessed through that session.

---

### User Story 2 - Access Only Authorized Areas (Priority: P1)

An authenticated user can access portal capabilities only when the user's assigned roles grant the required authorization. The experience communicates denied access without revealing protected information.

**Why this priority**: Role separation protects academic and administrative information and prevents unauthorized actions.

**Independent Test**: For each approved role-to-capability rule, test an allowed and denied request using users with matching and non-matching assignments.

**Acceptance Scenarios**:

1. **Given** an authenticated user whose assigned role is authorized for a capability, **When** the user requests it, **Then** the portal permits the action.
2. **Given** an authenticated user without a role authorized for a capability, **When** the user requests it directly or through a visible control, **Then** the portal denies the action and does not disclose protected information.
3. **Given** a user whose role assignment is removed or no longer valid, **When** the user next requests a protected capability, **Then** the portal denies capabilities that are no longer authorized.

---

### User Story 3 - Recover and Reset a Password (Priority: P2)

A user who cannot sign in can request password recovery and, after proving control of an approved recovery method, set a replacement password.

**Why this priority**: Recovery restores access for legitimate users while limiting account discovery and credential compromise.

**Independent Test**: Exercise recovery for a recognized account, an unrecognized account, an invalid or expired recovery proof, and a successful reset.

**Acceptance Scenarios**:

1. **Given** a user on the recovery screen, **When** the user submits an account identifier, **Then** the portal shows the same non-enumerating confirmation regardless of whether the account exists.
2. **Given** a recognized eligible account with a verified recovery method, **When** recovery is requested, **Then** the portal provides recovery instructions through the approved method without exposing credentials.
3. **Given** a valid recovery proof, **When** the user submits a new password that satisfies the approved password policy, **Then** the portal updates the credential and confirms completion.
4. **Given** an invalid, expired, or already-used recovery proof, **When** it is submitted, **Then** the portal rejects the reset and leaves the credential unchanged.

---

### User Story 4 - Select and Switch Assigned Roles (Priority: P1)

A user assigned more than one role can choose an active role after signing in and switch among assigned roles while authenticated. The active role is clear, and switching does not grant roles the user was not assigned.

**Why this priority**: Multi-role users need access to each authorized work context without managing separate accounts.

**Independent Test**: Sign in with a single-role account and a multi-role account; verify the role choice, active-role indication, switching, and denial for unassigned roles.

**Acceptance Scenarios**:

1. **Given** a user with one assigned role, **When** the user signs in, **Then** the portal enters that role's context without offering an unassigned role choice.
2. **Given** a user with two or more assigned roles, **When** the user selects one, **Then** the portal makes only that role's authorized experience active and clearly identifies it.
3. **Given** a user in an active role, **When** the user switches to another assigned role, **Then** the portal updates the active context and role-specific navigation and content without ending the user's identity session.
4. **Given** a user attempting to switch to a role not assigned to that user, **When** the request is processed, **Then** the portal rejects the switch and retains or safely restores an authorized context.
5. **Given** a user viewing a location unavailable in the newly selected role, **When** the role switch completes, **Then** the portal displays an authorized landing experience for the selected role rather than the unavailable location.

### Edge Cases

- An account has no assigned supported role: deny access to role-protected experiences and provide a safe, non-sensitive explanation and support path.
- A user's role assignment changes while the user is signed in: subsequent authorization decisions use the currently valid assignments; stale privileges are not retained.
- A user has multiple assigned roles but no active role selected: do not expose role-protected content until an assigned role is active.
- A recovery request uses an unknown identifier: show the same confirmation as for a recognized account.
- A recovery proof is malformed, expired, previously used, or associated with a different account: reject it without changing credentials.
- A password reset completes while other sessions may exist: revoke only the session used for the reset; leave other sessions unchanged, subject to their independent three-hour inactivity timeout.
- A user signs out and then uses browser navigation to revisit protected content: require authentication again and do not reveal cached protected data.
- A role switch is attempted while the current page is not available under the target role: move to the target role's authorized landing experience.
- Repeated form submission while an authentication or recovery action is processing: prevent duplicate processing and preserve clear progress feedback.

## Requirements *(mandatory)*

### Business Rules

- The supported role names are Admin, Student, Teacher, and Coordinator.
- A user may have one or more assigned supported roles. Role assignment itself is not part of this feature.
- A user may select or switch only to a role currently assigned to that account.
- Role switching changes the active role context; it does not create, remove, or modify role assignments or the underlying user identity.
- Role access MUST follow the responsibilities and record boundaries in the approved feature list: Admin uses the Admin Module; Student uses the Student Module and their own academic and payment information; Teacher access is limited to assigned courses and assessment components; Coordinator access is limited to assigned batches and their students' results and reports.
- When a user has multiple roles, the active role MUST limit the experience to that role's feature-list responsibilities; role switching MUST NOT combine the permissions of all assigned roles into an implicit broader role.
- All supported roles MUST authenticate using an email address and password. Students MUST use the personal email address recorded for their account.
- Password reset MUST revoke only the session used to complete the reset; other active sessions remain active.
- An authenticated session MUST expire after three hours without authenticated activity. Each authenticated request MUST refresh the inactivity window; there is no separate maximum session age.
- Unauthenticated users cannot access protected portal experiences. Hiding a control is not authorization.
- Password recovery must not disclose whether an account exists.

### Functional Requirements

#### Authentication and session behavior

- **FR-001**: The portal MUST require successful authentication before granting access to protected experiences.
- **FR-002**: The portal MUST authenticate users with an email address and password. Students MUST use their personal email address recorded for the account; other roles MUST use the email address recorded for their account.
- **FR-003**: The portal MUST provide clear required-field validation and generic sign-in failure feedback that does not disclose whether an account exists or which credential was incorrect.
- **FR-004**: The portal MUST display a loading state during sign-in and prevent duplicate submissions while a request is processing.
- **FR-005**: After successful sign-in, the portal MUST place single-role users in their sole assigned role context and multi-role users in a role-selection step before displaying role-specific content.
- **FR-006**: The portal MUST support Admin, Student, Teacher, and Coordinator assignments and MUST allow one account to have more than one of these roles.
- **FR-007**: The portal MUST authorize access according to the role and record boundaries in the approved feature list: Admin Module capabilities for Admin; Student Module and own-record capabilities for Student; assigned-course and assigned-assessment capabilities for Teacher; and assigned-batch, student-result, and reporting capabilities for Coordinator.
- **FR-008**: Authorization MUST be enforced at the trusted system boundary for every protected action, including requests made without using the visible interface.
- **FR-009**: For denied access, the portal MUST provide an understandable unauthorized state, avoid exposing protected data, and offer a safe navigation path where appropriate.
- **FR-010**: A multi-role user MUST be able to switch among currently assigned roles while remaining the same authenticated user. The portal MUST identify the active role and update the authorized experience after a switch.
- **FR-011**: If the active page is not authorized in the newly selected role, the portal MUST navigate to an authorized landing experience for that role.
- **FR-012**: Signing out MUST end the current authenticated session, return the user to the unauthenticated experience, and require authentication before protected content can be viewed again.
- **FR-013**: The portal MUST expire an authenticated session after three hours without authenticated activity. Each authenticated request MUST refresh the inactivity window, with no separate maximum session age. A successful password reset MUST revoke only the session used to complete the reset; other sessions MUST remain active subject to the inactivity timeout.

#### Password recovery and reset

- **FR-014**: The portal MUST provide a password-recovery request flow for users who cannot sign in.
- **FR-015**: The portal MUST return the same user-facing confirmation for recognized and unrecognized account identifiers to reduce account enumeration.
- **FR-016**: For an eligible recognized account, the portal MUST send recovery instructions to the email address recorded for that account. A replacement password MUST contain at least 8 characters, including at least one letter and at least one number.
- **FR-017**: The portal MUST require valid, unexpired, single-use recovery proof before changing a credential.
- **FR-018**: The portal MUST validate replacement credentials against the stated minimum length and letter-and-number requirements, confirm successful reset without exposing sensitive data, reject unsuccessful reset attempts without changing the existing credential, and revoke only the session used for the reset.
- **FR-019**: Recovery and reset views MUST provide clear validation, loading, success, and error feedback consistent with the project design system.

#### User experience and security requirements

- **FR-020**: Authentication, recovery, role selection, role switching, unauthorized, and not-found states MUST follow [the project design system](../design-system.md); this feature MUST NOT define separate colors, typography, spacing, or component styles.
- **FR-021**: The portal MUST preserve user-entered non-secret values when safe after validation failures and MUST NOT expose passwords, recovery proofs, or other secrets in messages or logs.
- **FR-022**: Authentication and authorization errors MUST be actionable and MUST NOT expose stack traces, internal details, or protected record contents.
- **FR-023**: All user-controlled authentication and recovery inputs MUST be validated before they are accepted.
- **FR-024**: Security-relevant authentication and authorization failures MUST be available for diagnosis through appropriately protected operational records, without recording secrets or credentials.

#### Acceptance Criteria

- **AC-001 - Sign-in**: For an eligible account, valid credentials establish an authenticated session; invalid credentials establish no session and produce generic feedback. Required fields, loading feedback, and duplicate-submission prevention are present. (FR-001–FR-005)
- **AC-002 - Assigned roles and access**: A user can enter only a supported role currently assigned to the account. Each protected capability is allowed or denied according to the role and record boundaries in the approved feature list; direct unauthorized requests are denied without exposing protected content. (FR-006–FR-009)
- **AC-003 - Role selection and switching**: A single-role user enters that role's context. A multi-role user can select and switch only among assigned roles, sees the active role, and is moved to an authorized landing experience when the current page is unavailable in the selected role. (FR-005–FR-011)
- **AC-004 - Sign-out and session lifecycle**: Signing out ends the current authenticated session and blocks protected content afterward. Authenticated activity refreshes a three-hour inactivity window; a session idle for three hours is rejected. After password reset, only the session used for reset is revoked; other sessions remain active subject to their own inactivity windows. (FR-012–FR-013, FR-018)
- **AC-005 - Password recovery and reset**: Recovery requests return the same confirmation for recognized and unrecognized identifiers. Only valid, unexpired, single-use recovery proof plus a replacement password of at least 8 characters containing a letter and a number can complete reset; rejected attempts leave the existing credential unchanged. (FR-014–FR-018)
- **AC-006 - Feedback, validation, and security**: Authentication, recovery, and access-control states follow the design-system reference; user-controlled inputs are validated; messages reveal no secrets or account-existence information; security events are diagnosable without logging credentials or recovery proofs. (FR-019–FR-024)

### Key Entities *(include if feature involves data)*

- **User account**: A person's portal identity, authentication eligibility, and assigned supported roles.
- **Role assignment**: The relationship that grants a user membership in one supported role; the approved feature list defines the role's in-scope module and record boundaries.
- **Active role context**: The assigned role currently selected for an authenticated user's interaction with the portal.
- **Recovery proof**: A short-lived, single-use means delivered to the email address recorded for an account to verify authorization to replace its credential.
- **Authenticated session**: The period during which the portal recognizes the user's authenticated identity and active role; it ends on explicit sign-out, specified revocation, or three hours without authenticated activity.

### Non-Functional Requirements

- Authentication and protected-access behavior MUST comply with the constitution's security-by-default and least-privilege principles.
- The user experience MUST be usable with keyboard navigation and assistive technologies and meet the project design system's accessibility requirements.
- Loading, success, validation, unauthorized, and error feedback MUST be understandable, timely, and consistent across supported screen sizes.
- Authentication and recovery flows MUST not disclose credentials or account-existence information through visible messages.
- An explicit sign-out action MUST be readily available throughout authenticated experiences; on shared devices, users should sign out rather than rely on the three-hour inactivity timeout.
- Authentication and role behavior MUST be testable independently for each supported role, for users with multiple roles, and for unauthorized users.

### Out of Scope

- Account registration, user provisioning, or role assignment and removal.
- Defining capabilities or access to records beyond the role responsibilities and record boundaries in the approved feature list.
- Multi-factor authentication, single sign-on, social sign-in, or identity-provider selection unless separately approved.
- Administrative user-management screens or tools for changing role assignments.
- Feature-specific academic content or dashboard functionality for any role.
- Selecting data schemas, API endpoints, client/server frameworks, components, file structure, or implementation architecture.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: At least 95% of representative usability-test participants complete valid sign-in and sign-out without assistance.
- **SC-002**: 100% of authorization tests for capabilities outside a user's assigned-role permissions deny access without exposing protected information.
- **SC-003**: 100% of password-recovery tests for unknown identifiers return the same user-facing confirmation as recognized identifiers.
- **SC-004**: 100% of tests using invalid, expired, or already-used recovery proof leave the existing credential unchanged.
- **SC-005**: At least 95% of representative multi-role usability-test participants can select or switch to an assigned role and identify the active role without assistance.
- **SC-006**: 100% of session-lifecycle tests refresh a session on authenticated activity and reject it at or after three hours without activity.

## Assumptions

- The approved IIT Academic Portal feature list supplied by the user is the scope boundary; its role-module responsibilities define access scope for this feature.
- Accounts and role assignments are supplied by an existing or separately approved user-provisioning process; this feature does not create accounts or assign roles.
- The four role names supplied in the request are the complete supported set for this feature.
- The project design system at `specs/design-system.md` is the source of truth for authentication and access-control presentation.
- Students use their personal email address recorded for the account; other roles use their recorded account email address for email/password sign-in.
- Password-recovery instructions are delivered to the email address recorded for the account.
- Authenticated sessions use a sliding three-hour inactivity timeout with no separate maximum age. Password reset revokes only the session used for reset; other sessions remain subject to their own inactivity timeout.
