# User Management API Contract

**Status**: Implemented contract for [spec.md](../spec.md). All account routes use explicit DTOs and the existing ASP.NET Core Problem Details conventions.

## Common Behavior

- Account-administration routes require an authenticated session whose **active** role is Admin through `PortalPolicies.AdminModule`. Anonymous requests receive 401; any other active role receives 403. UI visibility is not authorization.
- State-changing requests require the existing `X-CSRF-Token` anti-forgery header.
- Account changes use the request-scoped Identity/`PortalDbContext` unit of work, the shared last-Admin transaction lock, and safe allowlisted audit events.
- Never return password hashes, proofs, setup links, provider credentials, or raw Identity/provider errors.
- Admin account request/response DTOs do not contain passwords or setup proofs. The public setup-completion endpoint uses a separate request-only DTO; its `proof` and `newPassword` are accepted only in that request and MUST never appear in responses, audit records, or logs.
- Do not expose physical deletion.

## Routes

| Method and route | Contract |
|---|---|
| `GET /api/admin/users` | Search all existing portal accounts, including Admin, Student, Teacher, and Coordinator role assignments. Query: `search` (registered email or stable user ID), `role`, `status`, `page` (1-based, default 1), and `pageSize` (1-100, default 25). The UI requests bounded pages sequentially through the shared infinite-loading pattern and shows no numbered pages or totals. Stable ordering is normalized email then user ID. |
| `GET /api/admin/users/{userId}` | Return stable user ID, registered email, account status, assigned roles, default role, audit timestamps/actors, and an optional Teacher Management link/reference when a TeacherProfile exists. Never return credentials or setup proof. |
| `POST /api/admin/users` | Create a generic Admin or Student Identity account. Body: `email` and `accountType` (`Admin` or `Student`). Normalize/validate email, enforce global uniqueness, create the selected single role/default role, set `PendingCredentialSetup`, create no password, stage audit, commit, then request the common setup notification. Teacher creation delegates to existing `POST /api/admin/teachers`; Coordinator cannot be directly created/assigned and requires its assignment owner, which requests the common setup event after committing a newly provisioned Coordinator account/assignment. |
| `PUT /api/admin/users/{userId}/email` | Body: `email` and `expectedConcurrencyStamp`. Email is the only generic account identity field editable through User Management. Update only the Identity email/username using existing normalization/uniqueness rules; a conflict returns 409 without partial state. Teacher email updates use the existing Teacher Management profile operation. Audit the successful change without writing the email to arbitrary metadata. |
| `POST /api/admin/users/{userId}/deactivation` | Deactivate an Admin/Student account, retain its account/roles/history, revoke all sessions, and audit atomically. Reject an Admin attempting to deactivate their own account. A Teacher account delegates to the existing Teacher Management deactivation operation so its profile and roles remain consistent. Coordinator-derived membership is not edited directly. |
| `PUT /api/admin/users/{userId}/roles` | Permit only the clarified role policy. For a Teacher with a TeacherProfile, delegate to the existing Teacher role service: retain Teacher while active, allow Admin add/remove, preserve Coordinator as assignment-derived, preserve an assigned default role, and serialize last-Admin protection. Student membership is Student-only; other unsupported role changes return a safe 400/409. |
| `POST /api/admin/users/{userId}/credential-setup-reissues` | Empty body; require active Admin and `PendingCredentialSetup`. Rotate the security stamp to invalidate earlier setup links, request a new `AccountCredentialSetup` occurrence through the notification service, and audit the reissue without returning a link/proof. Active/deactivated accounts return a safe 409. |
| `POST /api/auth/account-credential-setups` | Public setup completion. Body: `userId`, `proof`, and `newPassword`. Verify the distinct setup-token purpose and pending status; apply the existing password policy; add the password, activate the account, rotate its security stamp, and audit in one transaction. Invalid/expired/replayed proof returns a generic invalid-link response and no account details. |

## Credential Setup Security Contract

- Generate the setup proof with Identity's configured Data Protection token provider using a purpose distinct from password recovery; never call `GeneratePasswordResetTokenAsync` for setup.
- The setup provider rechecks that the account is still pending and the destination still equals the registered address immediately before delivery.
- Proof values and rendered security content exist only transiently for the delivery attempt. The setup API never returns a proof except through the email link.
- Each Admin reissue invalidates earlier setup links and is distinct from generic notification resend, which accepts only Failed notification records.
- A setup or delivery failure leaves `AccountStatus` pending. Password recovery remains non-enumerating and does not activate pending accounts.

## Errors

- `400`: invalid email/password/role/request shape; use field-keyed Problem Details.
- `401` / `403`: anonymous or non-Admin account-management caller.
- `404`: unknown account.
- `409`: duplicate email, stale concurrency stamp, self-deactivation or last-Admin violation, account type owned by another feature, or credential setup attempted for an account not pending.

## Implemented DTOs

- `UserAccountResponse`: `userId`, `email`, `accountStatus` (`Active`, `PendingCredentialSetup`, `Deactivated`), `roles`, `defaultRole`,
  `teacherId` (only when Teacher Management owns a profile), `concurrencyStamp` (Identity's optimistic-concurrency value, not a credential),
  and `createdAtUtc`/`createdByUserId`/`createdBy`/`lastModifiedAtUtc`/`lastModifiedByUserId`/`lastModifiedBy`. For a teacher these come from
  the teacher profile; for a generic account they come from its `UserAccount` audit events; they are omitted when neither exists (for example
  the bootstrap Admin). Null values are omitted.
- `UserAccountPageResponse`: `items`, `page`, `totalCount`, `totalPages` (the UI uses only whether more pages exist; it shows no numbers).
- `CreateUserAccountRequest` (`email`, `accountType`), `UpdateUserEmailRequest` (`email`, `expectedConcurrencyStamp`),
  `UpdateUserRolesRequest` (`roles`, `defaultRole`). Unknown JSON members are rejected, so a `password` field is a 400.
- `CredentialSetupReissueResponse`: `account`, `setupEmailRequested` (false when the notification store could not retain the request; the
  stamp has still rotated), and `message`. It never contains a link or proof.
- `AccountCredentialSetupRequest` (public, request-only): `userId`, `proof`, `newPassword`. Success is 204. Every refused link is the same 400
  Problem Details titled "Setup link not valid"; a policy failure is a 400 keyed by `NewPassword`.

## Implemented status codes

- Directory: invalid `role`, `status` (names only, never numbers), `page`, or `pageSize` is a field-keyed 400; an unknown ID is 404.
- Create: missing or invalid `email` or an `accountType` other than the four roles is a field-keyed 400; `Teacher` and `Coordinator` and a
  duplicate normalized email are 409.
- Email: a missing `expectedConcurrencyStamp` is a field-keyed 400; a stale stamp or a duplicate email is 409. For an account with a teacher
  profile the change is made by the Teacher Management profile operation (its validation messages are field-keyed 400s).
- Roles: Coordinator, an unsupported role, an empty set, Student combined with anything, a Student account given another role, or a default
  role that is not assigned is a field-keyed 400; Teacher for an account without a teacher profile, a deactivated account, and the last active
  Admin are 409. A teacher account is delegated to the Teacher Management role service.
- Deactivation: the caller's own account and the last active Admin are 409; repeating it on a deactivated account changes nothing. A teacher
  account is delegated to Teacher Management deactivation.
- Reissue: 404 unknown, 409 not pending.
