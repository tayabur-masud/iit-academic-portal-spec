# Research: Teacher Management

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Research date**: 2026-10-14

## Findings and Decisions

### User identity, pending state, and authentication

- **Finding**: `PortalUser` currently contains `DefaultRole` but no account lifecycle status. `PortalAuthenticationService.SignInAsync` validates password/lockout, while `ValidateAsync` validates a session and resolves current roles/default role; neither rejects pending or deactivated accounts.
- **Decision**: Add a persisted account status with `PendingCredentialSetup`, `Active`, and `Deactivated` states. Existing accounts migrate as `Active`; accounts created by Teacher Management start `PendingCredentialSetup`. Both password sign-in and validated-session authentication reject non-Active accounts. Credential setup transitions PendingCredentialSetup to Active in later email-mechanism work; this feature sends no email and offers no alternate credential handoff.
- **Rationale**: The current code has no status boundary, so a pending account would otherwise be able to sign in if a password were later set through another path. A shared user status also makes deactivation effective for every role on the account.
- **Alternatives considered**: Reusing `EmailConfirmed` was rejected because current authentication does not enforce it and it represents a different identity-verification fact. Identity lockout was rejected because it models temporary security lockout and could be cleared by normal lockout policy.

### Teacher profile and Identity roles

- **Finding**: ASP.NET Identity owns user credentials, normalized email, and role-assignment tables. `PortalRoles` defines Admin, Student, Teacher, and Coordinator. No Teacher profile, user-management service, or persisted batch-coordinator assignment exists.
- **Decision**: Add one `TeacherProfile` per `PortalUser`, linked by a unique, restrictive `UserId` foreign key. Keep profile data separate from Identity credentials. The profile contains display name, optional normalized employee/staff ID, optional phone, optional Designation, and active status. Use the existing `NameNormalizer` convention: names trim, collapse repeated whitespace, and compare using invariant case normalization; the optional employee/staff ID is unique across all teacher profiles when supplied.
- **Role decision**: Use `UserManager<PortalUser>` and the existing role catalog. An active teacher profile always has Teacher; Student is prohibited; Admin may be added/removed by an active Admin; Coordinator is not directly editable. The account default must be among assigned roles. If a removed role was the default, resolve a valid remaining default using the existing `PortalRoles.ResolveDefault` behavior.
- **Rationale**: Reuses Identity and its unique normalized email index, avoids duplicating password/role state, and follows the existing role-resolution policy.
- **Alternatives considered**: A second teacher credential store, a second role model, manually managed Coordinator role, and a separate user service were rejected.

### Admin authority and last-Admin invariant

- **Finding**: `PortalPolicies.AdminModule` is the active-session Admin authorization boundary. Current user provisioning in tests uses `UserManager`; no general user-management endpoint exists.
- **Decision**: Apply the existing Admin policy to every teacher route and mutation. Permit role changes only as specified above. Serialize Admin-role removal/deactivation checks with the mutation so concurrent operations cannot remove the last active Admin. Return a conflict without mutation if the invariant would be violated.
- **Rationale**: Hiding controls is not authorization, and an Admin-only feature must not strand the installation without an active administrator.
- **Alternatives considered**: Client-side role checks, client-supplied roles, or a best-effort count outside the write transaction were rejected.

### Identity writes, audit, and transaction boundary

- **Finding**: Identity `UserManager` operations persist user and role changes through the EF store as part of manager calls. Teacher provisioning and role changes span those saves, the TeacherProfile, session revocations, and a mandatory audit event.
- **Decision**: Use the request-scoped `PortalDbContext` and an explicit database transaction around Identity manager calls, TeacherProfile changes, session revocations, and `IAuditEventRecorder.Stage`/final save. Commit only after all steps succeed; roll back on validation, Identity, audit, or persistence failure. Serialize the last-Admin invariant check with the role/status mutation using an appropriate PostgreSQL transaction/locking strategy.
- **Rationale**: This preserves atomic business-change/audit behavior despite multiple Identity saves and prevents concurrent operations from removing the final active Admin.
- **Alternatives considered**: Independent saves without an encompassing transaction were rejected because they can leave an orphaned account/profile or unaudited role change. A custom role store or transaction framework was rejected as unnecessary.

### Deactivation, sessions, and audit

- **Finding**: `IAuthSessionRepository` revokes one session at a time and `IAuditEventRecorder.Stage` places business audit in the caller's unit of work without saving. `PortalDbContext` and existing configuration services use restrictive relationships and same-context saves.
- **Decision**: Deactivation sets TeacherProfile inactive and PortalUser status to Deactivated, retains both records and all Identity role assignments for history, and revokes every active session for the user through a bulk repository operation. Stage one allowlisted business audit event and save the profile, status, session revocations, and audit event atomically. Do not physically delete a user or profile.
- **Rationale**: Ensures deactivation blocks both future sign-in and existing sessions while preserving references and audit history.
- **Alternatives considered**: Deleting the Identity user, revoking only the current session, or relying only on the Admin menu were rejected.

### Coordinator synchronization integration

- **Finding**: The service has Coordinator role/policy support but no batch-coordinator assignment entity or workflow. The feature spec explicitly assigns batch assignment UI/data to another workflow.
- **Decision**: Add an internal application contract for the owning batch-assignment use case to report the authoritative post-change state (`hasActiveCoordinatorAssignments`) for a teacher. A `TeacherCoordinatorRoleSynchronizer` idempotently adds Coordinator when true and removes it when false; it never removes Teacher and reconciles the default role when needed. Do not add a public manual Coordinator role endpoint or invent a batch-assignment model here.
- **Integration limit**: Unit/API tests can verify the synchronizer contract. End-to-end assignment-trigger verification remains an integration gate for the future batch-assignment owner, which must call the contract in the same unit of work as its assignment change.
- **Alternatives considered**: A new batch-assignment table/workflow or unrelated message-bus dependency was rejected as out of scope and absent from the repository.

### Email scope

- **Decision**: Do not integrate email transport or perform setup-email delivery in Teacher Management. New accounts remain PendingCredentialSetup and cannot authenticate. Email invitation delivery, retry behavior, credential activation, and email-specific tests are deferred until the shared email mechanism is ready.
- **Rationale**: The user explicitly deferred email mechanism and email testing; a fake sender or unsafe Admin-set password would violate that scope and security boundary.

### API, UI, and testing patterns

- **Finding**: Configuration controllers use explicit DTOs, `AdminModule`, Problem Details, and 404/409/ValidationProblem mapping. Angular Admin navigation is role-filtered in `shell.ts`; routes are nested under `roleArea('Admin')`. API tests use xUnit/`WebApplicationFactory` with SQLite plus PostgreSQL 18 integration tests; frontend tests use Vitest/`HttpTestingController`.
- **Decision**: Reuse these conventions for `/api/admin/teachers`, `/admin/teachers`, typed DTOs, accessible directory/profile forms, and tests. Add PostgreSQL coverage for the migration/indexes and API coverage for account status and transaction invariants.
- **Rationale**: Avoids a new framework, dependency, or architecture pattern.

## Technical Goals and Open Inputs

- The spec sets no numeric performance SLO. Use bounded directory pagination and avoid loading unrelated Identity fields.
- No specification clarification remains open. The email mechanism and the batch-assignment producer are explicitly deferred/external dependencies, not implementation choices to guess.
