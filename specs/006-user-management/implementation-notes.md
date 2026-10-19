# Implementation Notes: User Management

**Feature**: [spec.md](spec.md) | **Tasks**: [tasks.md](tasks.md) | **Validation**: [quickstart.md](quickstart.md)

Decisions made during implementation that the plan left open, and where the code differs from a first reading of the design.

## Backend

- **No new account storage.** The directory (`UserAccountStore`) projects `PortalUser`, Identity roles, and an optional teacher-profile ID.
  Writes go through `UserManager` on the request's `PortalDbContext`, using `ITeacherStore` for the transaction, the shared last-Admin advisory
  lock (`4_004_001`), the active-Admin check, and saving. Reusing that store, rather than adding a second lock, is what serializes generic and
  Teacher changes against each other (proven in `UserManagementPostgresTests`).
- **Audit history for generic accounts.** `PortalUser` has no audited columns (it is exempt from `AuditedEntity`), so the detail's
  created/last-modified facts come from the account's own audit events (`EntityType = UserAccount`). A teacher's come from its profile. The
  bootstrap Admin and accounts created by seeding have neither, so the fields are omitted.
- **Audit events.** Business: `account.created`, `account.email.changed`, `account.roles.changed`, `account.deactivated`,
  `account.credential-setup.reissued`, `account.credential-setup.completed` (approved fields `accountStatus`, `roles`, `defaultRole`; `email`
  is a change indicator only). Security: `auth.credential-setup.rejected`, with a reason and no account, like a failed reset.
- **Ownership delegation.** For an account with a teacher profile, the email, role, and deactivation endpoints call `TeacherProfileService`,
  `TeacherRoleService`, and `TeacherLifecycleService`, so the `teacher.*` events and profile state stay authoritative. Generic Admin/Student
  role changes are deliberately narrow: Student is Student-only, Teacher needs a profile (409), Coordinator is never accepted, so for a generic
  account the endpoint mostly changes the default role.
- **Self-deactivation** is decided from `IAuditRequestContext.ActorUserId`, the signed-in account, so no caller ID is accepted from the client.
  The last-Admin rule can only be reached through the API by an Admin acting on another account while being the other active Admin, so its
  refusal is tested directly against the service with no request actor.
- **Setup tokens.** A dedicated `DataProtectorTokenProvider<PortalUser>` (`AccountCredentialSetupTokenProvider`) is registered with its own
  name, protector purpose, and lifespan (3 days by default). The default provider keeps password reset at 1 hour. Tokens are bound to the
  security stamp, which rotates on reissue, completion, and email change.
- **Pending accounts and recovery.** A reset link to a pending account would set a password without activating it and then block setup, so
  `PasswordResetContentProvider` now refuses pending accounts (`AccountPendingSetup`). Setup completion also removes any password a pending
  account somehow has before adding the new one, so the setup link always works while the account is pending.
- **Teacher invitation.** `TeacherProvisioningService` now requests `AccountCredentialSetup` (occurrence = user ID) instead of
  `TeacherInvitation`; this closes feature 005's T016. `TeacherInvitationProducerTests` moved into `AccountCredentialSetupOwnerTests`, and two
  005 tests that assumed the old event were updated (`NotificationDispatchTests`, `NotificationResendTests`, which now uses a direct
  `StudentInvitation` request to keep covering the unavailable-provider path).
- **Seed.** The new event is appended last in `NotificationEventType`, so every earlier deterministic template ID is unchanged; the migration
  inserts template `...0008` and its revision only.

## Frontend

- Every new screen is lazy-loaded, so the initial bundle stays under its 500 kB budget (473.40 kB after this feature).
- `/admin/users/:userId` is both the account detail and the maintenance view (the task list names a detail view in `user-list.ts`; a separate
  routed component keeps the directory small and gives each account a linkable URL).
- The account-setup page lives under `features/users` and reuses the reset page's password rules. It removes `userId` and `proof` from the
  address bar on load and drops the proof from memory after any answer from the service.
- No new design-system component was needed: the pages use the existing tables, fields, choice rows, alerts, badges, cards, and the shared
  infinite-loading footer.

## Not done

- **Coordinator setup-email producer**: no batch-assignment owner exists. The test is present and skipped with the gate as its reason.
- **SC-002 usability study** with 10 representative Admins: not run.
- **Operational gates** (unchanged from the plan): public https setup URL in Production, a persisted and shared Data Protection key ring (setup
  links are protected with it and become invalid if keys are lost), the first-Admin bootstrap, and a Production email provider.

## Correction (2026-10-19)

The Teacher form's pending-account wording was changed after the last frontend test run recorded in quickstart.md, so
`teacher-form.spec.ts` ("explains a pending account") failed from then on. Feature 007 updated that spec to the current wording; the
recorded 251-passed result predates that edit.
