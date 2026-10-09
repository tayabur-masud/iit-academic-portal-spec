# Implementation Notes: Audit System

**Branch**: `feature/002-audit-system` (spec repo and both submodules) · **Recorded**: 2026-10-09 · **Not committed or pushed**

## Status

| Story | State |
|---|---|
| US1 Record important events | Implemented and tested |
| US2 Understand relevant changes | Implemented and tested |
| US3 Search and review | Implemented and tested (service and Angular) |
| US4 Protect records, reliable recording | Implemented and tested (service and PostgreSQL) |

The code for the release gate (US1 + US3 + US4) is complete. **The release gate is not yet met**, because of the
owner decisions below and the manual checks listed under "Not done".

## Verification (T049)

| Check | Result |
|---|---|
| `dotnet build -c Release` | 0 warnings, 0 errors |
| `dotnet test`, no PostgreSQL | 184 passed, 18 PostgreSQL tests skipped with a clear message, 0 failed |
| `dotnet test` with `AUDIT_TEST_POSTGRES` set (PostgreSQL 18.3) | **202 passed, 0 skipped, 0 failed** |
| Frontend `ng test` (Vitest) | 47 passed (9 files), 0 failed (re-run after the accessibility fixes) |
| Frontend `ng build` | Succeeds; initial bundle 377 kB raw / 97 kB transferred |

SQLite is not accepted as proof of PostgreSQL behavior; the 18 PostgreSQL tests are what proves the items below.

### PostgreSQL migration and grants

- Migration `20261009133302_AuditEventsAndSecurityOutbox` is applied to the local development database
  (`dotnet ef migrations list` shows all four migrations applied). The generated DDL was inspected: `timestamptz`
  columns, no foreign keys on actor or entity IDs, and a descending `(OccurredAtUtc, Id)` index.
- Proven against a disposable database with a separate DDL owner and a restricted runtime role, through
  `runtime-grants.sql` and `audit-outbox-recovery.sql`: the runtime role can `SELECT` and `INSERT` audit events and
  cannot `UPDATE`, `DELETE`, or `TRUNCATE` them; it cannot `DELETE` outbox rows but can update delivery state; it
  does not own the table; it can run the sign-in and session flows; a table created by a later migration is usable
  without a new grant; event IDs are unique; concurrent inserts with equal timestamps lose nothing; historical
  actor and entity IDs survive deletion of the source row; only the schema owner can run the recovery function,
  which only touches `Exhausted` items and requires an operator ID and reason.
- The local development database still connects as `postgres`. That is **not** evidence of the runtime boundary;
  only the disposable-database tests are.

### Durable outbox and fallback evidence

Covered by `SecurityAuditOutboxTests` and `SecurityAuditFallbackTests`: retry with backoff, restart, idempotent
delivery by event ID, two workers never holding one item, expired lease reclaimed, exhaustion retained, no silent
discard, and primary outcomes unchanged when the outbox write fails. When the outbox write fails, a safe envelope
reaches the file sink and the event-loss counters and critical logs fire. Outside Development the service refuses
to start unless `AuditFallback:Sink` is `File` with a `Directory`.

## Performance (T052, SC-006)

- **Method**: a scratch PostgreSQL database with the real migrations; `seed-audit-volume.sql` loaded **1,000,000**
  events over 400 days (about 54 s); nine first-page query shapes (no filter, 7-day range, 30-day range plus
  category, event type, outcome, actor, entity, correlation ID, event type plus outcome plus 90 days), the same
  `ORDER BY "OccurredAtUtc" DESC, "Id" DESC LIMIT 26` the API issues, 300 runs each through `pgbench`, per-run
  latencies taken from its log. The scratch database was dropped afterwards.
- **Environment**: developer laptop, Windows 11, PostgreSQL 18.3 on `127.0.0.1`, default `shared_buffers` (128 MB),
  warm cache, single client.
- **Result**: 95th percentile **0.23 to 0.48 ms** per shape; worst single run 9.6 ms. The target is 3 s.
- **Limits of this measurement**: database time only (no HTTP, JSON, or `actorDisplay` lookup), warm cache,
  uniformly distributed data. A very rare filter combined with a very wide range could scan further than these
  shapes did. No index was added; the existing indexes were sufficient.
- **Volume**: 1,000,000 events confirmed by the owner on 2026-10-09. Repeat on production-like hardware before release.

## Contract comparison (T051)

`contracts/audit.openapi.json` was compared with the implementation: both routes are `GET` only; the 11 query
parameters match `AuditSearchQuery`; `AuditEvent` (including `actorDisplay`), `AuditChange`, and the page schema
match the DTOs; the 400, 401, 403, and 404 outcomes are covered by tests. Approved difference: the mutation guard
returns `405` with `Allow: GET`, which the contract describes in the route notes rather than as separate
operations.

## Accessibility and responsive review (T050)

Run on 2026-10-09 against the real API and Angular dev server, in headless Chrome driven by a script, with axe-core
(WCAG 2.0/2.1/2.2 A and AA rules) on each state.

| Check | Result |
|---|---|
| axe: search page, validation error, empty result, detail, event not found | No violations after the fixes below |
| Keyboard: tab order | Skip link, navigation, sign out, then filters in reading order, Apply, Clear, then the event links; the focus ring is a visible 2 px outline |
| Validation error | Message shown, `aria-invalid` set, focus moves to the invalid field |
| Status announcement | The `role="status"` region reads "No events match these filters." |
| Horizontal page overflow at 1280, 768, 375 px, 640 px (200% zoom of 1280), and 320 px (400% reflow) | None; the table scrolls inside its own region |
| Navigation | "Audit history" is the active item on `/admin/audit`; "Dashboard" is not |

Three defects were found by this run and fixed:

1. **Contrast**: success and failure text on the tinted badge missed 4.5:1. Outcome badges now use `badge-success`
   and `badge-error` (white surface, border); recorded in `specs/design-system.md`.
2. **Page overflow at 768 px and narrower**: hidden screen-reader text inside the table escaped its scroll region and
   widened the page. The scroll regions are now positioned.
3. **Focus after a validation error** did not move to the field, because the attribute was read before it rendered.

**Not done**: a pass with a real screen reader (NVDA or VoiceOver) and a person using the keyboard in a normal
browser, so T050 stays open. The screenshots were taken at 1280, 768, 375, 640, and 320 px and reviewed at 375 px.

## Quickstart scenarios (T054)

**Run live** on 2026-10-09 (development API on PostgreSQL, 37 of 37 checks passed, using the seeded accounts):
scenario 1 (Admin allowed; a multi-role non-Admin session denied before and after a role switch; a switch to an
unassigned Admin role denied; anonymous 401; no client-supplied role bypass), scenario 2 (combined filters, newest
first, cursor pages without overlap, over-limit page size and inverted range rejected), scenario 3 (sign-in success
and failure with no account and no submitted email, role switch success and denial, an authenticated 403, a
mutation attempt, audit reads, source IP and correlation ID present), scenario 9 (POST, PUT, PATCH, and DELETE on
the audit route return 405 with `Allow: GET`, with and without a CSRF token; anonymous gets 401), scenario 12
(email resolved as the actor), scenario 8 in part (no password or session cookie in any response), and the
protected `/health/audit` (anonymous 401, non-Admin 403, Admin 200). Starting the service without the Development
environment refused to start because no durable sink was configured, which is the intended Production guard.

**Covered by automated tests only**: scenarios 4 to 7 (stand-in business event, atomic transactions, outbox
delivery and exhaustion, outbox-write failure and fallback), scenario 10 (`AuditRetentionTests`: no deletion or
archive job exists), scenario 11 (historical identity and concurrent inserts, on PostgreSQL), and the PostgreSQL
parts of scenario 9. They need fault injection that a live run cannot do safely.

**Not done**: scenario 13 beyond the automated checks above (see T050), and a hand walk of the full UI by a person.
**FR-014 compliance is not claimed** until the production durable sink volume and an alert destination exist and
are verified. SC-005 reviewer usability sessions are to be scheduled separately and have not happened.

## Unresolved release gates (T002, T053)

These are owner decisions, recorded in the service README under "Deployment notes":

1. ~~The production durable operational-event sink~~ **Decided 2026-10-09**: the `File` sink on a persistent volume.
   The volume itself must still be provisioned and `AuditFallback` configured per environment.
2. **Open**: the protected alert destination for the four alert signals.
3. **Open**: the person authorized to run recovery for exhausted security events.
4. ~~Confirmation of the 1,000,000-event volume~~ **Confirmed 2026-10-09**.
5. Trusted proxy addresses for `ForwardedHeaders:KnownProxies` per environment.
6. Creating the runtime role and running `runtime-grants.sql` in each deployed environment.

## Discrepancies found while verifying (T001)

None between the three READMEs, `package.json`, and the project files for Angular 21, Node 24, .NET 10, EF Core 10,
and PostgreSQL 18. The README's `dotnet ef` step now explains `ConnectionStrings:PortalMigrations`, and the 001
quickstart carries a note that the command is unchanged for a single local account.

## Documents updated in this work

Service README (audit section, configuration table, database accounts, fallback sink, recovery, monitoring,
retention, feature-owner guide, deployment notes), frontend README (audit feature, UTC timestamp, security note),
`specs/design-system.md` (audit timestamp and paging convention), `specs/001-role-based-auth/quickstart.md`,
`specs/002-audit-system/data-model.md` (retry stamps the Handled fields), and this file.
