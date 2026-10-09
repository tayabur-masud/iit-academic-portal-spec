# Implementation Plan: Audit System

**Branch**: `feature/002-audit-system` | **Date**: 2026-10-09 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification at `specs/002-audit-system/spec.md`

## Summary

Add a reusable server-side audit capability to the existing IIT Academic Portal layered monolith. The capability defines a versioned common event contract, atomic persistence of mandatory business events in the same PostgreSQL transaction as their business change, durable retry delivery of security events through a PostgreSQL outbox, Admin-only review bound to the active session role, safe allowlisted change summaries, and no automatic archival or deletion without approved institutional policy. The feature provides the infrastructure and review experience; each owning business feature adds its own mandatory events and field allowlist when that feature is delivered. See [research.md](research.md) for decisions and tradeoffs.

## Technical Context

**Language/Version**: Angular 21/TypeScript with Node.js 24 LTS; ASP.NET Core 10/C# 14.

**Primary Dependencies**: Existing Angular libraries and shared design-system tokens; ASP.NET Core Identity 10; EF Core 10; Npgsql EF Core provider 10.x. Add no UI or queue package for the initial design. Use the existing hosted-service pattern to drain persisted security-event outbox records.

**Storage**: Existing PostgreSQL 18 database. Add formal audit-event and security-event outbox persistence through version-controlled EF Core migrations. Audit events are append-only to the runtime role; migration/DDL ownership is separate from the runtime application role.

**Testing**: Existing xUnit API tests through `dotnet test` and WebApplicationFactory/SQLite; add PostgreSQL-backed migration and permission verification. Existing Angular/Vitest tests through `npm test -- --watch=false`; production frontend build through `npm run build`.

**Target Platform**: Existing Angular browser application and ASP.NET Core Web API over HTTPS, preferably same-origin/reverse-proxy hosted. API and worker run in the service application; PostgreSQL remains the system of record.

**Project Type**: Web application, layered monolith with Angular presentation and ASP.NET Core Domain/Application/Infrastructure/API layers.

**Performance Goals**: Meet SC-006: at least 95% of first-page searches complete within three seconds over a representative dataset of 1,000,000 events (until the operations owner confirms another volume). Record the dataset size and test environment with the measurement. Keep indexes aligned with supported filters and newest-first keyset pagination.

**Constraints**: Business changes and their mandatory audit events commit in one PostgreSQL transaction. Security events do not alter primary authentication/authorization outcomes solely because audit-event persistence is delayed; persist them to a durable outbox and retain for retry/recovery. If outbox insertion itself fails, forward the safe event envelope to a separately configured durable operational-event sink and raise a protected critical alert; verify this sink before release. Do not reuse the existing in-memory channel as the durable path. Enforce review against the current session's active Admin role on every request. The runtime database role must not own the audit table and must lack UPDATE/DELETE there. No retention duration, automatic archive, or automatic deletion is introduced without approved policy. Raw credentials, cookies, reset proofs, and unapproved field values are excluded.

**Scale/Scope**: Reusable event contract and write path, Admin review search/detail, business transaction integration seam, security-event outbox/worker, indexes, monitoring, and operational recovery. The SC-006 baseline is 1,000,000 events until the operations owner confirms a volume. Retention duration is unspecified; preserve records indefinitely until policy approval.

## Clarified Decisions (2026-10-09)

These decisions come from the task review and are recorded in `spec.md` Clarifications; [research.md](research.md) has the rationale.

- **Denials**: Record HTTP 403 responses to authenticated sessions as `access.denied`. Anonymous 401 responses, including the portal's signed-in check, are counted in a metric and are not events.
- **Source**: Client IP from trusted forwarded headers only. Forwarded-header configuration becomes a documented deployment requirement.
- **Authentication events**: One `auth.sign-in` event per attempt. A failure records no account. Session revocation is `auth.session.revoked` with a reason of logout, idle timeout, or password reset. Password-reset completion and role switches have their own events.
- **Audit reads**: Each Admin search and detail view records `audit.review.accessed`.
- **Actor display**: Store only the stable ID; the API resolves the email when an event is viewed and returns no email for removed accounts.
- **Recording**: Explicit only. No automatic entity-change capture in the initial capability.
- **Database accounts**: A separate migrations connection string; `dotnet ef` uses the DDL account and the API uses the runtime account. Runtime grants cover the whole schema, with default privileges for future tables, and only SELECT and INSERT on audit events.
- **Fallback sink**: Development uses a JSON-lines file sink. In Production the service refuses to start without a configured durable sink.
- **Release gate**: US1, US3, and US4 must all be complete.

## Constitution Check

*Gate: checked before research and rechecked after design.*

- **I. IIT visual identity and academic usability — PASS**: Audit search/detail use the existing table, search, status, form, and accessibility patterns in `specs/design-system.md`; no feature-local tokens or component library are introduced.
- **II. Maintainable layered architecture — PASS**: Domain models event invariants; Application owns event contracts/use cases; Infrastructure owns EF persistence, the outbox worker, and database access; API controllers remain thin; Angular owns presentation only.
- **III. RESTful API contracts — PASS**: Review uses resource-oriented GET collection/detail endpoints with explicit filters, stable bounded pagination, and standard problem responses. No client event-write endpoint is exposed.
- **IV. Security and data integrity — PASS WITH RELEASE GATES**: Enforce active-Admin session authorization on every read; exclude secrets and non-allowlisted values; commit mandatory business events atomically; grant the runtime role append/read but not update/delete access to formal audit records; retain a durable security-event retry path and a separate durable fallback for outbox-write failure. The current Development connection uses the `postgres` account, so runtime/migration role separation and grant verification are mandatory before release. Application controls do not claim protection from privileged DBAs.
- **V. Reliable, simple, and observable software — PASS WITH OPERATIONS GATE**: Test rollback, retry/idempotency, role switching, mutation denial, search, and retention. The current in-memory queue is unsuitable. The outbox must survive process restarts, retain exhausted work for authorized recovery, and expose actionable monitoring/alerts. A durable operational-event sink and its alert destination must be wired and verified in deployment before release.
- **Data/access gate — PASS WITH MIGRATION REQUIRED**: Add audit/outbox persistence and required indexes by reviewed EF Core migration. Runtime access is granted by a separate non-owner database role; migration/DDL credentials are not used by the API at runtime.
- **Scope gate — PASS**: No business-domain workflows are implemented here; owning features add mandatory business-event calls and allowlists when delivered. No new role, separate audit service, retention period, archival mechanism, or deletion job is introduced.

### Post-Design Re-check

- **Architecture and transactions — PASS**: Mandatory business events share the existing database transaction with the feature's business mutation; no second database or distributed transaction is required.
- **Security — PASS**: Collection/detail API authorization derives only from the server-validated active role in the current session. UI visibility is not authorization. Security event payloads are allowlisted and delivery is idempotent by event ID.
- **Persistence — PASS WITH MIGRATION REQUIRED**: Formal event and security-outbox storage use PostgreSQL 18/EF Core 10. A DDL/migration principal owns schema objects; runtime grants are scoped so it cannot update/delete formal events. Verify grants using PostgreSQL, not SQLite alone.
- **Reliability — PASS WITH OPERATIONS GATE**: Persisted outbox rows retry across worker restart and audit-sink faults while PostgreSQL is available. Retry exhaustion remains recoverable and alerts. If the outbox insert itself fails, a separately configured durable operational sink retains the safe event envelope and alerts; verify this path before release. A total PostgreSQL outage is a broader portal availability incident because Identity/session validation already depends on that database.
- **UX/accessibility — PASS**: Admin-only search/detail follow the existing design system and keyboard/screen-reader table patterns.

## Project Structure

### Documentation (this feature)

```text
specs/002-audit-system/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── audit.openapi.json
├── checklists/
│   └── requirements.md
└── tasks.md              # Produced by /speckit-tasks, not this plan
```

### Source Code Repositories

```text
iit-academic-portal/
└── src/app/
    ├── core/auth/                 # Existing session and active-role state
    ├── features/audit/            # Admin audit search/detail feature
    └── layout/shell/              # Existing navigation integration

iit-academic-portal-service/
├── src/IitAcademicPortal.Domain/  # Audit event and change-summary invariants
├── src/IitAcademicPortal.Application/
│   ├── Abstractions/              # Internal event-writing boundary
│   └── Auditing/                  # Classification, validation, search use cases
├── src/IitAcademicPortal.Infrastructure/
│   ├── Persistence/               # EF mappings and migrations
│   └── Auditing/                  # Durable outbox processing and persistence adapters
├── src/IitAcademicPortal.Api/
│   └── Controllers/               # Read-only audit resource endpoints
├── database/postgresql/           # Runtime-role grants and operator recovery scripts
└── tests/IitAcademicPortal.Api.Tests/
    ├── Infrastructure/            # Existing WebApplicationFactory harness
    └── Auditing/                  # Focused audit behavior tests (new)
```

**Structure Decision**: Extend only the existing Angular and service submodules. Use Application abstractions for all event writes and searches, Infrastructure for EF/outbox/worker concerns, and an Admin-only API surface. Future business features call the shared application event contract within their own PostgreSQL unit of work. No separate repository or microservice is created.

## Complexity Tracking

| Added complexity | Why needed | Simpler alternative rejected because |
|---|---|---|
| PostgreSQL-backed security-event outbox and hosted worker | Security events must survive process restarts and transient audit-sink failure without blocking the primary authentication/authorization outcome. | Existing in-memory `PasswordRecoveryQueue` can drop work and loses state on restart; direct best-effort writes do not meet retry requirements. |
| Separate DDL owner and restricted runtime database role | The application must append/read but not update/delete audit records. | The current Development `postgres` connection is a superuser/table owner and cannot enforce this trust boundary. |
