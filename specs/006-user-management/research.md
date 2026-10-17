# Research: User Management

**Date**: 2026-10-18  
**Feature**: [spec.md](spec.md)  
**Scope**: Resolve account-model, role/lifecycle, setup-email, API/UI, migration, and verification choices using the checked-out portal.

## Existing Identity and Account State

**Decision**: Reuse ASP.NET Core Identity's `PortalUser`, role assignments, normalized-email unique index, `DefaultRole`, and `AccountStatus`. Do not create a parallel account or role table. Use the existing `PendingCredentialSetup`, `Active`, and `Deactivated` states. The User Management list is an account projection; it does not own TeacherProfile or future academic profiles.

**Evidence**: [PortalUser.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Domain/Identity/PortalUser.cs), [TeacherAccountStatus.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Domain/Identity/TeacherAccountStatus.cs), and [PortalDbContext.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs) define and persist these fields. [PortalAuthenticationService.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/Authentication/PortalAuthenticationService.cs) and [PortalSessionAuthenticationHandler.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Api/Authentication/PortalSessionAuthenticationHandler.cs) deny sign-in and existing-session access unless the account is Active.

**Rationale**: The schema already contains the required account state and uniqueness rules. Duplicating Identity would create competing credentials and authorization state.

**Alternatives considered**: A new generic user/profile table was rejected because Teacher Management already owns a one-to-one TeacherProfile, while the current portal has no generic profile entity.

## Account-Type Ownership and Role Rules

**Decision**: The Admin account directory searches and inspects accounts with Admin, Student, Teacher, or Coordinator assignments. User Management creates generic Admin and Student Identity accounts. Teacher creation, profile update, role update, and deactivation delegate to the existing Teacher Management owner. Coordinator is never assigned directly by User Management; it remains derived from an authoritative batch-coordinator assignment. A Coordinator account is provisioned only through that assignment-owning workflow. Student accounts remain Student-only. Admin may assign or remove Admin on a Teacher while preserving Teacher and the last-active-Admin invariant. Every non-null default role remains assigned.

**Evidence**: [TeacherProvisioningService.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherProvisioningService.cs) creates the account and linked profile atomically. [TeacherRoleService.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherRoleService.cs) enforces Teacher-required, Student-prohibited, Admin-editable, Coordinator-derived, default-role, and last-Admin rules. [TeacherCoordinatorRoleSynchronizer.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherCoordinatorRoleSynchronizer.cs) defines the owner-facing Coordinator synchronization contract; no assignment producer currently calls it.

**Rationale**: This covers all supported account roles without creating duplicate TeacherProfile data or a second Coordinator authority.

**Alternatives considered**: Generic creation or role-edit endpoints for TeacherProfile fields or Coordinator were rejected because they would bypass existing ownership and synchronization invariants.

**External owner gates**: Student-specific profile/academic data and batch-coordinator assignment workflows do not exist in the current source. Account Management only owns generic Admin/Student Identity creation; Teacher Management owns Teacher provisioning. The batch-assignment owner creates Coordinator accounts/assignments and MUST request the common `AccountCredentialSetup` event after its account/assignment transaction commits. Its event-producer integration is a release gate and must be tested before Coordinator provisioning is considered complete. The first active Admin must be established by the existing bootstrap/deployment process because every management route is active-Admin-only.

## Transactions, Audit, and the Last-Admin Invariant

**Decision**: Use `UserManager<PortalUser>` over the existing request-scoped `PortalDbContext`, an explicit transaction, and the existing audit recorder. Account creation/status/role changes and their allowlisted audit events commit or roll back together. Any operation that could remove/deactivate the last active Admin must acquire the same PostgreSQL transaction advisory lock used by Teacher Management before checking and changing state. Reuse the existing session repository to revoke all sessions on deactivation.

**Evidence**: [TeacherStore.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/TeacherStore.cs) implements the lock with key `4_004_001`, transaction, and account queries. [TeacherProvisioningService.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherProvisioningService.cs) demonstrates Identity writes, audit staging, and one unit of work. [TeacherLifecycleService.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherLifecycleService.cs) shows session revocation and deactivation. [AuditEventRecorder.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/Auditing/AuditEventRecorder.cs) stages allowlisted events.

**Rationale**: Reusing the same lock prevents concurrent generic-account operations from racing with Teacher role removal/deactivation and removing the last active Admin.

**Alternatives considered**: A second lock key or an unlocked account count was rejected because it would not serialize with existing Teacher Management operations. Application-only checks were rejected as insufficient against concurrent requests.

## Credential Setup and Reissue

**Decision**: Implement one account-credential setup flow for all new account roles, distinct from password recovery. Use Identity's configured Data Protection token provider with a dedicated purpose such as `PortalAccountCredentialSetup`; verify the purpose-specific token, require `AccountStatus == PendingCredentialSetup`, add the password using Identity validation, activate the account, rotate its security stamp, and stage a safe completion audit event in the same transaction. Never persist the token/proof or rendered security content.

For email delivery, add a single `AccountCredentialSetup` security event/template to the shared notification catalog. Rewire Teacher provisioning to request this common setup event after its existing account/profile transaction; generic Admin/Student creation requests the same event. The batch-assignment owner must request the same event after committing a newly provisioned Coordinator account and authoritative assignment. The notification security-content provider re-checks pending status and the current registered destination before generating a fresh setup link in memory. Keep existing password-reset behavior and purpose separate.

An active Admin may reissue only for an account still pending. Reissue rotates the security stamp before requesting a new durable notification occurrence so prior setup tokens no longer validate; the notification request is audited without any token, address, or message body. Delivery failure leaves the account pending and reports a safe failure. Generic failed-notification resend is not used for setup reissue.

**Evidence**: [PasswordRecoveryService.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/PasswordRecovery/PasswordRecoveryService.cs) already uses Identity proofs but only for password reset, and it does not activate pending users. [DependencyInjection.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/DependencyInjection.cs) registers a password-reset provider plus unavailable Teacher/Student invitation providers. [NotificationDispatchProcessor.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Notifications/NotificationDispatchProcessor.cs) obtains security content just in time and never stores the rendered message. [NotificationTemplateCatalog.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/NotificationTemplateCatalog.cs) seeds event wording; no common setup event currently exists.

**Alternatives considered**: Reusing a password-reset token was rejected because credential recovery must not activate a pending account. Persisting setup proofs was rejected by the notification security-content rules. A new token table was not selected; Identity's purpose-specific protected token and security stamp are sufficient without new persistent state.

## API, UI, and Testing Patterns

**Decision**: Add a typed Admin account API and lazy-loaded Angular Admin feature using the existing active-role policy, DTO/Problem Details conventions, anti-forgery filter, design-system controls, and bounded list behavior. The account list may display TeacherProfile-backed fields only as a read projection; mutations of Teacher profile/email/roles/deactivation call or navigate to existing Teacher Management operations. Use existing xUnit `WebApplicationFactory`/SQLite tests, PostgreSQL-gated tests for lock/grants/migrations, and Angular/Vitest `HttpTestingController` tests.

**Evidence**: [TeachersController.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Teachers/TeachersController.cs), [TeacherDirectoryController.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Teachers/TeacherDirectoryController.cs), [PortalPolicies.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Api/Authorization/PortalPolicies.cs), [teacher-api.ts](../../iit-academic-portal/src/app/features/teachers/teacher-api.ts), [teacher-list.ts](../../iit-academic-portal/src/app/features/teachers/teacher-list.ts), and [app.routes.ts](../../iit-academic-portal/src/app/app.routes.ts) are the nearest patterns. Existing test fixtures include [PortalApiFactory.cs](../../iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Infrastructure/PortalApiFactory.cs) and [PostgresFixture.cs](../../iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Infrastructure/PostgresFixture.cs).

**Rationale**: These patterns preserve active-session authorization, API consistency, accessible UI behavior, and existing deployment boundaries.

## Migration and Release Impact

**Decision**: No new account columns or account tables are planned. Add a version-controlled EF migration to seed the `AccountCredentialSetup` notification template and initial revision; update the model snapshot. No new runtime tables or grants are required by this feature unless implementation discovers a missing Identity/notification schema element. If adding a new append-only table, runtime update/delete/truncate must be revoked.

**Release gates**: Supply and verify a publicly reachable setup URL; persist/share the Data Protection key ring across restarts; provision the first active Admin through deployment bootstrap; obtain Student and batch-assignment owners for any end-to-end profile/Coordinator integration; configure a Production email provider and verify its content-retention/access behavior against the notification privacy requirement.

**Testing decision**: Use `dotnet test` in `iit-academic-portal-service/`, PostgreSQL-gated tests with `AUDIT_TEST_POSTGRES`, and `npm test -- --watch=false` plus `npm run build` in `iit-academic-portal/`. No new package is needed.
