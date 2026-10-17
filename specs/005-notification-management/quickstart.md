# Quickstart: Notification Management Validation

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Contracts**: [Admin API](contracts/notification-management-api.md), [producer contracts](contracts/notification-producer-contracts.md)  
**Data model**: [data-model.md](data-model.md)

## Prerequisites

- Node.js 24 LTS and .NET 10 SDK.
- PostgreSQL 18 development database and the Notification Management EF migration applied through the schema-owner workflow.
- Active Admin test account and non-production Student/Teacher email addresses.
- Development Brevo API key and sender identity supplied through local user secrets or environment configuration, plus an explicit allowlist of controlled, non-production test mailboxes. An empty allowlist blocks all sends. Never place the API key in source control or logs; never send real user data or credentials through the Development provider.
- Provider substitution test configuration, durable/shared protected Data Protection key ring, runtime grants for new tables/sequences, and operational alert/reconciliation ownership.
- Before Production, select a different provider and verify its content-retention/access behavior against FR-018; Brevo must be rejected by Production configuration unless separately approved.
- Use authoritative producer fixtures/contracts until Student Management, publication, feedback-window, confirmation, and credential-setup owners exist.

## Build and Automated Tests

Apply the migration with the schema-owner workflow, then re-run `database/postgresql/runtime-grants.sql` so the append-only revokes on the notification history tables take effect.

Run from the repository root after implementation:

```powershell
Push-Location iit-academic-portal-service
dotnet tool restore
dotnet ef database update -p src/IitAcademicPortal.Infrastructure -s src/IitAcademicPortal.Api
dotnet test
Pop-Location

Push-Location iit-academic-portal
npm ci
npm test -- --watch=false
npm run build
Pop-Location
```

The existing API host uses xUnit, `WebApplicationFactory`, SQLite, controllable time, and a fake email sender. Run PostgreSQL-specific migration, uniqueness, lease, cursor, and runtime-grant tests with the existing `AUDIT_TEST_POSTGRES` fixture where supported. SQLite does not prove provider-specific SQL or PostgreSQL grants.

## End-to-End Scenarios

1. Start PostgreSQL, the API with its HTTPS profile, and Angular. Sign in as active Admin and open Notification Management.
2. Trigger a teacher invitation using a synthetic/test identity. Verify the account action succeeds before delivery, Brevo accepts the individual-recipient request, the Admin receives no password, and portal history records actual destination and safe status. Keep live invitation acceptance blocked until the account owner provides the secure credential-setup proof/link flow; sending alone must not activate the account.
3. Attempt to send to a destination outside the Development test-mailbox allowlist. Verify the request is rejected before Brevo receives it; with an empty allowlist, verify every send is rejected.
4. Simulate Brevo provider failure. Verify bounded retries, no repeat of the originating action, retained attempts/history, and actionable failure after exhaustion. Simulate ambiguous acknowledgement and verify `Uncertain`, not falsely sent.
5. Request password recovery for known and unknown synthetic accounts. Verify the public response stays equivalent, the old in-memory direct-send path does not run in parallel, replay yields one logical message, and no proof/content enters permanent storage or logs.
6. Edit and preview a template with synthetic values. Verify invalid/missing placeholders fail, preview sends nothing, security preview uses only a visibly synthetic nonfunctional link and no live proof, preview output is not persisted, stale revision updates conflict, and ordinary historical snapshots remain unchanged.
7. Search with combined recipient/event/status/UTC filters, load cursor batches, inspect ordinary detail, and verify security detail omits subject/body/proof but marks content withheld.
8. Request template/history/reminder/resend APIs anonymously and as Student, Teacher, Coordinator, and a non-Admin active role on a multi-role account. Verify 401/403 without restricted response data.
9. Resend an eligible failed ordinary record twice with the same idempotency key. Verify one linked execution, recorded destination/content, an audit event, and unchanged predecessor attempts. Verify ineligible or stale records return safe conflict reasons.
10. Configure one positive global reminder interval. With a controllable clock and synthetic windows, verify due time, policy changes, one reminder per window/recipient, and suppression after close/cancel/reschedule without reading submission state.
11. Exercise publication and feedback-opening contracts with owner-supplied recipients. Confirm finalization alone emits no result notice and no marks, feedback text, ratings, or submission identifiers enter messages/history. Do not mark end-to-end integration complete until each owner exists and passes its tests.
12. Fail dispatch persistence after a committed event. Verify the originating action remains successful, an operational signal is raised, and source replay creates exactly one logical notification.
13. Run the producer contract against a substitute email adapter. Verify an existing feature's event request is unchanged, the replacement requires only adapter/configuration changes, and Production rejects the Development-only Brevo configuration.

## Verification Results (2026-10-16)

- `dotnet test` from `iit-academic-portal-service/` with `AUDIT_TEST_POSTGRES` set: **882 passed, 0 failed**. About 270 of these are new (dispatch, templates, history, resend, reminders, producers, provider and Production guards, PostgreSQL). The notification tests are in `tests/IitAcademicPortal.Api.Tests/Notifications/`: dispatch, templates, history, resend, reminders, academic producers, the teacher-invitation producer, provider contract and Production guard, and PostgreSQL (`NotificationPostgresTests`: seed, idempotency keys, lease claims and recovery, interrupted send, keyset paging, append-only and mutable permissions as the runtime role).
- `npm test -- --watch=false`: **197 passed**. `npm run build`: succeeds with no warnings; the notification screens are lazy chunks and the initial bundle is 469.1 kB (budget 500 kB).
- Provider substitution: the same producer request is delivered through the Brevo adapter (over a fake HTTP layer, with an allowlisted address) and through a substitute adapter with no change to the request; a host started in Production with Brevo selected refuses to start; a destination outside the allowlist, or an empty allowlist, is rejected before any HTTP request.

### Performance validation (SC-002, SC-003)

`NotificationPerformanceTests` run with the rest of the suite (SQLite, in memory) and print their numbers; run them alone with `dotnet test --filter NotificationPerformanceTests --logger "console;verbosity=detailed"`.

| What | Fixed workload | Limit | Measured here (2026-10-16) |
|---|---|---|---|
| Originating-action overhead (SC-002) | 100 measured Admin teacher creations (after 10 warm-up), then 100 producer calls, each against a host with notifications disabled and a host whose provider fails every call, with the worker running every 10 actions | 95th percentile at most 200 ms above notifications disabled; every action keeps its outcome | teacher creation +0.8 ms, producer call +1.5 ms; all 201 and Accepted |
| Time to provider submission (SC-003) | 500 immediate notifications (5 events of 100 recipients) accepted together; a worker with the real batch of 50 and a 5-second wait after a pass that does not fill the batch | at least 95% within 60 s | 100% within 60 s (p95 0.69 s, 11 passes) |

The default worker (batch 50, poll 5 s) sustains 10 notifications per second when every pass is full, and a burst of up to 600 is guaranteed to be picked up inside 60 s even if every pass waits the full poll interval. These numbers come from an in-memory test database on one machine and prove the logic and the margins, not production capacity: measure a representative volume against PostgreSQL before sizing Production (the specification gives no volume).

### Channel independence (SC-009)

`NotificationChannelIndependenceTests` register a fake non-email channel and a template for it in the test only. The same unchanged producer calls (results published, and a password-reset request) then produce one notification per channel; a failure on one channel does not affect the other; a replay creates nothing new; a resend stays on its channel; and production is pinned to one channel and seven email templates.

### Not verified here (explicit gates)

- **No real Brevo request was made.** Everything above uses a fake HTTP layer. A Development smoke test with a real key, a verified sender, and a controlled test mailbox still has to be done by someone holding those credentials (scenario 2 above).
- **Invitation delivery (resolved by feature 006).** A new teacher now gets the common account-setup email with a working link (see `specs/006-user-management/quickstart.md`). Student invitation and email confirmation have no producer or link provider at all.
- **The academic producers are contract-only.** Results publication and feedback-window owners do not exist; their calls are tested with fake owners and nothing calls them. Their end-to-end integration is not complete until each owner calls the contract and passes its own tests.
- **Production gates:** choose a Production email provider other than Brevo and verify its content retention and access against the no-stored-security-content rule; persist the Data Protection key ring; re-run the runtime grants; name an owner for the persistence-failure and retry-exhausted signals. No Production provider is selected by this feature.
- **No live browser check** (headless Chrome with axe-core) and **no migration applied to the Development database** in this pass.

### Branded email (added 2026-10-16)

- Every email is sent as plain text plus the IIT-branded HTML layout. Branding is configuration, not an Admin screen: `Notifications:LogoUrl` (absolute https; default `{Notifications:PortalUrl}/images/iit-logo.png`, and for real recipients it must be publicly reachable), `Notifications:BrandColor` (six-digit hex, default `#2E3192`, must keep a 4.5:1 contrast with white), `Notifications:PortalName`, `Notifications:LogoAlt`, `Notifications:FooterText`, and `Notifications:ButtonLabel`. A change needs a restart; an invalid value stops the service from starting.
- Verified by tests: the layout (logo with alt text, heading, footer, brand color, encoding of hostile text, link rules, button for a lone address, fallback without a usable logo address), the branded send for an ordinary and a security message, configured branding in a sent email, the startup validation of the branding values, the Brevo payload with both versions, the https-only logo rule (an http portal address needs an explicit https logo or `ShowLogo=false`), and the preview (including the nonfunctional placeholder for secure messages).
- **Not verified:** how the layout renders in real mail clients (Gmail, Outlook, mobile), and whether the logo loads from a deployed address. Check both with a Development smoke test to an allowlisted mailbox before relying on the look.

## Where Things Are

- Admin API and producer contracts: [contracts/](contracts/).
- Admin UI: `/admin/notifications` with template, history/detail/resend, and reminder-policy views.
- Persistence: `PortalDbContext`, the new EF migration, and updated runtime table/sequence grants.
- Development email delivery: Brevo transactional email API, configured only with local secrets/environment settings and synthetic test recipients.
- A database migration is mandatory for notification tables, revisions, attempts, transient request work, and reminder policy.
