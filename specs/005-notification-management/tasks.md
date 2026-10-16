# Tasks: Notification Management

**Input**: Design documents from `/specs/005-notification-management/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/)

**Tests**: Automated tests are included because the specification defines acceptance contracts and the project constitution requires tests for business-critical and security-sensitive behavior.

**Organization**: Tasks are grouped by user story. Each story has an independent test criterion; shared persistence and dispatch foundations must be completed first.

## Phase 1: Setup

**Purpose**: Configure the Development-only email provider safely without adding a provider SDK.

- [X] T001 Add typed Brevo Development configuration and fail-closed `AllowedRecipients` validation in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Email/BrevoOptions.cs`, bind it in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs`, and add only non-secret defaults in `iit-academic-portal-service/src/IitAcademicPortal.Api/appsettings.Development.json`; never commit the API key and reject Brevo outside Development.

## Phase 2: Foundational

**Purpose**: Complete the shared notification domain, persistence, rendering, transport, and worker required by every story.

- [X] T002 Define the channel-independent event request, seven event types, `Email` channel, producer port, and lifecycle status values in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Notifications/` and `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/`.
- [X] T003 Add notification template, immutable revision, record, dispatch-state, attempt, transient password-recovery request, reminder-policy, and reminder-schedule entities in `iit-academic-portal-service/src/IitAcademicPortal.Domain/Notifications/`; preserve the model constraints verbatim: dispatch status is `Pending`, `Retrying`, `Sent`, `Failed`, `Suppressed`, or `Uncertain`; history records and attempts are immutable; security content snapshots are null; recipient IDs are not cascading foreign keys; reminder interval is positive whole minutes with no default; reminder schedule states are `Scheduled`, `Cancelled`, or `Dispatched` and contain no feedback-submission data.
- [X] T004 Map immutable notification history/attempts separately from mutable dispatch state, enforce unique event-occurrence-recipient-channel and template-revision keys, use restrictive history relationships, and add bounded-query indexes in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs` and notification persistence configuration files.
- [X] T005 Create the version-controlled EF Core migration for templates, revisions, immutable records/attempts, mutable dispatch state, protected transient request work, and reminder policy/schedule in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/Migrations/`; update `iit-academic-portal-service/database/postgresql/runtime-grants.sql` so the runtime role may update dispatch state but has no update/delete privilege on notification records, template revisions, or delivery attempts.
- [X] T006 Implement PostgreSQL history, dispatch-state, and transient-request stores plus `NotificationRequestService` and `INotificationFailureSignal` in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Notifications/NotificationStore.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/NotificationRequestService.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/INotificationFailureSignal.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/PasswordRecovery/PasswordRecoveryQueue.cs`; protect transient submitted addresses, delete them after processing/expiry, enforce event idempotency, and on persistence failure emit a secret-free operational signal without failing the owner action while exposing source-occurrence reconciliation.
- [X] T007 Seed one validated email template for each of the seven event types and an immutable initial revision in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/NotificationTemplateCatalog.cs`; security templates must start enabled and ordinary academic templates must have no inferred user preference.
- [X] T008 Implement strict placeholder validation, plain-text rendering, and synthetic preview values in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/NotificationTemplateRenderer.cs`; security previews may return only visibly synthetic, nonfunctional action-link placeholders and must never generate a proof or persist preview output.
- [X] T009 Implement `BrevoEmailSender` behind the existing `IEmailSender` in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Email/BrevoEmailSender.cs`; submit one recipient per request to Brevo's transactional email endpoint, use only Development configuration, reject recipients outside the explicit allowlist before HTTP submission, and map provider outcomes without logging credentials or message content.
- [X] T010 Implement the leased notification dispatch worker and validated options in `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Notifications/NotificationDispatchWorker.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Notifications/NotificationDispatchOptions.cs`; update only mutable dispatch-state rows, use eight total attempts, five-second base delay, exponential backoff capped at fifteen minutes, five-second polling, batches of 50, two-minute leases, terminal permanent failures, and `Uncertain` for ambiguous provider acceptance.
- [X] T011 Add deterministic notification test support using a fake provider, controllable `TimeProvider`, and isolated Data Protection keys in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Infrastructure/NotificationTestHost.cs` so worker restart, retry, proof, and provider-selection behavior can be tested without Brevo credentials.

**Checkpoint**: Database migration applies, the shared worker can accept and dispatch one idempotent notification, Development Brevo rejects non-allowlisted recipients, and the fake-provider test host is available.

## Phase 3: User Story 1 - Receive Reliable Event Notifications (Priority: P1)

**Goal**: Existing account workflows request durable notification work without waiting for delivery or exposing credentials; failures are recoverable and retries do not duplicate logical messages.

**Independent Test**: Request password recovery for known and unknown synthetic accounts, simulate provider outage/restart, and verify the public response remains equivalent, one logical work item is retained, bounded retries occur, and proofs/content never enter permanent history. Teacher invitation delivery remains gated on the account owner's secure setup-link provider.

### Tests for User Story 1

- [X] T012 [P] [US1] Add dispatch integration tests for provider outage, process restart, bounded retry, event replay idempotency, permanent failure, ambiguous acknowledgement, and notification-store write failure; verify the failure signal is raised, the originating action remains successful, source-occurrence replay recovers exactly one logical notification, and immutable history/attempts are unchanged in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationDispatchTests.cs`.
- [X] T013 [P] [US1] Extend `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/PasswordRecoveryTests.cs` to verify known/unknown requests retain the same public response, one accepted request creates one logical notification, fresh Identity proofs are generated only for eligible delivery, and no proof or rendered security content is persisted or logged.
- [X] T014 [P] [US1] Add teacher-invitation producer tests in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/TeacherInvitationProducerTests.cs` proving a committed account requests an invitation without setting or returning a password, a rolled-back account requests none, and pending accounts remain unable to sign in.

### Implementation for User Story 1

- [X] T015 [US1] Replace the direct in-memory password-recovery send path with durable notification requests in `iit-academic-portal-service/src/IitAcademicPortal.Application/PasswordRecovery/PasswordRecoveryService.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/PasswordRecovery/PasswordRecoveryQueue.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/PasswordRecovery/PasswordRecoveryWorker.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs`; preserve the generic response and remove the parallel legacy sender path.
- [ ] T016 [US1] Wire committed teacher provisioning to the channel-independent invitation producer in `iit-academic-portal-service/src/IitAcademicPortal.Application/Teachers/TeacherProvisioningService.cs` using the account owner's secure setup-link contract; do not invent a proof flow or activate the account. This task is blocked until the owning account workflow supplies that contract, and end-to-end invitation acceptance must remain marked incomplete until it is implemented and verified. **Status (partial, still blocked):** the request side is wired and tested (`TeacherProvisioningService` requests a `TeacherInvitation` after the account commits; `TeacherInvitationProducerTests`), and the notification fails visibly with `SetupLinkProviderMissing` because no secure setup-link contract exists. This stays unchecked: end-to-end invitation delivery is not implemented and cannot be until the account owner supplies that contract and its `ISecurityContentProvider` replaces `UnavailableSecurityContentProvider`.

**Checkpoint**: Password recovery and supported existing account producers are durable, idempotent, provider-independent, and do not weaken anti-enumeration or credential rules; teacher invitation's external link-provider gate is visible.

## Phase 4: User Story 2 - Maintain and Preview Message Templates (Priority: P1)

**Goal**: Active Admins can manage immutable template revisions and safely preview wording without sending mail or generating security proofs.

**Independent Test**: Create a valid revision, reject invalid/stale edits, preview with synthetic values, reject disabling a security template, and confirm a later request uses the saved revision while earlier records remain unchanged.

### Tests for User Story 2

- [X] T017 [P] [US2] Add API tests for template list/detail, placeholder validation, required security action links, stale revision conflicts, security-template enable invariants, and safe audit events in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationTemplateTests.cs`.
- [X] T018 [P] [US2] Add Angular tests for template loading, form validation, synthetic preview, disabled security-template controls, save conflicts, and accessible error states in `iit-academic-portal/src/app/features/notifications/notification-templates.spec.ts`.

### Implementation for User Story 2

- [X] T019 [US2] Implement template list/view/update services and `GET /api/admin/notification-templates` plus `GET/PUT /api/admin/notification-templates/{eventType}/email` in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/NotificationTemplateService.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/NotificationTemplatesController.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/NotificationTemplateContracts.cs`; create immutable revisions, enforce expected-revision concurrency, and audit successful changes safely.
- [X] T020 [US2] Implement synthetic-only preview and the Admin template list/editor/preview UI in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/NotificationTemplatesController.cs` and `iit-academic-portal/src/app/features/notifications/notification-templates.ts`; wire the Admin route/navigation in `iit-academic-portal/src/app/app.routes.ts` and `iit-academic-portal/src/app/layout/shell/shell.ts`, with loading, empty, validation, success, and failure states.

**Checkpoint**: Admins can manage revisions and preview only synthetic, nonfunctional security-link placeholders; subsequent notifications pin the correct revision.

## Phase 5: User Story 3 - Search History and Recover Failed Deliveries (Priority: P1)

**Goal**: Active Admins can search immutable history, inspect safe detail, and resend eligible failures without overwriting prior attempts.

**Independent Test**: Search seeded records with combined filters and multiple cursor batches, verify content-withheld security detail and active-role authorization, then resend a failed record and verify one linked execution plus an audit record.

### Tests for User Story 3

- [X] T021 [P] [US3] Add API tests for combined recipient/event/status/UTC filters, inclusive/exclusive bounds, cursor ordering, page-size limits, account-removal history, and active-role 401/403 behavior in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationHistoryTests.cs`.
- [X] T022 [P] [US3] Add resend tests for eligibility checks, exact ordinary destination/content, fresh security proof generation, repeated idempotency keys, unchanged predecessor attempts, safe conflict reasons, and audit events in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationResendTests.cs`.
- [X] T023 [P] [US3] Add Angular tests for combined filters, cursor-based infinite loading, retained rows on retry, security content withholding, confirmation, and safe ineligible-resend reasons in `iit-academic-portal/src/app/features/notifications/notification-history.spec.ts`.

### Implementation for User Story 3

- [X] T024 [US3] Implement bounded keyset history search and safe detail DTOs in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/NotificationHistoryService.cs`, `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/NotificationHistoryController.cs`, and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/NotificationHistoryContracts.cs`; use `(RequestedAtUtc DESC, NotificationId DESC)`, default page size 25, maximum 100, UTC half-open ranges, and omit security subject/body/proof/link while returning `contentWithheld: true`.
- [X] T025 [US3] Implement idempotent failed-only resend execution in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/NotificationResendService.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/NotificationHistoryController.cs`; preserve predecessor attempts, validate current recipient eligibility, require fresh owner authorization for security messages, and audit safe request outcomes.
- [X] T026 [US3] Implement the Admin notification history/detail/resend screens in `iit-academic-portal/src/app/features/notifications/notification-history.ts` and `iit-academic-portal/src/app/features/notifications/notification-detail.ts`; use combined filters, shared cursor infinite loading, accessible Load more/retry states, content-withheld detail, and explicit resend confirmation.

**Checkpoint**: History is bounded, append-only, permission-checked, and content-safe; only eligible failed records can create one linked resend execution.

## Phase 6: User Story 4 - Receive Academic Publication and Feedback Notices (Priority: P2)

**Goal**: Publish result notices and feedback opening/reminder requests through owner contracts without exposing marks or feedback-submission state.

**Independent Test**: Use fake authoritative result/window owners and a controllable clock to verify publication-only notification, eligible-recipient selection, global reminder timing, policy/close-time rescheduling, and zero dependence on feedback submissions.

### Tests for User Story 4

- [X] T027 [P] [US4] Add reminder-policy and schedule tests for positive-value validation, unconfigured state, current close-time recomputation, one reminder per window occurrence/recipient, cancellation, superseding occurrences, and zero feedback-submission reads in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/FeedbackReminderTests.cs`.
- [X] T028 [P] [US4] Add producer contract tests for committed ResultsPublished and FeedbackWindowOpened events, owner-supplied eligible recipients, finalization without publication, and exclusion of marks/submission data in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/AcademicNotificationProducerTests.cs`.

### Implementation for User Story 4

- [X] T029 [US4] Implement the portal-wide reminder policy service in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/FeedbackReminderPolicyService.cs`; accept only positive whole-minute values, expose an explicit unconfigured state, audit changes, and apply one policy to all windows.
- [X] T030 [US4] Implement the feedback-window schedule processor in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/FeedbackReminderScheduler.cs` using an authoritative `IFeedbackWindowSource`; recompute unsent schedules on policy/close-time changes, cancel closed/cancelled/superseded windows, and never read feedback submissions.
- [X] T031 [US4] Expose `GET/PUT /api/admin/notification-settings/feedback-reminder` in `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/NotificationSettingsController.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/NotificationSettingsContracts.cs`; validate input, return safe errors, and audit changes.
- [X] T032 [US4] Add internal producer contracts and fake-owner acceptance fixtures for ResultsPublished, FeedbackWindowOpened, and feedback closing reminders in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/AcademicNotificationProducers.cs` and `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/AcademicNotificationProducerTests.cs`; keep student-management, publication, and feedback-collection implementation out of scope and mark end-to-end work incomplete until their owners integrate.
- [X] T033 [US4] Implement the Admin reminder-policy control in `iit-academic-portal/src/app/features/notifications/feedback-reminder-setting.ts`; provide a labeled positive-minute input, unconfigured/loading/validation/success/error states, and no per-window override.

**Checkpoint**: Reminder behavior is proven with fake authoritative producers; no feature completion claim is made for missing academic producers until their owning workflows invoke the contracts.

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Validate provider isolation, database-specific behavior, accessibility, operational safety, and requirement traceability.

- [X] T034 [P] Add PostgreSQL 18 integration tests for migration application, unique idempotency keys, lease recovery, keyset queries, and runtime table/sequence permissions; verify the runtime role can update dispatch-state rows but cannot update/delete notification records, template revisions, or delivery attempts in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationPostgresTests.cs`.
- [X] T035 [P] Add provider substitution and Production guard tests proving the same producer event works through Brevo in Development and a substitute adapter without producer changes, Production refuses Brevo configuration, and Development rejects a non-allowlisted destination and an empty allowlist before any HTTP request in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationProviderContractTests.cs`.
- [X] T036 [P] Verify responsive layout, keyboard access, focus/error announcements, infinite-scroll fallback, and stable loading/error/empty states for notification Admin screens in `iit-academic-portal/src/app/features/notifications/notification-accessibility.spec.ts`.
- [X] T037 Update `specs/005-notification-management/quickstart.md` with actual migration, test, Brevo allowlist, and provider-substitution results after implementation; keep Production-provider retention verification and missing producer-owner gates explicit.

## Phase 8: Branded Email Layout (added 2026-10-16)

**Purpose**: Apply IIT branding (logo and colors) to every email without making templates editable HTML. Every notification type already has its own template; this adds one shared, fixed layout.

- [X] T038 Add `EmailLayout` (a fixed IIT-branded HTML layout built in memory from the plain-text subject and body, with encoding, link and button rules), `EmailMessage.HtmlBody`, the `Notifications:LogoUrl` setting, HTML support in the Brevo and SMTP adapters, send-time wiring in `NotificationDispatchProcessor`, and the preview `html` in `iit-academic-portal-service/src/`.
- [X] T039 Add `BrandedEmailTests.cs` in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/` covering the layout, encoding of hostile text, link rules, the logo fallback, the branded send (ordinary and security), nothing HTML stored or logged, the Brevo payload, and the preview.
- [X] T040 Show the branded preview in a sandboxed frame beside the plain-text version in `iit-academic-portal/src/app/features/notifications/notification-template-editor.*` with spec coverage, and update the spec (clarification and FR-027), research, contract, quickstart, READMEs, and design system.

- [X] T041 Make the email branding configurable: `EmailBranding` and `Notifications:*` settings (`LogoUrl`, `BrandColor`, `PortalName`, `LogoAlt`, `FooterText`, `ButtonLabel`) with the IIT defaults, startup validation (hex color with 4.5:1 contrast against white, absolute https logo, single-line text), tests in `BrandedEmailTests.cs`, and the spec, research, quickstart, README, and design-system updates.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 establishes the Development-only Brevo configuration contract.
- **Foundational (Phase 2)**: T002-T011 depend on setup as applicable and block all user stories. Complete persistence/migration before dispatch worker integration.
- **User Stories (Phase 3+)**: US1, US2, and US3 can proceed after the foundation and can be staffed in parallel; US4 can implement policy/scheduler contracts after the foundation but cannot claim producer end-to-end completion without its external owners.
- **Polish (Phase 7)**: Depends on all stories selected for release; PostgreSQL and provider substitution gates must pass before Production.

### User Story Dependencies

- **US1 (P1)**: Starts after Phase 2. Password recovery can be independently validated. Teacher-invitation end-to-end delivery is blocked on the secure setup-link contract from the account workflow owner (T016).
- **US2 (P1)**: Starts after Phase 2; depends on shared template persistence/catalog/renderer, not on UI work in another story.
- **US3 (P1)**: Starts after Phase 2; tests can seed history records and do not require the US1 producer integrations to be complete.
- **US4 (P2)**: Starts after Phase 2; uses fake authoritative producer contracts. Student invitation, results-publication, and feedback-window owners remain external integration gates.

### Task-Level Dependency Graph

```text
T001 -> T002 -> T003 -> T004 -> T005 -> T006 -> T010 -> T011
                    ├-> T007 -> T008 ───────┘
T001 ───────────────────────-> T009 ────────┘

T011 -> US1 (T012/T013 -> T015; T014 -> T016, which requires the external setup-link provider)
T011 -> US2 (T017 -> T019 -> T020; T018 -> T020)
T011 -> US3 (T021 -> T024; T022 -> T025; T023 and T024/T025 -> T026)
T011 -> US4 (T027 -> T029 -> T031/T033; T028 -> T032; T030 uses T029 and T027)
All selected stories -> T034-T037
```

### Parallel Opportunities

- After T003, the persistence mapping/migration path (T004-T006), template catalog/renderer (T007-T008), and Brevo adapter (T009) can be developed on separate files with the listed prerequisites.
- US1 test tasks T012-T014 can be authored in separate test files before their producer integrations; T016 waits for the external setup-link contract.
- US2 API tests (T017) and Angular tests (T018) can run in parallel after the shared contracts are stable.
- US3 history tests (T021), resend tests (T022), and Angular tests (T023) touch separate files and can be prepared concurrently.
- US4 reminder tests (T027) and academic producer contract tests (T028) can be developed independently against the shared fake clock/provider fixtures.
- After the foundation, US1, US2, and US3 can be assigned to separate implementers; US4 can proceed against fake producer contracts.

## Parallel Examples

```text
# Once the shared API contracts and test host exist:
Task: T017 Template API tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationTemplateTests.cs
Task: T018 Template UI tests in iit-academic-portal/src/app/features/notifications/notification-templates.spec.ts

Task: T021 History API tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationHistoryTests.cs
Task: T023 History UI tests in iit-academic-portal/src/app/features/notifications/notification-history.spec.ts

Task: T027 Reminder schedule tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/FeedbackReminderTests.cs
Task: T028 Academic producer contract tests in iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/AcademicNotificationProducerTests.cs
```

## Implementation Strategy

### MVP First

1. Complete T001-T011: provider guard, data model/migration, durable dispatch, rendering, and test harness.
2. Complete US1: password-recovery durable dispatch and retries; keep teacher invitation end-to-end blocked until its secure setup-link owner contract exists.
3. Validate outage/restart, idempotency, generic recovery responses, Development allowlist rejection, and security-content exclusion before expanding the Admin experience.

### Incremental Delivery

1. Add US2 template management and synthetic preview, then validate revision pinning and security-template invariants.
2. Add US3 searchable history, safe details, and failed-only resend.
3. Add US4 policy/scheduler behavior with fake owners; integrate each academic producer only when its source workflow exists.
4. Complete PostgreSQL grants/migration, Production provider replacement/retention review, accessibility, and quickstart validation before Production.

## External Integration Gates

- The account workflow owner must supply a secure initial credential-setup link provider before teacher invitations can be sent end-to-end; do not reuse password-reset proof or activate pending accounts.
- Student invitation, email confirmation, results publication, and feedback-window producers absent from the current source remain contract-only until their owners integrate them.
- Production provider selection and FR-018 content-retention/access verification are release gates; Development Brevo credentials and recipient allowlists must never be reused in Production.

## Phase 9: Convergence

- [X] T042 Validate the effective logo URL, including the `PortalUrl` fallback, as absolute HTTPS in `iit-academic-portal-service/src/IitAcademicPortal.Application/Notifications/EmailBranding.cs`; require an explicit reachable HTTPS `Notifications:LogoUrl` when the Development `PortalUrl` is HTTP, and add `BrandedEmailTests.cs` cases proving an HTTP fallback is rejected, a valid HTTPS override succeeds, and no HTTP `<img src>` is emitted per FR-027 (partial). **Done:** the validator now checks the logo address that will actually be used. With `Notifications:ShowLogo` true, a blank `LogoUrl` is accepted only when `PortalUrl` is https (the fallback is then https); an http `PortalUrl` without an explicit https `LogoUrl` stops the service from starting, and `ShowLogo=false` is the explicit way to send emails with the portal name as text instead (the Development default, since `PortalUrl` is http there). `EmailLayout` also refuses to emit an http `<img>` even if one reaches it. Tests are in `BrandedEmailTests.cs`.
- [X] T043 Add a test-only non-email channel adapter and producer conformance test in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationChannelIndependenceTests.cs`; prove an existing producer sends the same channel-independent request through the fake adapter without adding a production channel per SC-009 (missing). **Done:** a real delivery-channel seam (`IDeliveryChannel`; `EmailDeliveryChannel` is the only production adapter) and a reserved `NotificationChannel.Test` value that is never seeded or registered outside tests. The request service creates one notification per registered channel that has a template, so the same producer call reaches an extra channel with no change. `NotificationChannelIndependenceTests.cs` proves it with a fake adapter and pins that production has one channel and seven email templates.
- [X] T044 Add a repeatable notification performance validation in `iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Notifications/NotificationPerformanceTests.cs`; document a fixed representative workload, assert originating-action p95 overhead stays within 200 ms during provider outage and at least 95% of accepted immediate notifications reach provider submission within 60 seconds per SC-002/SC-003 (partial). **Done:** `NotificationPerformanceTests.cs` with a documented fixed workload. Measured here: p95 overhead of 0.8 ms (teacher creation) and 1.5 ms (producer call) during an outage against the 200 ms limit, and 100% of 500 notifications submitted within 60 s (p95 0.69 s).
