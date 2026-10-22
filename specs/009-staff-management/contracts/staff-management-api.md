# Staff Management API Contract

**Status**: Implemented on 2026-10-22 (provisioning/attachment by email and deactivation); see [implementation-notes.md](../implementation-notes.md).

## Security and Common Behavior

- All routes require an authenticated session actively using the Admin role through the existing Admin policy. The target staff account must already have the Admin role assigned.
- State-changing requests use the existing anti-forgery protection.
- DTOs are explicit; no password, credential-setup proof, role mutation, or verification document is accepted or returned.
- Invalid input returns field-keyed validation errors and changes neither StaffProfile nor audit history. Staff profile changes and their required audit event commit atomically.
- Staff Management may provision a new Admin PortalUser with its StaffProfile atomically or attach a StaffProfile to an eligible existing Admin account. It never changes account roles or credentials; the only account-status change it performs is the specified staff deactivation. My Profile has no write route.

## Routes

| Method and route | Behavior |
|---|---|
| `GET /api/admin/staff` | Return a bounded, searchable list of StaffProfiles with Name, Mobile number, Designation, and linked account identity. |
| `POST /api/admin/staff` | Create a staff member from `email`, profile fields, and `manualVerificationConfirmed: true`. For a new email, atomically create a pending Admin PortalUser and StaffProfile, then request credential setup after commit. For an existing Admin account that is not deactivated (active or pending setup) and has no StaffProfile, attach the profile without changing the account or requesting setup again. |
| `GET /api/admin/staff/{staffProfileId}` | Return one StaffProfile, its linked account identifier/email, current Designation name, and record history. |
| `PUT /api/admin/staff/{staffProfileId}` | Replace the profile's Name, Mobile number, Designation reference, and manual-verification confirmation. The same route/workflow is used when the acting Admin updates their own StaffProfile. |
| `POST /api/admin/staff/{staffProfileId}/deactivation` | Deactivate the StaffProfile and linked Admin account, revoke active sessions, and preserve profile/audit history. Reject self-deactivation and removal of the last active Admin. |

No physical deletion, account role-management, or My Profile update route is defined. Staff deactivation is irreversible through Staff Management.

## Request and Response Fields

- `email`: required on create for account lookup/provisioning. A new email creates a pending Admin account; an existing Admin account that is not deactivated and has no StaffProfile may be linked. Email is not changeable through Staff Management.
- `name`: required display name, trimmed; maximum 200 characters.
- `mobileNumber`: required contact number, trimmed; maximum 50 characters. The field is not represented as mobile-verified.
- `designationId`: required reference to an existing active Designation.
- `manualVerificationConfirmed`: required boolean and must be `true` on every create/update. It is an attestation for the audit event, not a StaffProfile verification status.
- Responses include `id`, `userId`, account email, `accountStatus`, `hasAdminRole`, Name, Mobile number, `designationId`, resolved `designationName`, `isActive`, and created/last-modified actor and timestamps. Create responses also carry `accountCreated` and `setupEmailRequested` (both omitted from every other response). Verification confirmation is visible in the audit event; no documents or proofs are returned.

## List Query

- `search` optionally matches normalized Name, account email, Mobile number, and Designation name.
- Results are bounded and paginated using the existing Admin-directory conventions; no unbounded profile list is returned.

## Validation and Errors

- `400`: invalid staff-list `page`/`pageSize`, invalid email, missing/blank Name or Mobile number, over-limit fields, missing/false verification confirmation, invalid/deleted/inactive Designation, or unknown request fields. Errors are keyed to the corresponding field or query parameter.
- `401` / `403`: anonymous or non-Admin caller.
- `404`: unknown StaffProfile ID.
- `409`: email belongs to a deactivated or non-Admin account, an Admin account already has a StaffProfile, profile/account lifecycle conflicts, a self/last-Admin deactivation is attempted, or a concurrent uniqueness conflict occurs.
- A rejected request does not save profile changes or a successful-change audit event.

## Audit

Successful creation, updates, and deactivation use allowlisted events `staff.profile.created`, `staff.profile.updated`, and `staff.profile.deactivated`; creation of a new PortalUser also records the existing account-created event. Events record the acting Admin, target PortalUser/StaffProfile, UTC timestamp, changed fields, outcome, and `manualVerificationConfirmed: true` for profile creates/updates. New account/profile events commit in one transaction; deactivation profile/account events and session revocations commit atomically. Credential-setup email is requested only after the new account/profile commit. No submitted identity document, evidence image, or proof content is retained.

## Contract Details

- **Directory.** `GET /api/admin/staff?search=&page=&pageSize=` returns `{ items, page, totalCount, totalPages }`; 25 per page by default,
  100 at most (`pageSize` outside 1-100 or `page` below 1 is a 400 on that key). `search` is normalized (trimmed, single-spaced, upper
  case) and matched as "contains" against the name, mobile number, account email, and Designation name. Ordered by name, then id.
- **Response.** `id`, `userId`, `email`, `accountStatus` (`Active`, `PendingCredentialSetup`, `Deactivated`), `hasAdminRole`, `name`,
  `mobileNumber`, `designationId`, `designationName` (the current catalog name, even if the Designation was deactivated or deleted since),
  `isActive`, and `createdAtUtc`/`createdByUserId`/`createdBy`/`lastModifiedAtUtc`/`lastModifiedByUserId`/`lastModifiedBy`. A create
  response adds `accountCreated` and `setupEmailRequested`. No credential, proof, document, or verified state.
- **Create.** `POST /api/admin/staff` with `email`, `name`, `mobileNumber`, `designationId`, `manualVerificationConfirmed`; `201` with the
  profile and account outcome. A new email creates one pending Admin PortalUser plus StaffProfile atomically, then requests credential
  setup after commit (`accountCreated: true`; `setupEmailRequested` is false when the notification request was not accepted, and the
  account stays pending so the setup can be reissued from Accounts). An existing Admin account that is not deactivated (active or pending
  setup) and has no StaffProfile is linked without changing account state or resending setup (`accountCreated: false`,
  `setupEmailRequested: false`). A deactivated or non-Admin account, or an account that already has a profile, is `409`.
- **Update.** `PUT /api/admin/staff/{staffProfileId}` with `name`, `mobileNumber`, `designationId`, `manualVerificationConfirmed`; `userId`
  or any other field is a 400. A deactivated staff member, or a profile whose account was later deactivated or lost the Admin role, stays
  readable but cannot be changed (`409`). Saving unchanged values still needs the confirmation, and saves and audits nothing. A Designation the profile already has may
  stay after it is deactivated; a newly chosen one must be live and active.
- **Validation messages.** `email`: "Enter the staff member's email address." / the portal's email-format and length messages (the same
  check as User Management).
  `name`: "Enter the staff member's name." / at most 200 characters. `mobileNumber`: "Enter the mobile number." / at most 50 characters.
  `designationId`: "Choose a designation." / "Choose an active designation." `manualVerificationConfirmed`: "Confirm that you manually
  verified this information against the institution's records." Name and mobile number are trimmed; inner spaces are kept.
- **Audit.** Profile events use entity type `StaffProfile` and the profile id. On new-email provisioning, the linked account-created audit
  event and profile-created event commit together. Changes keep before/after values for `name` and `designationId`; `mobileNumber` is
  recorded as "changed" only (personal data). Create/update metadata holds the target `userId`, `manualVerificationConfirmed: true`, and
  (on create) `accountCreated`. `staff.profile.deactivated` records `isActive` and `accountStatus` before/after, with `userId` and
  `sessionsRevoked` in its metadata, atomically with the change.
- **Deactivation.** `POST /api/admin/staff/{staffProfileId}/deactivation` marks the profile inactive, deactivates the Admin account, and
  revokes active sessions (reason `AccountDeactivated`); profile/account/audit records and roles remain. `200` with the profile.
  Self-deactivation is `409` ("You cannot deactivate your own staff account. Ask another Admin to do it."); deactivating the last active
  Admin is `409` with Teacher Management's last-Admin message, checked under the shared Admin-invariant advisory lock. Repeating a
  completed deactivation returns `200` and changes and audits nothing; Staff Management has no reactivation or physical delete route.
- **No other routes.** There is no `PUT /api/admin/staff/me`, eligible-account listing, role-edit route, or My Profile write route.
- **Configuration.** A Designation used by a staff profile cannot be deleted (409, "Deactivate it instead"), like one used by a teacher.

