# Research: Role-Based Authentication and Access Control

**Date:** 2026-10-08
**Scope:** Resolve planning choices using the constitution-approved Angular 21/Node.js 24 LTS and .NET 10/C# 14/Identity 10/EF Core 10/PostgreSQL 18 stack while avoiding assumptions about unavailable repository internals.

## Workspace Evidence and Version Policy

**Decision:** Use the constitution-approved stack: Angular 21, Node.js 24 LTS, ASP.NET Core 10 Web API, C# 14, ASP.NET Core Identity 10, EF Core 10, PostgreSQL 18, and pgvector. Use the Npgsql EF Core provider 10.x compatible with EF Core 10. Major framework versions MUST NOT change without explicit project-level approval.

**Rationale:** The user-approved version set is authoritative. The current workspace contains no Angular manifests, .NET solution/project files, EF migrations, or test projects, so exact package patch versions, internal source paths, and test commands must be confirmed in the submodules. Keep every listed major version unless explicit project-level approval changes it.

**Alternatives considered:** Replacing any approved major version with a different or newer major was rejected because the constitution requires explicit project-level approval for major-version changes.

## Authentication Transport and Browser Integration

**Decision:** Use ASP.NET Core Identity for credential validation and a protected, `HttpOnly`, `Secure` browser cookie containing an opaque per-session identifier. Keep authentication state server-validated. Prefer same-origin hosting or a reverse proxy where deployment permits; if the Angular client and API use different origins, configure exact trusted origins, credentialed requests, and CSRF protection rather than wildcard CORS.

**Rationale:** The client is a browser application. HttpOnly cookies prevent ordinary application JavaScript from reading the credential. Same-origin routing reduces cross-origin configuration. Cookie-authenticated state-changing requests still require CSRF defenses. The cookie/session arrangement must preserve the specified no-automatic-expiration behavior; browser deletion or user sign-out can still remove access.

**Alternatives considered:** Client-readable bearer tokens were not selected because they increase exposure to script access and typical token expiration conflicts with the explicit no-timeout requirement. ASP.NET Core Identity's built-in bearer tokens are intended for simple cases and are not JWTs. A separate OIDC provider is outside the supplied scope unless the existing deployment already provides one.

**Sources:** [Identity for SPA API backends](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity-api-authorization?view=aspnetcore-10.0), [cookie authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/cookie?view=aspnetcore-10.0), [CSRF protection](https://learn.microsoft.com/en-us/aspnet/core/security/anti-request-forgery?view=aspnetcore-10.0), [CORS](https://learn.microsoft.com/en-us/aspnet/core/security/cors?view=aspnetcore-10.0).

## Role Assignment and Active Role

**Decision:** Model the supported role membership as a user-to-role many-to-many relationship using Identity's role support. Keep the active role in per-session server-side context, not as a user-wide preference. On every protected request, validate that the active role remains assigned and enforce role and record-level boundaries from the approved feature list.

**Rationale:** Users can hold several roles, yet the selected role must not inherit the union of all assigned-role capabilities. A per-session context prevents one browser's role switch from silently changing another session. Server-side assignment checks prevent stale role claims from retaining privileges after assignments change. Role membership supplies coarse authorization; course, assessment, batch, and student-record rules require policy/resource checks.

**Alternatives considered:** A single role field on the user cannot represent multiple roles. Trusting a role value supplied by the client or relying on Angular route guards is not an authorization boundary. Combining all role claims into OR-based role checks can grant the broader union prohibited by the spec.

**Sources:** [Identity model customization](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/customize-identity-model?view=aspnetcore-10.0), [role-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/roles?view=aspnetcore-10.0), [resource-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/secure-data?view=aspnetcore-10.0).

## Session Lifecycle and Password Reset

**Decision:** Use a minimal server-side authenticated-session record to support an opaque session handle, active-role context, individual revocation, and a sliding three-hour inactivity timeout. Initialize `LastActivityAt` at sign-in and update it on every authenticated request; reject and revoke a session when its last activity is at least three hours old. The browser cookie uses the same sliding lifetime. There is no separate maximum-age cutoff. Explicit logout revokes the current record. Password reset revokes only the session that completed the reset; other sessions remain subject to their own idle timeout.

**Rationale:** The user-approved idle-timeout and current-session-only reset rules are not provided by stock Identity defaults. Identity's security-stamp flow can invalidate other sessions after credential changes; using it unchanged can contradict the spec. Session-level state is the simplest dependable way to preserve role context, refresh activity, and selectively revoke one session. This requires a narrowly justified deviation from stateless cookie-only handling.

**Alternatives considered:** Account-wide security-stamp invalidation is simpler and conventional, but would revoke other sessions and violates the approved requirement. Client-only sign-out does not reliably revoke a copied credential. A fixed absolute session lifetime was not selected; active users retain their session while each authenticated request refreshes the three-hour idle window.

**Residual security risk:** A session can remain usable on an unattended or lost device for up to three hours after its last authenticated request. Required mitigations are a readily available explicit sign-out action, protected cookies, HTTPS, CSRF defenses, server-side authorization on every request, and secret-free security logging. Password reset intentionally does not revoke other sessions under the approved policy. This risk must remain visible in security review.

**Sources:** [Identity configuration and security stamps](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity-configuration?view=aspnetcore-10.0), [account confirmation and password recovery](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/accconfirm?view=aspnetcore-10.0), [cookie authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/cookie?view=aspnetcore-10.0), [ASP.NET Core 10 UserManager source](https://github.com/dotnet/aspnetcore/blob/v10.0.0/src/Identity/Extensions.Core/src/UserManager.cs).

## Password Recovery and Validation

**Decision:** Use Identity's reset-token generation/validation with email delivery to the account's recorded email. Return the same public confirmation for known and unknown email addresses. Enforce the specified minimum of eight characters with at least one letter and one number; do not add uppercase, symbol, or periodic-expiry rules absent from the spec. Rate-limit recovery requests and do not log reset proofs.

**Rationale:** Identity provides credential hashing and reset-token primitives. Its default password policy differs from the feature policy and must be explicitly aligned. Its documented recovery flow does not itself guarantee non-enumerating public responses; that response contract must be enforced by the application.

**Alternatives considered:** Custom credential storage or hand-built reset tokens were rejected because they duplicate security-sensitive Identity responsibilities. Stronger password rules may be recommended separately, but must not be silently added to this approved feature spec.

**Sources:** [Identity configuration](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity-configuration?view=aspnetcore-10.0), [account confirmation and password recovery](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/accconfirm?view=aspnetcore-10.0), [ResetPasswordAsync](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.identity.usermanager-1.resetpasswordasync?view=aspnetcore-10.0).

## Angular Experience

**Decision:** Use existing Angular form and styling conventions if present. Provide persistent labels, accessible validation messages, stable loading states, and generic authentication/recovery feedback. After sign-in, single-role users enter their only role context; multi-role users select from assigned roles. Switching is server-validated, refreshes role-specific data, and moves to an authorized landing page when needed. Route guards improve navigation only; the API remains authoritative.

**Rationale:** These behaviors follow the feature spec and design-system accessibility and consistency requirements. The actual frontend source is not checked out here, so no form library, component library, Tailwind adoption, route layout, or file structure is assumed.

**Alternatives considered:** A frontend-only role switch or authorization guard was rejected because it cannot secure direct API requests. Introducing a new UI dependency without source evidence was rejected under the constitution's dependency discipline.

**Sources:** [Angular form validation](https://angular.dev/guide/forms/form-validation), [route guards](https://angular.dev/guide/routing/route-guards), [accessibility](https://angular.dev/best-practices/a11y), [HTTP interceptors](https://angular.dev/guide/http/interceptors), [HTTP requests](https://angular.dev/guide/http/making-requests).

## Data and Migration Impact

**Decision:** Reuse the existing Identity user, role, and user-role assignment model where available. Add a logical authenticated-session entity with a non-reversible session-handle digest, user reference, active role reference, creation and last-activity timestamps, and revocation time/reason. Store no password or raw reset token in this entity. Persist reset proofs through the configured Identity token provider rather than inventing a separate reset-request table unless existing project behavior requires one.

**Rationale:** The session entity is needed for per-session active-role context and current-session-only revocation. The workspace does not show whether Identity tables or a session store already exist.

**Migration implication:** A version-controlled EF Core migration is required to add `LastActivityAt` to the existing `AuthSessions` table. Existing rows are backfilled from `CreatedAt`; apply the migration before deploying the updated service.

**Sources:** [Identity model customization](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/customize-identity-model?view=aspnetcore-10.0), [EF Core indexes](https://learn.microsoft.com/en-us/ef/core/modeling/indexes), [EF Core many-to-many relationships](https://learn.microsoft.com/en-us/ef/core/modeling/relationships/many-to-many), [EF Core migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/), [Npgsql EF Core provider](https://www.npgsql.org/efcore/).

## Version and Testing Decisions

**Decision:** Use the approved Angular 21/Node.js 24 LTS and .NET 10/C# 14/ASP.NET Core Identity 10/EF Core 10/PostgreSQL 18 stack. Angular's official compatibility table includes Node.js 24 for Angular 21. Npgsql EF Core provider 10.x supports EF Core 10; its 10.0 release notes document PostgreSQL 18 support, and version 10.0.3 is a stable compatible package at research time. Reuse existing test projects and commands. Add unit and API/integration coverage for sign-in, generic failures, non-enumerating recovery, password policy, role assignments/boundaries, role switching, logout, sliding three-hour session expiry, single-session reset revocation, and concurrent/session-stale authorization.

**Rationale:** The constitution pins the technology majors and explicitly prohibits unapproved major-version changes. The compatibility evidence supports the requested versions, but repository manifests are still needed to verify the exact patch versions, package lock, and runnable test commands.

**Sources:** [Angular version compatibility](https://angular.dev/reference/versions), [Npgsql EF Core 10.0 release notes](https://www.npgsql.org/efcore/release-notes/10.0.html), [Npgsql EF provider on NuGet](https://www.nuget.org/packages/Npgsql.EntityFrameworkCore.PostgreSQL), [EF Core providers](https://learn.microsoft.com/en-us/ef/core/providers/), [Angular testing](https://angular.dev/guide/testing), [ASP.NET Core integration tests](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-10.0).
