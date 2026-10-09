# Quickstart: Audit System Validation

**Feature**: [Audit System](spec.md)  
**Related artifacts**: [Data model](data-model.md), [Audit review contract](contracts/audit.openapi.json), [Research decisions](research.md)

## Prerequisites

- Node.js 24 LTS and the .NET 10 SDK are installed.
- PostgreSQL 18 is available for migration, permission, transaction, and outbox integration tests. The normal API test suite can run without PostgreSQL using its in-memory SQLite test fixture.
- A development database is configured through user secrets or environment variables; do not commit connection strings or credentials.
- Apply the version-controlled audit/outbox migration using the separate DDL/migration principal (the migrations connection string), then apply `database/postgresql/runtime-grants.sql`. Configure the API with the restricted runtime principal; verify it is not the audit-table owner and cannot update/delete audit events.
- Development: point the file sink at a writable folder for failures to write the security-event outbox. Production: configure the durable sink and alert destination named by the deployment owner; the service refuses to start without them. Console-only logging is never accepted as durable retention.
- Behind a reverse proxy, configure trusted forwarded headers so events record the client IP.
- Configure development Admin, Student, Teacher, Coordinator, and Teacher+Coordinator accounts using the existing development seeder. The multi-role test account starts with Teacher as active role.
- Keep development email delivery on the existing local pickup sink for password-reset event tests; never expose reset proofs in test logs.

## Build and Test

Run the service and frontend checks from their respective submodules:

```powershell
Push-Location iit-academic-portal-service
dotnet tool restore
dotnet test
Pop-Location

Push-Location iit-academic-portal
npm ci
npm test -- --watch=false
npm run build
Pop-Location
```

For PostgreSQL-specific checks, start the service on the existing HTTPS launch profile and apply/verify migrations with the separate DDL principal. Run the dedicated PostgreSQL integration suite against a disposable database and a runtime credential with the restricted grants. SQLite tests do not verify PostgreSQL grants or provider-specific behavior.

## End-to-End Scenarios

1. **Admin-only review**: Sign in as Admin and retrieve the audit collection and one event detail. Switch the same multi-role session to Teacher or Coordinator and repeat; both reads must return access denied. Switch back to Admin and verify access is restored. Repeat with a Student session. Confirm changing Angular navigation or submitting a different role value cannot grant access.
2. **Filters and pagination**: As Admin, combine a start-inclusive/end-exclusive UTC range with actor, category, event type, entity type/ID, outcome, and correlation ID filters. Verify results match all filters, are newest first with event ID tie-break, and follow keyset cursors without duplicates across concurrent inserts. Verify page size defaults to 25 and values above 100 are rejected or bounded consistently with the contract.
3. **Initial security event catalog**: Exercise successful and failed sign-in (one event per attempt, no account recorded on failure), password-reset completion/rejection, session revocation by logout, by idle timeout, and by password reset (each with its reason), active-role switch success/denial, an authenticated 403 denial, and an attempted audit-record mutation. Confirm event type, outcome, actor or null actor, UTC time, server-generated correlation ID, and client-IP source are correct.
   - **Anonymous traffic is not audited**: open the sign-in page signed out and call a protected route without a session. Confirm no audit events appear and the anonymous-401 metric increases.
   - **Reads are audited**: as Admin, run a search and open an event. Confirm each appears as `audit.review.accessed` without copying the returned contents.
4. **Business event integration seam**: Use a representative integrated business feature/test fixture to submit a mandatory business event and its approved change-field allowlist. Confirm this reusable capability does not require unbuilt course, fee, enrollment, elective, result, account, or coordinator workflows to exist.
5. **Atomic business transaction**: Inject audit-event persistence failure while an integrated business mutation is pending. Verify neither the business mutation nor its mandatory event commits. Repeat with successful persistence and verify both commit together.
6. **Security-event delivery**: Cause an audit-event sink/consumer failure after the security event reaches the durable outbox. Verify the primary authentication/authorization result is unchanged, the outbox item remains retryable, retries are idempotent by event ID, and delivery eventually creates one formal event. Exhaust automatic retries and verify alerting plus authorized recovery, with no discard.
7. **Outbox-write failure fallback**: Simulate failure to persist the outbox itself while the request outcome is known. Verify the primary outcome is unchanged, the configured durable operational-event sink receives a safe envelope, a protected critical alert fires, and the event is held for authorized recovery. Verify console-only logging is not accepted as passing this gate.
8. **Sensitive-field handling**: Exercise allowlisted before/after changes and sensitive/unapproved fields. Confirm only approved values are stored, protected fields are omitted or represented by a safe changed indicator, and credentials, cookies, reset proofs, or tokens appear nowhere in stored events or user-facing responses.
9. **Append-only boundary**: Through the API and ordinary application services, attempt event update/delete (POST, PUT, PATCH, and DELETE on both audit routes, with and without a CSRF token) and verify each returns 405 with `Allow: GET` and records one `audit.mutation.attempted` event, while an anonymous attempt returns 401 and records nothing. Using the runtime PostgreSQL role directly, verify SELECT/INSERT succeed and UPDATE/DELETE fail on formal audit events. Confirm outbox delivery-state updates remain permitted and do not alter delivered event rows.
10. **Retention and archive**: With no approved policy configured, verify no automatic deletion or archival runs. If a later approved archive policy is enabled for a test, verify archived records remain searchable/detail-retrievable by Admin and remain denied to other sessions. Do not test or implement deletion without an approved policy and authorized deletion process.
11. **Historical identity and concurrency**: Deactivate/remove an actor or affected entity and confirm historic event identifiers remain reviewable. Concurrently submit events with matching timestamps and verify unique IDs, no overwrite, and deterministic pagination.
12. **Actor and time display**: Open an event whose actor is active, one whose account was removed, and one with no actor. Confirm the email, "(account removed)", and a blank actor are shown respectively. Confirm every timestamp is labeled UTC.
13. **Accessible review UI**: Validate table/search/loading/empty/error/detail states with keyboard navigation, screen reader, responsive breakpoints, and 200% zoom using `specs/design-system.md`.

## Contract Smoke Checks

The API exposes only read routes for browser review:

- `GET /api/audit-events` with supported UTC time, actor, category, event type, entity, outcome, correlation, page size, and cursor filters.
- `GET /api/audit-events/{eventId}` for one safe event detail.

Both routes require the existing session cookie and an active Admin role checked by the server on each request. No client-side event creation or audit mutation route exists. Expected errors use Problem Details and do not reveal event contents.
