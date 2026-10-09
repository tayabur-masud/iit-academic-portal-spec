# Implementation Plan: Role-Based Authentication and Access Control

**Branch**: `feature/001-role-based-auth` (all three repositories) | **Date**: 2026-10-08 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-role-based-auth/spec.md`

## Summary

Provide email/password authentication, registered-email password recovery, per-role authorization, and switching among a user's assigned Admin, Student, Teacher, and Coordinator contexts. Preserve the feature-list role/module boundaries and the design-system visual language. Use Angular 21/Node.js 24 LTS and ASP.NET Core 10/C# 14/Identity 10/EF Core 10 with PostgreSQL 18 and pgvector. See [research.md](research.md) for provider compatibility and session tradeoffs.

## Technical Context

**Language/Version**: Angular 21 and TypeScript on Node.js 24 LTS; C# 14 on .NET 10 / ASP.NET Core 10.

**Primary Dependencies**: Angular 21 using existing frontend libraries and styling conventions; ASP.NET Core Identity 10; EF Core 10; Npgsql.EntityFrameworkCore.PostgreSQL 10.x. Do not add a UI component library or Tailwind unless present or separately approved. *Update 2026-10-08: the project owner approved Tailwind CSS v4, themed from the design-system tokens, and Swagger UI for Development only.*

**Storage**: PostgreSQL 18 with pgvector, accessed through EF Core 10 and the compatible Npgsql 10.x provider. The specified per-session active-role/revocation behavior requires persistent session state; confirm the existing schema before implementation. A version-controlled EF migration is expected if this state is not already persisted.

**Testing**: Use the .NET 10 SDK with the service repository's existing unit and API/integration test framework, and Node.js 24 LTS with the frontend repository's existing test scripts. *Update 2026-10-08: the service uses xUnit (`dotnet test`) and the frontend uses Vitest through the Angular CLI (`npm test`).*

**Target Platform**: Angular 21 browser application communicating with an ASP.NET Core 10 Web API over HTTPS. Prefer same-origin hosting or a reverse proxy where deployment permits; otherwise configure exact trusted origins, credentials, and CSRF protection.

**Project Type**: Web application using the constitution's layered-monolithic architecture.

**Performance Goals**: No feature-specific numeric SLO is defined. Preserve existing application SLOs; ensure authorization reflects current role assignments, atomically refreshes per-session activity, and measure the cost of per-request session/role validation against the existing baseline.

**Constraints**: Use the constitution-pinned major versions; changing a major framework version requires explicit project-level approval. Email/password sign-in; students use their recorded personal email; replacement password minimum is 8 characters with at least one letter and one number; reset revokes only the current session; sessions expire after three hours without authenticated activity, with no separate maximum age. Enforce authorization on the server and reuse the design-system tokens. A session can remain usable on an unattended or lost device for up to three hours after its last authenticated request; include this residual risk in security review.

**Scale/Scope**: Four supported roles, single- and multi-role accounts, sign-in/out, recovery/reset, active-role selection/switching, and feature-list role/record boundaries. Account volume is not specified; use existing application scale assumptions rather than inventing a target.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **I. IIT visual identity and academic usability — PASS**: Authentication, role selection, loading, validation, error, and unauthorized states reference `specs/design-system.md`; no new brand styling is proposed.
- **II. Maintainable layered architecture — PASS**: Angular remains presentation; ASP.NET Core application/business logic remains separate from persistence; keep endpoints thin and reuse existing Identity infrastructure.
- **III. RESTful API contracts — PASS**: Proposed external contracts use resource-oriented session and password-recovery resources, semantically appropriate methods, consistent errors, and DTOs rather than persistence entities.
- **IV. Security and data integrity — PASS WITH DOCUMENTED RESIDUAL RISK**: Enforce server-side least privilege, current role-assignment checks, generic login/recovery responses, protected cookies, CSRF defenses, and secret-free logs. The user selected a three-hour inactivity timeout and current-session-only reset revocation. These decisions require persistent per-session activity state plus a custom Identity integration; retain the residual unattended-device risk in security review.
- **V. Reliable, simple, and observable software — PASS**: Add unit/API integration coverage for auth, assignment boundaries, role switching, reset, logout, session persistence, and denial cases. Use existing test frameworks and structured, secret-free security logging.
- **Data/access gate — PASS WITH MIGRATION REQUIRED**: Keep persistence behind the application boundary; use Identity role assignments and a minimal session record. Add the required `LastActivityAt` state through a reviewed, version-controlled EF Core migration.
- **Scope gate — PASS**: No account provisioning, additional roles, MFA/SSO, business modules, or feature-specific UI library is added.

### Post-Design Re-check

- **Architecture and REST — PASS**: The Angular client calls resource-oriented API contracts. Each request carries its protected session credential; the server validates it and current role membership for that request. Session state is modeled as a resource for active-role selection and individual revocation, not connection-bound conversational state.
- **Security — PASS WITH ACCEPTED RESIDUAL RISK**: The server validates session revocation, active-role assignment, and record boundaries; authenticated requests refresh a three-hour inactivity window. A session can remain exposed on an unattended/lost device for up to three hours after its last activity; mitigate with protected cookies, CSRF defenses, prominent logout, and security review. Reset deliberately leaves other sessions active.
- **Persistence — PASS WITH MIGRATION REQUIRED**: EF Core 10 with the Npgsql 10.x provider supports PostgreSQL 18; `SessionIdleTimeout` adds and backfills the required per-session activity timestamp.
- **UX and accessibility — PASS**: The OpenAPI outcomes and quickstart scenarios map to the shared design-system states; no new visual tokens or component library are introduced.

## Project Structure

### Documentation (this feature)

```text
specs/001-role-based-auth/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── contracts/
│   └── authentication.openapi.json
├── quickstart.md
├── checklists/
│   └── requirements.md
└── tasks.md              # Created by /speckit-tasks, not this plan
```

### Source Code Repository Boundaries

Implement this feature within the existing Angular frontend and ASP.NET Core service submodules. Preserve the layered-monolithic architecture and each repository's established internal structure. Do not create new repositories or assume source/test subfolders that are not present in this planning workspace.

```text
iit-academic-portal/             # Existing Angular frontend submodule
iit-academic-portal-service/     # Existing ASP.NET Core Web API submodule
```

**Structure Decision**: Use the two existing submodule repositories and the constitution's layered-monolithic architecture. UI work MUST follow `specs/design-system.md`.

*Update 2026-10-08:* the submodules contained only README files, so the project owner approved scaffolding this layout:

```text
iit-academic-portal-service/
├── IitAcademicPortal.slnx
├── src/IitAcademicPortal.Domain/          # entities and rules with no infrastructure
├── src/IitAcademicPortal.Application/     # use cases: sessions, recovery, policies
├── src/IitAcademicPortal.Infrastructure/  # EF Core DbContext, migrations, email, queue
├── src/IitAcademicPortal.Api/             # thin controllers, auth handler, policies
└── tests/IitAcademicPortal.Api.Tests/     # xUnit API and unit tests
iit-academic-portal/
└── src/app/{core,features,layout,pages}   # Angular 21; tokens in src/styles/tokens.css
```

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Per-session server-side validation, sliding idle expiry, and revocation | The feature requires a three-hour inactivity timeout, active role per session, and password reset to revoke only the current session. | Stock Identity security-stamp validation can invalidate other sessions after a password reset; client-only logout cannot reliably revoke a copied credential or enforce activity-based expiry. Use the smallest session record/validation mechanism compatible with the existing Identity store. |
