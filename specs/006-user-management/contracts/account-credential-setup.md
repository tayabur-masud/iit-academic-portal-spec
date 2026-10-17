# Account Credential Setup Contract

**Status**: Implemented internal Identity/notification contract for [spec.md](../spec.md). This is separate from password recovery and does not expose a public credential-management API to Admins.

## Request and Ownership

- Generic Admin/Student account creation and Teacher provisioning request one common `AccountCredentialSetup` security event only after the owning account transaction commits. The batch-assignment owner MUST request the same event after committing a newly provisioned Coordinator account and authoritative assignment.
- Teacher account/profile creation remains owned by Teacher Management. User Management delegates to its existing provisioning operation and does not create/update `TeacherProfile`.
- Coordinator is not directly assigned or created by User Management. Its assignment owner supplies authoritative account/role state through the existing Coordinator synchronization contract and requests the common setup event after a new Coordinator account/assignment transaction commits.
- Each event uses a stable account ID, registered email lookup, safe account-role context, and a unique occurrence ID. Producers do not name a delivery channel or pass a proof.
- Admin account DTOs exclude credentials and proofs. The public setup endpoint accepts `userId`, `proof`, and `newPassword` only through a dedicated request-only DTO; those values MUST NOT be returned, audited, or logged.
- The event adds a security template to the shared notification catalog. Delivery remains through the provider-neutral notification adapter; Development uses its configured allowlist and Production provider gate.

## Issuance and Reissue

1. Account owner commits the new Identity account, role, pending status, and required account audit event.
2. The owner requests `AccountCredentialSetup` after commit. A delivery failure cannot activate the account or roll back account creation.
3. The security-content provider verifies the target account still exists, is `PendingCredentialSetup`, and the registered email still matches the accepted destination.
4. The provider generates an Identity token with the dedicated purpose `PortalAccountCredentialSetup`, creates the setup URL, and returns transient template values to the notification dispatcher. The token, URL, and rendered body are never persisted or logged.
5. For an active-Admin reissue, verify pending status, rotate the security stamp to invalidate earlier tokens, and request a fresh event occurrence with a safe audit event. Do not route reissue through generic failed-notification resend.
6. If request persistence or email delivery fails, signal the failure and leave the account pending. Admin may request another fresh reissue while it remains pending.

## Completion and Activation

- The public setup page accepts the account ID, proof, and new password and posts to `POST /api/auth/account-credential-setups`.
- Verify the token against the dedicated purpose, pending account status, and current account security stamp. Password-reset proofs are not accepted for setup.
- Apply the existing password policy. Add the password to the passwordless Identity user, set status to `Active`, rotate the security stamp to consume/invalidate the setup token, stage a safe audit event, and commit these changes atomically.
- A wrong, expired, replayed, or stale proof makes no account change and returns a generic invalid-link response. Never include account-existence details or proof values in the response/log/audit.
- Pending accounts cannot sign in before successful setup. Existing password recovery remains generic and does not activate pending accounts.

## Test Evidence Required

- Generic setup issue and completion for each directly provisioned role; Teacher provisioning follows its existing owner path.
- Coordinator role is never assigned by User Management; verify the batch-assignment owner requests the common setup event for a newly provisioned Coordinator after its transaction commits. This owner integration remains a release gate until its producer test passes.
- Fresh reissue invalidates the old proof; only pending accounts can be reissued; Admin authorization and safe audit are enforced.
- Setup proof is purpose-separated from password reset; proof expiry/replay and password policy are enforced.
- Setup delivery failure leaves the account pending; reissue does not disclose secret material.
- Account status/password/audit complete or roll back atomically under validation and persistence failures.

## Implementation

- Event: `NotificationEventType.AccountCredentialSetup` (appended last, template ID `7a110000-0000-4000-8000-000000000008`, seeded by the
  `AccountCredentialSetupTemplate` migration with no schema change). Owners call `AccountCredentialSetupService.InitialRequest(userId)`; the
  occurrence is the user ID, so a replay is a duplicate. A reissue uses `{userId}:{guid}`.
- Token: `AccountCredentialSetupTokenProvider`, a `DataProtectorTokenProvider<PortalUser>` registered under the provider name
  `PortalAccountCredentialSetup` (also its Data Protection purpose) and used with the token purpose `PortalAccountCredentialSetup`. Its lifespan
  is `AccountCredentialSetup:ProofLifespan` (default 3 days), separate from `PasswordRecovery:ProofLifespan`.
- Link: `{AccountCredentialSetup:SetupPageUrl}?userId=...&proof=...`; an empty setting means `{Notifications:PortalUrl}/account-setup`.
- Provider refusals (notification Failed, nothing sent): `RecipientRemoved`, `DestinationChanged`, `AccountNotPending`.
- Completion removes a password left on a pending account by any other path before adding the new one, then sets Active and rotates the
  security stamp; `account.credential-setup.completed` names the account holder as actor. A refused attempt records the security event
  `auth.credential-setup.rejected` with only a reason (`invalid_link` or `password_rejected`) and no account.
- Password recovery no longer sends a reset link to a pending account: `PasswordResetContentProvider` refuses it with `AccountPendingSetup`,
  so recovery can never bypass setup and the public recovery response is unchanged.
- An email change rotates the security stamp, so a setup link sent to the old address stops working; the Admin reissues to the new address.
- Coordinator: no batch-assignment owner exists, so its producer test is a skipped release gate in `AccountCredentialSetupOwnerTests`.
