# Research: Audit System

**Date**: 2026-10-09  
**Feature**: [Audit System](spec.md)  
**Scope**: Resolve architectural and operational questions needed to plan reusable audit recording and review in the existing IIT Academic Portal.

## Existing Stack and Boundaries

**Decision**: Keep the implementation in the existing layered monolith: Angular 21/TypeScript frontend, ASP.NET Core 10/C# 14 API, EF Core 10/Npgsql 10.x, and PostgreSQL 18. Reuse the existing Angular feature structure, `PortalDbContext`, application/repository boundary, authenticated session authorization, hosted-service pattern, and xUnit test project. Do not create another application or service.

**Evidence**: `iit-academic-portal-service/README.md` describes Domain, Application, Infrastructure, and API projects. `Infrastructure/DependencyInjection.cs` registers Npgsql `PortalDbContext`, repositories, and a hosted recovery worker. `PortalSessionAuthenticationHandler` loads the current session; `PortalClaims` carries the active role. `PortalPolicies` has the authenticated fallback and Admin module policy. Angular uses `src/app/features`, the shared shell, and the centralized design tokens.

**Rationale**: Existing boundaries keep persistence and audit policy out of controllers and UI components. Audit writes can use application-level services and existing EF Core transactions.

**Alternatives considered**: A separate audit service or repository would duplicate the current application structure and is out of scope. Frontend-generated audit events are not trusted because clients can omit or forge them.

## Event Classification and Ownership

**Decision**: Preserve the three classes defined in `spec.md`: mandatory business events, security events, and optional diagnostic events. The audit capability owns the common event contract, recording paths, and protected review. Each owning feature declares and implements its mandatory business event types and approved before/after field allowlist when that feature is delivered. The initial security event catalog includes authentication successes/failures, password-reset outcomes, session creation/revocation and logout, active-role switch outcomes, access-control denials, and audit-record mutation attempts.

**Rationale**: This separates the reusable system from course, fee, enrollment, elective, result, account, and coordinator workflows that are not built by this feature. Input-validation errors and transient technical failures are not formal business events; access-control denials remain security events.

**Alternatives considered**: Implementing events for all listed business domains now would violate the clarified feature scope. Recording every request or database operation would create noisy, low-value audit history.

## Atomic Mandatory Business Events

**Decision**: For a business action already integrated with audit, persist the business update and its mandatory `AuditEvent` in the same PostgreSQL transaction through the existing EF Core unit of work/`PortalDbContext`. Save both or neither. Each future feature's integration point must enlist its audit event in the same transaction as its owned data change.

**Rationale**: Both records are in the same PostgreSQL database, so one local transaction is sufficient. It avoids a distributed transaction and prevents committed business changes with missing required audit history.

**Alternatives considered**: A separate audit database, cross-database transaction, or asynchronous business-event write was rejected for the mandatory business path because it can leave the business change and its required audit record inconsistent. EF Core's normal `SaveChanges` transaction may be reused when both writes share the same context; explicit transaction handling is needed only when an existing feature spans multiple saves or other resource boundaries.

## Security Event Delivery and Retry

**Decision**: Use a PostgreSQL-backed `SecurityAuditOutbox` as the initial durable retry mechanism and a hosted worker to deliver pending security events into the append-only audit-event store. The request path records a safe event envelope with a stable event ID; the worker persists the audit event and acknowledges the outbox item only after successful persistence. Retries use bounded backoff, retain exhausted items for authorized recovery, raise protected alerts, and do not silently discard them. Consumers are idempotent on the stable event ID so a replay cannot create a second formal event. If the outbox write itself fails, emit the safe event envelope through a separately configured durable operational-event sink and raise a critical alert; that fallback sink is a deployment prerequisite because none is configured in the current application.

**Rationale**: The backend already uses PostgreSQL and ASP.NET Core hosted services, so a database-backed outbox is durable across worker/process restarts and can retry when the audit-event sink or consumer is failing while PostgreSQL remains available. It adds no queue package or separate database. Authentication, sessions, and authorization already depend on PostgreSQL; a complete database outage is therefore a broader portal availability incident, not an audit-only failure. The separate durable operational sink is needed for events already decided by the request path when the outbox itself cannot accept them. Monitoring and authorized recovery remain required.

**Evidence and constraint**: The existing `PasswordRecoveryQueue` is a bounded in-memory `Channel<string>` with capacity 500. A full queue logs a warning and drops work; processing errors are logged and swallowed. It is not suitable as the only security-event delivery mechanism.

**Alternatives considered**:

- Reusing the in-memory channel was rejected because it loses events on process restart and can drop work when full.
- A separate message broker could provide an independent failure domain, but no broker, deployment target, or operations integration is configured in either application submodule. Introducing one would add an unapproved infrastructure dependency.
- A local disk spool was rejected as the default because no persistent shared-volume, multi-instance, or backup contract exists in the current deployment documentation.

**Operational gate**: If deployment requirements later demand security-event retention during total PostgreSQL unavailability, the plan's same-database outbox is insufficient. That stronger failure-domain guarantee requires an approved durable external broker or equivalent platform service, selected with the deployment owner before implementation; it is not assumed here.

**Release gate**: The implementation cannot claim full FR-014 conformance until deployment configures and verifies a durable operational-event sink for outbox-write failures, with a protected alert and authorized recovery procedure. Console logging alone is not sufficient evidence of durable retention.

## Session-Scoped Audit Review

**Decision**: Expose only read operations for search and event detail. The service checks the authenticated session's current active role on every request and allows access only when that role is Admin. A multi-role account in Teacher or Coordinator context is denied; switching to Admin permits the next request. Angular navigation is presentation only.

**Rationale**: The existing authentication system issues only the active role as a role claim. Server policy enforcement follows the same least-privilege boundary and does not add an auditor role or permission.

**Alternatives considered**: Checking whether an account has any Admin assignment, trusting a client-submitted role, or granting all users with other roles access were rejected because they ignore the current session-scoped context.

## Append-Only Database Boundary

**Decision**: Store formal audit events in a PostgreSQL table whose owner is a migration/DDL principal, not the runtime application role. Grant the runtime role only the required `SELECT` and `INSERT` rights on that table. Keep outbox delivery-state permissions separate from formal event immutability. Apply the same authorization checks to all review endpoints and expose no event update/delete operation.

**Rationale**: PostgreSQL table owners retain elevated control, so revoking `UPDATE`/`DELETE` from a role that owns the table does not establish the intended boundary. The current development connection uses `postgres` and migrations use the API project's connection configuration; this must be separated before the least-privilege database gate can pass. Grant provisioning is environment-specific and belongs in deployment/database setup, not hard-coded into application code.

**Alternatives considered**: Application-only append-only behavior was rejected as insufficient for the requested database-role boundary. Hash chains, external tamper evidence, and immutable archive storage remain optional enhancements; the design does not claim resistance to a privileged DBA or superuser.

## Search Contract and Experience

**Decision**: Provide `GET /api/audit-events` for filtered collection search and `GET /api/audit-events/{eventId}` for read-only detail. Require the active Admin session on both. Support UTC start-inclusive/end-exclusive bounds, actor, action type, category, entity type/ID, outcome, and correlation ID. Use bounded keyset pagination ordered by `(occurredAtUtc DESC, eventId DESC)`, default 25 and maximum 100. Return user-safe problem details for invalid ranges, unauthorized access, and missing events.

**Rationale**: Resource-oriented GET operations match existing REST conventions. Keyset pagination with a stable timestamp/ID pair handles concurrent inserts more predictably than offset paging. Existing design-system table, search, empty/error/loading, and accessibility rules apply.

**Alternatives considered**: Public create/update/delete endpoints were rejected; features write via the trusted server-side event contract. Client-side filters or route visibility are not authorization. Offset pagination was not selected because concurrent inserts can shift page boundaries.

## Retention, Archival, and Indexing

**Decision**: Initial delivery has no automatic deletion or archival. No retention duration is configured until an institutional policy is approved and applied through an authorized configuration process. Design indexes for UTC ordering and required filters based on actual query patterns. If archival is introduced later, it remains retrievable through authorized review and is distinct from deletion.

**Rationale**: The user explicitly supplied no retention period. A technical default would invent institutional policy. Searchable review history must not silently disappear after archival.

**Alternatives considered**: A guessed time-to-live, automatic archival, and deletion coupled to a technical database default were rejected. Archival platform and schedule remain future policy decisions.

## Testing Decisions

**Decision**: Reuse xUnit API tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests`, which host the real API pipeline using in-memory SQLite. Add PostgreSQL-specific integration checks for migration application and runtime database-role grants because SQLite cannot validate PostgreSQL grants or provider-specific SQL. Use existing Angular/Vitest tests for Admin-only navigation and search/detail UI. Use the documented `dotnet test` and `npm test -- --watch=false` commands.

**Required coverage**: Same-transaction business rollback; security outbox retry/idempotency and exhaustion/recovery; unchanged authentication/authorization results on audit-sink failure; secret redaction; Admin active-role access and denial after role switching; record append-only grants; all search filters and pagination; no deletion/archive without approved policy; and table accessibility/responsive behavior.

**Open planning dependency**: The operations owner must define the representative event volume used by SC-006 and provision an application runtime database role distinct from the migration owner before PostgreSQL deployment validation.

## Decisions from the Task Review (2026-10-09)

**Denials and anonymous traffic.** The portal's Angular app asks `GET /api/auth/sessions/current` whenever a signed-out visitor opens a page, and the server answers 401. Recording every 401 would add an audit event to each such page load, and attackers could add more at will. **Decision**: record only HTTP 403 responses to authenticated sessions as `access.denied`. Count anonymous 401 responses in an operational metric. The detail rejected: recording both and filtering in the review screen, which still grows storage without bound.

**Source.** The source is the client IP address from the deployment's trusted forwarded headers. A client-supplied header is trusted only when the service is configured to trust the proxy that sets it, so the service README gains a forwarded-headers requirement. No user-agent is stored because it is client-supplied and adds personal data without adding review value.

**Authentication and session events.** Sign-in success and failure are one `auth.sign-in` event per attempt; a separate session-created event would duplicate it. A failed attempt records no account: the actor is null, and the source and correlation identifier let a reviewer see repeated failures without the event revealing which account was targeted. Session revocation is `auth.session.revoked` with a reason (logout, idle timeout, password reset). Idle-timeout revocation happens in the session repository on the request path, not in a controller, so it is recorded there.

**Initial security event types.** `auth.sign-in`, `auth.password-reset.completed`, `auth.session.revoked`, `auth.role.switched`, `access.denied`, `audit.mutation.attempted`, and `audit.review.accessed`.

**Actor display.** Storing a display name in each event would copy personal data into append-only history that cannot be edited. The review API instead resolves the current email at view time and returns nothing for removed accounts, which the screen shows as "(account removed)". Events stay reviewable by stable ID either way.

**Audit reads.** Recording each Admin search and detail view lets the institution answer who looked at audit history. Admin-only access keeps the volume low. Each such event goes through the normal security-event path, so a recording failure never blocks the read.

**Explicit recording only.** The initial capability records events that owning features submit with their allowlisted fields. An automatic Entity Framework change interceptor is deferred until a business feature needs it, because no business feature exists yet to define which entities and fields it should cover.

**Database accounts.** Today the API runs as `postgres`, and `dotnet ef` reads the API's configuration. Splitting accounts means a migrations connection string used by `dotnet ef`, a runtime account for the API, and grants that cover every existing table and sequence plus default privileges for tables created by later migrations. Without the default privileges, the first migration after the split would leave new tables unreadable by the API. The service and 001 README commands change accordingly.

**Outbox worker.** A worker claims a pending row by setting a short lease with a conditional update, so more than one service instance can run without delivering twice, and a crashed worker's rows are claimed again once the lease expires. A lease works the same on PostgreSQL and on the SQLite used by the existing test host, unlike row-lock skipping, which is PostgreSQL-only. The runtime role can update outbox state but never delete rows. Delivered rows are kept; cleaning them up is not part of the initial capability and is not formal audit retention.

**Fallback sink.** Development writes JSON lines to a configured folder. Production startup fails unless the deployment owner has configured a durable sink and an alert destination. That owner and destination are still to be named.

**Monitoring mechanism.** The project has no metrics stack. Use the built-in .NET metrics counters, critical-level structured logs, and an ASP.NET Core health check that reports exhausted outbox items, so a deployment can alert on them without a new package.

**Field limits.** Attackers control some event content, such as the requested route. Strings and metadata have maximum sizes, and access-denied events store the matched route template and method, never the raw path or query string.

