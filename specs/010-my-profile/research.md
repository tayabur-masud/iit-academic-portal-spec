# Research: My Profile

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Research date**: 2026-10-23

## Findings and Decisions

### Session identity and active role

- **Finding**: A validated portal session places the account ID in the `NameIdentifier` claim and only the session's active role in the role claim. Assigned roles are separate claims. Module authorization policies check the active role. The current-session endpoint reports roles but does not provide profile data.
- **Decision**: The My Profile controller obtains the account ID and active role from the authenticated principal. The request has no account ID, role parameter, or arbitrary profile identifier. Support only the four existing active roles: Admin, Student, Teacher, and Coordinator.
- **Rationale**: Deriving both values from the validated session prevents users from selecting another account or switching the requested profile role independently of their active authorization context.
- **Alternatives considered**: Accepting a user ID or role in the query/path was rejected because it creates a cross-account or cross-role authorization risk. Reusing the current-session endpoint was rejected because it contains no profile information.

### Profile ownership and role mapping

- **Finding**: StaffProfile owns Admin staff Name, Mobile number, and Designation. StudentProfile owns the Student's personal/contact/address/guardian values and prior academic history. TeacherProfile owns teacher display name, optional staff identifier, optional phone, and optional Designation. Coordinator is synchronized as an additional role on a teacher account and has no separate profile entity.
- **Decision**: Map active Admin to StaffProfile, Student to StudentProfile, and Teacher or Coordinator to TeacherProfile. For a multi-role account, show only the profile associated with its active role. A missing profile returns an explicit unavailable state and never falls back to another role's profile.
- **Rationale**: The role's existing profile owner remains authoritative, while the same account can see the correct profile after an active-role switch. Coordinator's existing domain relationship is preserved without inventing a Coordinator profile.
- **Alternatives considered**: A new Coordinator profile was rejected because no such domain entity exists. Showing every assigned role's profile at once was rejected because the session already distinguishes an active role and the spec requires a role-specific view.

### Safe Student projection

- **Finding**: The current Student profile query returns the personal profile, prior academic history, account standing, audit-related identifiers, and every admission summary. The approved My Profile scope includes all personal Student profile details, guardian details, and prior academic history, but excludes internal/audit/security data; admissions and current results are separate records.
- **Decision**: Add a self-view-specific Student query/projection that loads the StudentProfile, resolved mailing/permanent locality names, and prior academic history only. Map it to an allowlisted My Profile response without profile/user/history/catalog IDs, account status, audit metadata, admission summaries, or academic-result data.
- **Rationale**: Reusing the full management detail would load and risk exposing unrelated admissions and internal metadata. A targeted projection follows data minimization and the constitution's requirement to retrieve only needed data.
- **Alternatives considered**: Reusing the existing Student management response was rejected because it includes every admission and internal identifiers. Omitting prior education or guardian values was rejected because the clarification explicitly includes them.

### Read API and errors

- **Finding**: There is no self-profile endpoint. Existing Staff and Teacher detail stores query by profile ID; Student has an account-scoped profile method, but it includes admissions. Existing API middleware requires authentication by default and marks authenticated responses `no-store`.
- **Decision**: Add one authenticated `GET /api/profile` endpoint returning an explicit active-role discriminator, account email, and one role-specific profile object. For a missing profile, return the user's email/active role with an empty profile state; do not return a different role's profile. Use existing problem responses for unauthenticated requests and safe errors for lookup failures.
- **Rationale**: A singleton self-resource supports the account menu for every role without allowing client-selected identities. Explicit response types prevent management-only fields from leaking.
- **Alternatives considered**: A generic management endpoint was rejected because existing management policies and DTOs are role-specific and include administrative fields. Separate public routes per profile type were rejected because the UI and session already have one active role.

### Frontend placement and behavior

- **Finding**: The account shell shows the active role and assigned-role switcher. My Profile is currently a disabled button. Role-scoped pages live under role areas; switching roles leaves a page not belonging to the new role's area.
- **Decision**: Add a lazy-loaded `/profile` route beneath the authenticated shell, enable the shared account-menu item, and render only the active role's response. Use existing UI conventions for loading, missing-profile, and recoverable error states. Keep every field view-only.
- **Rationale**: A shell-level route is available to all supported roles without duplicating pages under each role's area. Returning to the profile after a role switch requests the newly active role's profile.
- **Alternatives considered**: Duplicating the page under each role area was rejected because the data and interaction are shared. Adding profile edit controls was rejected by the approved scope; existing role-management features remain the owners of updates.

### Storage, migration, and dependencies

- **Decision**: Reuse existing account/profile/Designation tables and store interfaces, adding only user-scoped read methods and response models. No new dependency, persisted field, table, or database migration is needed.
- **Rationale**: The feature is a read-only projection over existing data; schema changes would duplicate information without adding value.
- **Alternatives considered**: Persisting a copied My Profile record was rejected because it could become stale and contradicts the spec.

## Repository Evidence

- `iit-academic-portal-service/src/IitAcademicPortal.Api/Authentication/PortalClaims.cs`: validated user ID, active role, and assigned-role claims.
- `iit-academic-portal-service/src/IitAcademicPortal.Api/Authorization/PortalPolicies.cs`: secure-by-default authentication and active-role policies.
- `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IStudentStore.cs`: existing account-scoped Student profile lookup.
- `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/ITeacherStore.cs`: existing Teacher profile lookup by account ID.
- `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IStaffStore.cs`: Staff details are currently queried by profile ID and need an account-scoped read.
- `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherCoordinatorRoleSynchronizer.cs`: Coordinator is associated with an active TeacherProfile.
- `iit-academic-portal/src/app/layout/shell/shell.html` and `shell.ts`: shared account menu, active role selection, and shell navigation.
- `iit-academic-portal-service/src/IitAcademicPortal.Api/PortalApi.cs`: authenticated responses receive `Cache-Control: no-store` by default.

No external service or unresolved technology choice is required.
