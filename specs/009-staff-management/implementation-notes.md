# Implementation Notes: Staff Management

**Feature**: [spec.md](spec.md) | **Tasks**: [tasks.md](tasks.md) | **Validation**: [quickstart.md](quickstart.md) | **Recorded**: 2026-10-21, revised 2026-10-22

## Backend

- `StaffProfile` (`Domain/Staff`) is an `AuditedEntity` with `UserId`, `Name` (at most 200), `MobileNumber` (at most 50), a required
  `DesignationId`, and `IsActive` (default true; `Deactivate()` only, no reactivation). `Create` and `Update` refuse a blank or
  over-long value or an empty Designation. There is no verified state or document field; profiles are never deleted.
- `StaffProfileService.CreateAsync` validates the email (User Management's `ValidateEmail`) and every field first, collecting all errors,
  then looks the email up with `UserManager.FindByEmailAsync`:
  - **New email:** in one transaction, a `PortalUser` with no password, `PendingCredentialSetup`, only the Admin role and Admin as
    default, the `account.created` event, the profile, and `staff.profile.created` (metadata `accountCreated: true`). After the commit
    it requests `AccountCredentialSetupService.InitialRequest`; `setupEmailRequested` is true only if the request was accepted (or was a
    duplicate). A failed request leaves the account pending for a reissue from Accounts.
  - **Existing account:** linked when it has the Admin role, is not deactivated, and has no profile. A pending-setup Admin is accepted
    too (decision: the spec only excludes deactivated and non-Admin accounts); its account and setup state are untouched.
- `StaffLifecycleService.DeactivateAsync` refuses the signed-in Admin's own profile, then in one transaction takes the shared
  Admin-invariant advisory lock (`TeacherStore.AdminInvariantLockKey`, now `internal`), refuses the last active Admin, marks the profile
  inactive, sets the account `Deactivated`, revokes its sessions (`SessionRevocationReason.AccountDeactivated`), and stages
  `staff.profile.deactivated`. A profile already inactive with a deactivated account returns unchanged with no audit.
- Updates refuse an inactive profile, and (as before) one whose account was deactivated or lost Admin (409). An unchanged save audits
  nothing.
- The eligible-account endpoint and `IStaffStore.ListEligibleAccountsAsync` were removed. `StaffStore` gained the transaction, lock, and
  other-active-Admin queries, mirroring `TeacherStore`.
- Migration `20261021174956_StaffProfiles` (applied to Development on 2026-10-22) is unchanged. `20261022043821_StaffProfileLifecycle`
  adds `IsActive boolean NOT NULL DEFAULT true` (applied to Development on 2026-10-22), so existing profiles stay active; the model has no default, so EF always writes the value.

## Frontend

- `/admin/staff` (directory), `/admin/staff/new`, and `/admin/staff/:id`, lazy-loaded, with a "Staff" entry in the Admin navigation after
  Teachers. The account dropdown's My Profile stays a disabled button.
- Adding a staff member takes an email address (no account picker). The directory's notice says which outcome happened: a new account
  and setup email sent, a new account whose setup email could not be requested (reissue from Accounts), or a link to an existing Admin
  account.
- The directory has a Staff column ("Active"/"Inactive" badge) and a "Deactivate…" (danger outline) action for active staff that asks
  for confirmation in the row; a refusal (own account, last active Admin) is shown in that row. A deactivated staff member's form is
  read-only with an info alert.
- The confirmation checkbox starts unticked and is cleared after every save.
- The session does not expose the signed-in user's id, so the directory cannot hide "Deactivate…" on the Admin's own row; the service
  refuses it with 409 and the row shows why.

## Known gap

- User Management's own account deactivation does not mark a linked StaffProfile inactive. The profile is then shown read-only because
  its account is deactivated, but `IsActive` stays true. Staff deactivation should be used for staff accounts.
