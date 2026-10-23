# Feature Specification: My Profile

**Feature Branch**: `feature/010-my-profile`

**Created**: 2026-10-23

**Status**: Ready for planning

**Input**: User description: "Next feature: My Profile."

## Clarifications

### Session 2026-10-23

- Q: Which personal information should a Student see on My Profile? → A: All personal details in the student profile, including guardian details and prior education; exclude internal, audit, and security data.
- Q: The account menu shows the word "Account". What should it show instead? → A: The signed-in user's name, from the active role's profile (the same name My Profile shows), falling back to another of the account's profile names and then to the email; initials replace the role letter in the avatar.
- Q: Where should the account menu and the theme choice sit, and how should the name read? → A: Swap them: the account menu moves to the right of the header and greets the user as "Welcome, <name>"; the theme choice moves to the bottom of the sidebar.
- Q: Should the role be shown under the name? → A: Only for accounts with more than one role, so a multi-role user can always see the active role (001 FR-010, SC-005); a single-role account shows just the greeting, its menu has no role section (only My Profile and Sign out), and its role is only in the screen-reader text.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View My Profile (Priority: P1)

As a signed-in portal user, I want to view my own account identity and role-specific profile so I can confirm the personal information associated with my account without asking another user to look it up.

**Why this priority**: The account menu already identifies a My Profile destination. Showing every portal user their own current information provides immediate value while preserving the previously agreed view-only boundary.

**Independent Test**: Sign in separately using Student, Teacher, Admin, and Coordinator active roles; open My Profile and verify that each view shows the signed-in account email and only its own applicable role profile. For Coordinator, verify the linked Teacher profile is shown. Confirm no profile edits are offered and a missing profile produces a clear unavailable state.

**Acceptance Scenarios**:

1. **Given** a signed-in user has an active Student, Teacher, or Admin role, **When** they open My Profile, **Then** they see their own account email and the data associated with that role's profile. A Student sees all personal details in their Student profile, including guardian details and prior education; Teacher and Admin views also show their current Designation when one is assigned.
2. **Given** a signed-in user has the active Coordinator role, **When** they open My Profile, **Then** they see their own account email and linked Teacher profile information, with no separate Coordinator profile implied.
3. **Given** an account has multiple roles, **When** its user switches the active role and opens My Profile, **Then** the view shows only information appropriate to the active role and still belongs to that account.
4. **Given** the user's profile information is changed in its owning management workflow, **When** the user next opens or refreshes My Profile, **Then** the page shows the current saved information rather than an independently maintained copy.
5. **Given** a signed-in user has no available profile for the active role, **When** they open My Profile, **Then** the page explains that profile details are unavailable and does not substitute another role's or user's information.
6. **Given** a user is viewing My Profile, **When** they inspect available actions or attempt to change profile data there, **Then** no edit or save action is available and no profile change is accepted through this feature.
7. **Given** a request is made by an anonymous user or a user requesting another account's data, **When** My Profile data is requested, **Then** access is denied without disclosing whether the other profile exists.
8. **Given** profile data is loading or cannot be retrieved, **When** the page is opened, **Then** the user receives a clear loading or recoverable error state without seeing stale or unrelated account details.
9. **Given** a user uses keyboard navigation or a supported screen size, **When** they open and read My Profile, **Then** the menu action and profile content remain operable and readable without content being obscured.
10. **Given** a signed-in user, **When** any page is shown, **Then** the account menu at the right of the header greets them as "Welcome, <name>" (the active role's profile name, otherwise another of their profile names, otherwise their email) with their initials (and, for a multi-role account, the active role under the greeting), and after a role switch it shows the new role's profile name and role.

### Edge Cases

- The signed-in account has no profile for its active role or the linked profile is unavailable; show an explicit unavailable state without substituting another role's or account's information.
- The profile or Designation changes after the page was previously opened; a fresh view reflects the current saved value.
- The requested account identity does not match the authenticated user's account; deny access without confirming whether the other profile exists.
- A request fails or returns incomplete profile data; show a safe error/unavailable state and do not expose internal details.
- The user has multiple portal roles; show only information appropriate to the active role, and update the view after a role switch.
- A Coordinator account has no linked Teacher profile; show an unavailable state and do not invent a Coordinator-specific profile.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The portal MUST allow an authenticated user with any supported active role (Admin, Student, Teacher, or Coordinator) to open My Profile from the account menu.
- **FR-002**: My Profile MUST display only the signed-in account's email and data from the profile associated with the active role: StaffProfile for Admin, StudentProfile for Student, and TeacherProfile for Teacher or Coordinator. For Students, this includes all personal details in their StudentProfile, including guardian details and prior education. It MUST display the current Designation name when that profile has an assigned Designation. It MUST exclude internal identifiers, audit metadata, credentials, and security data.
- **FR-003**: The page MUST show current saved profile values and MUST NOT maintain a separate copy of the account or role-profile data.
- **FR-004**: My Profile MUST be view-only. It MUST NOT provide profile create, update, deactivation, account status, credential, or role-change actions; profile changes remain in the existing role-specific management workflows.
- **FR-005**: The portal MUST enforce authentication and scope profile data to the signed-in account on every request, regardless of active role or interface visibility.
- **FR-006**: A missing or unavailable role profile MUST produce a clear state without revealing another role's or account's data or secrets.
- **FR-007**: The page MUST provide clear loading and recoverable error states and MUST NOT expose credentials, credential-setup proofs, internal identifiers, audit metadata, or unrelated account information.
- **FR-008**: The page and account-menu entry MUST be usable with keyboard navigation and assistive technology and remain readable at supported screen sizes.
- **FR-009**: The account menu, at the right of the header, MUST greet the signed-in account by its own name ("Welcome, <name>"), resolved by the service with each session response: the active role's profile name, otherwise another of the account's profile names (Staff, Teacher, Student), otherwise the account email. It MUST NOT show another account's name, keep a separate copy of the name, or need a request beyond the session response. A long name MUST be shortened with an ellipsis and available in full as a tooltip.

### Key Entities *(include if data involved)*

- **Portal account**: The signed-in user's identity, account email, assigned roles, and active role used to scope the view to that user.
- **StudentProfile**: A student's personal, contact, address, guardian, and prior-education profile information, maintained through Student Management.
- **TeacherProfile**: A teacher's display name, optional staff identifier, optional phone, and optional Designation; Coordinator is an additional role associated with a teacher account and does not define a separate profile.
- **StaffProfile**: An Admin staff member's Name, Mobile number, and Designation, maintained through Staff Management.
- **Designation**: The current catalog name associated with a TeacherProfile or StaffProfile and displayed without copying it into a separate profile value.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In all successful views, 100% of displayed profile fields correspond to the signed-in user's account and the applicable current role profile.
- **SC-002**: Across authorization checks, 0 profile details are disclosed to anonymous users or users requesting another account's profile, across all supported roles.
- **SC-003**: A signed-in user can reach their profile from the account menu in no more than two user actions after signing in.
- **SC-004**: 100% of My Profile views expose no profile edit, account deactivation, credential, or role-change action.
- **SC-005**: For accounts without a profile linked to the active role or during a retrieval failure, users receive an explicit safe state without unrelated or stale profile data.

## Assumptions

- My Profile is available to users actively using any supported role: Admin, Student, Teacher, or Coordinator. A Coordinator account uses its linked TeacherProfile because Coordinator is an additional role, not a separate personal profile.
- When an account has multiple roles, the active role determines which role profile is displayed; all views remain scoped to the same signed-in account.
- My Profile is read-only. Each role's existing management workflow remains responsible for changing that role's profile information.
- The view uses existing account and role-profile information; it introduces no independent profile data or verification workflow.
- My Profile does not change account identity, roles, credentials, activation, or deactivation.
