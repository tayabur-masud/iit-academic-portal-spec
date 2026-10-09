# Implementation Notes: Role-Based Authentication and Access Control

**Feature**: [spec.md](spec.md) | **Tasks**: [tasks.md](tasks.md) | **Date**: 2026-10-08

## Source baseline (T001–T003)

Both submodules contained only a README when implementation started. The project owner approved
scaffolding new projects instead of waiting for source. The owner also approved re-pointing
`iit-academic-portal-service` from the frontend repository URL to
`https://github.com/tayabur-masud/iit-academic-portal-service.git`.

| Area | Path | Versions |
|---|---|---|
| Frontend | `iit-academic-portal/` (Angular CLI workspace, `src/app`, Vitest specs beside sources) | Angular 21.2, TypeScript 5.9, Vitest 4, Node.js 24.19 |
| Service | `iit-academic-portal-service/IitAcademicPortal.slnx` (`src/` Domain, Application, Infrastructure, Api; `tests/IitAcademicPortal.Api.Tests`) | .NET SDK 10.0.401, ASP.NET Core / Identity / EF Core 10.0.12, Npgsql EF provider 10.0.3, xUnit 2.9 |

No major version deviates from the constitution. pgvector is not referenced because this feature stores no vectors.

## Implementation decisions

| Decision | Reason |
|---|---|
| Custom `PortalSession` authentication handler over an `AuthSessions` table, not the Identity cookie | Per-session active role, single-session revocation, and a sliding three-hour inactivity timeout. Identity's security-stamp cookie validation would revoke other sessions after a password reset. |
| Only the active role is issued as a role claim | Standard `RequireRole` policies then cannot grant the union of a multi-role user's roles. |
| Active role re-checked against current assignments on every request (one query joins the session and the user's roles) | Removed assignments stop authorizing on the next request (FR-007, edge case "role assignment changes"). |
| Recovery requests are queued and processed in the background | The response is identical in content and in timing for known and unknown emails (FR-015). |
| Recovery proof lifetime defaults to **1 hour** (`PasswordRecovery:ProofLifespan`) | The spec says "short-lived" without a value. **Confirm this value with the project owner.** Proofs are single-use because Identity rotates the security stamp on reset. |
| Unknown email at sign-in is verified against a dummy hash | The failure response and its timing match those for a wrong password. |
| Signing in again from the same browser revokes the browser's previous session | A replaced cookie would otherwise leave an orphaned session until its idle timeout elapsed. |
| Authentication and recovery endpoints are rate-limited to 10 requests per minute per client address (configurable) | The contract declares 429. The spec gives no limit, so the default is an assumption. |
| Account lockout is not enabled | The spec does not ask for it. Lockouts that an administrator sets on an account are still honored. |
| Record boundaries are expressed as `IStudentOwnedRecord`, `ITeacherAssignedRecord`, and `ICoordinatorAssignedRecord` with a `RecordAccess` policy | Course, batch, and result entities are out of scope. Future modules implement these interfaces, and the tests use stand-in records. |
| Role landing pages show an empty state | Module content is out of scope (spec "Out of Scope"). |
| Tailwind CSS v4 (approved by the project owner 2026-10-08), themed from the design tokens with the default palette removed | The plan allowed Tailwind only with approval. The tokens stay the single source, and the build rejects colors outside the design system. |
| Swagger UI (Swashbuckle UI only) over the built-in `/openapi/v1.json`, Development only, opened by the launch profiles | Requested by the project owner. It runs before authorization so the page is public, while every API endpoint keeps the secure-by-default policy. A request interceptor adds the anti-forgery header. |
| IIT logo (`public/images/iit-logo.png`, 600×327) in a shared, centered `AuthBrand` header on every focused screen (sign-in, forgot password, reset password, not found): 80px tall at its original aspect ratio, text alternative "IIT, University of Dhaka", with the portal name below | The project owner supplied the asset and asked for centered placement on 2026-10-08. The placement is now specified in design-system §10. |
| Stored default role per account (`AspNetUsers.DefaultRole`); sign-in enters it directly, with the fallback order Admin, Coordinator, Teacher, Student when it is missing or unassigned; the role-selection page is removed | Project-owner decision on 2026-10-09. A removed active role also falls back to the default among the remaining roles rather than leaving the session without a role. Administrators will change defaults through user management (later). |
| Favicon (`favicon.ico`, 16/32/48px) and 180px `apple-touch-icon.png` generated from the logo; this replaces the Angular default icon | Requested by the project owner on 2026-10-08. The browser-tab icon uses only the "IIT" lettermark on a white rounded tile, because the "University of Dhaka" bar is unreadable below 48px. The logo is not recolored or stretched (design-system §10). |
| Development-only account seeder, gated on `DevelopmentSeed:Password` | The quickstart needs test accounts, and provisioning is out of scope. No password is stored in source. |

## Test and build results (T026)

| Command | Result |
|---|---|
| `dotnet test` (service) | 54 passed, 0 failed (2026-10-09, after the idle-timeout and default-role changes) |
| `npx ng test --watch=false` (frontend) | 28 passed, 0 failed (2026-10-09, after the role-selection page was removed) |
| `npx ng build` (frontend) | Succeeded with no budget warnings; initial bundle 332 kB raw / 85 kB transferred, including Tailwind styles of 17 kB raw / 3.5 kB transferred |
| `dotnet run` (Api, https profile) | Boots without a database. The anti-forgery endpoint, a 401 problem on `GET /sessions/current`, and a 400 problem for sign-in without a CSRF token were all checked. |

Mutation checks confirmed that the tests catch real defects:
- Removing the server-side revocation filter fails the logout and reset-revocation tests.
- Removing the anti-forgery filter fails all four anti-forgery tests.

The service tests use in-memory SQLite through `WebApplicationFactory` because no PostgreSQL or Docker
daemon was available.

Quickstart mapping:

| Quickstart scenario | Automated coverage |
|---|---|
| 1 Single-role sign-in | `SignInAndSignOutTests`, `Login` spec |
| 2 Multi-role default and switching | `ActiveRoleTests`, `DefaultRoleTests`, `Login` and `Shell` specs |
| 3 Direct unauthorized access | `AuthorizationBoundaryTests` |
| 4 Generic sign-in failures | `SignInAndSignOutTests` |
| 5 Recovery enumeration | `PasswordRecoveryTests` |
| 6 Replacement-password validation | `PasswordRecoveryTests`, `PasswordPolicyTests`, `ResetPassword` spec |
| 7 Session reset semantics | `PasswordRecoveryTests` |
| 8 Three-hour inactivity timeout | `SignInAndSignOutTests` (authenticated activity refreshes the window; a session idle past three hours is revoked) |
| 9 Logout and browser navigation | `SignInAndSignOutTests`, plus `Cache-Control: no-store` on API responses |
| 10 Loading, validation, accessibility | `Login`, `ForgotPassword`, and `ResetPassword` specs |

Run against PostgreSQL 18.3: the migration is applied, the development accounts are seeded, and sign-in has created
sessions.

**Still to run manually:** the full browser pass of quickstart scenarios 1–10.

## Accessibility review (T027)

Reviewed against WCAG 2.2 AA and design-system §6, §7, §10, §12, and §13 by reading the code:

- Inputs have persistent labels.
- Errors are text, linked with `aria-describedby`, and set `aria-invalid`.
- Field errors start with a decorative error icon instead of a bare "!" (changed 2026-10-08 at the project owner's request; design-system §7).
- Alerts use `role="alert"`, and progress is announced with `role="status"`.
- Submit buttons are disabled while a request is pending.
- Focus moves to the first invalid field.
- Focus is always visible (`:focus-visible`).
- A skip link is provided.
- Active navigation is marked with weight, an indicator bar, and `aria-current`.
- Status is never shown by color alone.
- Animation stops under reduced-motion preferences.
- Targets are at least 44px tall.

Fixed during review:
- The role switcher was 40px tall. It now uses the standard 44px control height.
- The password-rule list re-announced on every keystroke. It is now read through `aria-describedby`.

**Still open:**
- Visual check at each breakpoint and at 200% zoom.
- Screen-reader pass (design-system §17 items 1–2).

## Security review (T028)

| Control | Status |
|---|---|
| Session cookie | `__Host-iit-session`: `HttpOnly`, `Secure`, `SameSite=Strict`, `Path=/`, no `Domain`. Opaque 256-bit handle; only its SHA-256 digest is stored. |
| CSRF | ASP.NET Core antiforgery is required on every non-safe method, including anonymous sign-in and recovery, through the `X-CSRF-Token` header. `SameSite=Strict` adds defense in depth. |
| Current-role checks | On every request. Removing a role assignment takes effect immediately (tested). |
| Logout and reset revocation | Server-side and limited to one session (tested with a replayed credential). |
| Account enumeration | Identical sign-in failures and identical recovery responses, with background processing and dummy-hash timing. A small timing difference remains on `POST /password-resets` between an unknown email and a known email with a bad proof; it is low risk because proofs cannot be guessed. |
| Logging | Logs include user IDs, session IDs, and roles only. No emails, passwords, cookies, handles, or proofs. The recovery worker logs only the exception type. |
| Error responses | `ProblemDetails` everywhere, with no stack traces outside Development. Responses use `Cache-Control: no-store`. |
| Transport | HTTPS redirection, plus HSTS outside Development. |

**Accepted residual risk: up to three hours of idle exposure.** Per FR-013, authenticated requests refresh the
sliding inactivity window; a stolen or unattended device can remain usable until three hours after its last
authenticated request. A password reset does not revoke the user's other sessions (FR-018), so resetting does
not immediately lock out an attacker who already holds another active session.

Mitigations in place:
- Sign-out is in the account section at the bottom of the sidebar (design-system §9). It is always visible on wide screens and one tap away in the mobile menu. Moved from the header on 2026-10-09 at the project owner's request.
- The cookie is protected.
- CSRF protection is in place.
- Authorization is checked on every request.
- Diagnosis logs record no secrets.

This risk needs explicit sign-off at release. An administrator action to revoke all sessions is the natural
next control, but it is out of scope for this feature.

Deployment requirements:
- Persist the Data Protection key ring. Recovery proofs and anti-forgery tokens depend on it.
- Configure forwarded headers behind a proxy so rate limiting sees client addresses.
- Serve the frontend and API from the same origin.
- Keep connection strings and SMTP credentials out of source.

## Contract comparison (T029)

Every path, method, and schema in `contracts/authentication.openapi.json` is implemented as specified.

Differences, now recorded in the contract:
- `DELETE /sessions/current` can return 400 (anti-forgery failure).
- `GET /sessions/current` re-issues the session cookie.
- A successful reset clears the cookie of the session that performed it.

Other behavior:
- `POST /sessions` also returns `Location: /api/auth/sessions/current`.
- Validation errors use `ValidationProblemDetails` (with `errors`), which is a superset of `ProblemDetails`.
- 403 responses for module and record denial come from future protected endpoints, not this contract.

## Database (T030)

- Initial migration: `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/…_InitialIdentityAndSessions`.
- It creates the Identity schema, the `AuthSessions` table, and the seeded role catalog.
- It adds a unique `EmailIndex` on `NormalizedEmail`, a unique `HandleDigest`, and a cascading `UserId` foreign key.
- `dotnet ef migrations script --idempotent` produces valid PostgreSQL DDL.

Npgsql 10.0.3 documents PostgreSQL 18 support. This is a new database, so there are no existing duplicate
normalized emails to resolve.

**Applied 2026-10-08** to the local development database on **PostgreSQL 18.3** with `dotnet ef database update`.
The development accounts were then seeded, and sign-in has written `AuthSessions` rows. `dotnet ef` reads the API
project's configuration (`appsettings.Development.json` plus user secrets); there is no separate design-time
connection string.

## Session timeout update (2026-10-09)

- Implemented a sliding three-hour inactivity timeout; each authenticated request updates `LastActivityAt` and reissues the cookie with a matching expiry.
- Added API coverage for refreshing activity and revoking a session after three hours idle. Focused `SignInAndSignOutTests` passed: 9 passed, 0 failed, using an isolated artifacts directory because the running Debug API holds its output assemblies open.
- Generated `20261008204920_SessionIdleTimeout`, which adds `LastActivityAt` and backfills existing rows from `CreatedAt`.
- **Applied 2026-10-09** to the local PostgreSQL database. `dotnet ef migrations list` confirms both `InitialIdentityAndSessions` and `SessionIdleTimeout` are applied.
- Restart the running API with the updated build before using it: the existing Debug process predates `LastActivityAt` and cannot create sessions against the migrated schema.
- The full backend test suite and migration-script generation passed. Manual browser scenarios and security/release sign-off remain outstanding.
