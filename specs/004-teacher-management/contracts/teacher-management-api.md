# Teacher Management API Contract

**Status**: Planning contract for [spec.md](../spec.md). The implementation publishes the generated OpenAPI document through the existing ASP.NET Core OpenAPI setup.

## Security and Common Behavior

- All routes require an authenticated session whose active role is Admin through `PortalPolicies.AdminModule`; non-Admin active roles receive 403 and anonymous callers receive 401.
- State-changing requests require the existing `X-CSRF-Token` anti-forgery header.
- Use explicit request/response DTOs; do not expose Identity entities, password hashes, or secrets.
- Validation errors use field-keyed Problem Details. Duplicate email or employee/staff ID conflicts do not create partial users or profiles.
- Successful business changes stage allowlisted audit events in the same `PortalDbContext` unit of work.

## Routes

| Method and route | Behavior |
|---|---|
| `GET /api/admin/teachers` | Search/list teachers. Query: `search` (name, email, or staff ID, case-insensitive), `status` (`Active`, `PendingCredentialSetup`, or `Deactivated`; absent = all), `page` (1-based, default 1), `pageSize` (1 to 100, default 25). The Angular directory requests bounded pages sequentially and appends them with infinite loading; it does not show page numbers, totals, or a page-size selector. The numbered `page` response `{ items, page, totalCount, totalPages }` remains available to non-UI API clients; a page past the end returns no items. Invalid values return 400 with a field error. |
| `POST /api/admin/teachers` | Body: `displayName` (required), `email` (required, valid, unique), optional `employeeStaffId`, `phone`, `designationId` (an active Designation). Unknown fields (including `isActive` and `roles`) are rejected with 400. Create a TeacherProfile and one linked PortalUser in PendingCredentialSetup; assign Teacher and set Teacher as default. Do not send email or set a password. Return 201 with a safe response and `Location`. |
| `GET /api/admin/teachers/{teacherId}` | Return `id`, `userId`, `email`, `displayName`, optional `employeeStaffId`/`phone`/`designationId`/`designationName`, `isActive`, `accountStatus`, `roles` (canonical order), `defaultRole`, and `createdAtUtc`/`createdByUserId`/`createdBy`/`lastModifiedAtUtc`/`lastModifiedByUserId`/`lastModifiedBy`. Never return secrets. |
| `PUT /api/admin/teachers/{teacherId}` | Replace `displayName`, `email`, `employeeStaffId`, `phone`, `designationId`; optional `isActive`. Enforce Identity email and staff-ID uniqueness (a staff ID is unique across active and inactive profiles). `isActive: false` deactivates exactly as the deactivation route does, in the same transaction; `isActive: true` on a deactivated teacher is rejected (400, `isActive`) because reactivation needs the later credential-setup flow. A request that changes nothing records no audit event. |
| `POST /api/admin/teachers/{teacherId}/deactivation` | No body. Make the profile inactive, set the account to Deactivated, and revoke every active session, in one transaction. Roles, profile, and history are kept; there is no delete route. Repeating it returns 200 and changes nothing. |
| `PUT /api/admin/teachers/{teacherId}/roles` | Body: `roles` (the editable roles wanted: `Teacher`, and `Admin` if wanted) and optional `defaultRole`. Reject Student, Coordinator, unsupported names, and a missing Teacher while the profile is active (400, `roles`); reject a `defaultRole` that is not among the resulting assigned roles (400, `defaultRole`). A held Coordinator role is kept untouched and counts toward the default-role check. If the default is no longer assigned it is reset with `PortalRoles.ResolveDefault`. The roles of a deactivated teacher cannot be changed (409). Coordinator is derived by the internal batch-assignment synchronization contract. |

Active Admin means an account with the Admin role whose status is Active. A request to remove Admin or deactivate a teacher that would leave no active Admin accounts returns 409 without changing role, status, sessions, or audit history. A deactivated teacher account is not physically deleted. Pending and deactivated accounts cannot authenticate.

## Provisioning and Deferred Email

Teacher creation creates a pending account only. This feature does not attempt password-setup email delivery and provides no alternate credential handoff. The user remains unable to sign in until a future email/credential-setup integration transitions the account to Active. Email delivery, retries, and related end-to-end tests are explicitly deferred until the shared email mechanism is ready.

## Coordinator Role Synchronization

The batch assignment workflow calls an internal application service after an assignment mutation with the authoritative teacher user ID and whether any active coordinator assignments remain. This is not a public endpoint. The service adds Coordinator when assignments exist and removes it when none remain, without removing Teacher or violating the default-role invariant.
