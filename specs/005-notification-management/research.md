# Research: Notification Management

**Date**: 2026-10-16  
**Feature**: [spec.md](spec.md)  
**Scope**: Resolve delivery, persistence, retry, security, API, and UI design choices using the checked-out portal and its approved technology stack.

## Existing Application and Boundaries

**Decision**: Extend the existing Angular 21 and ASP.NET Core 10 layered monolith. Keep notification requests and business policy in the application layer, event/template value objects in Domain, email-provider adapters and PostgreSQL in Infrastructure, and DTO/HTTP mapping in API. Reuse the portal session authorization, audit system, `PortalDbContext`, and existing test projects.

**Evidence**: The [service README](../../iit-academic-portal-service/README.md) documents Domain, Application, Infrastructure, and API projects. Infrastructure registers PostgreSQL, ASP.NET Core Identity, `IEmailSender`, and hosted workers. The frontend already has Admin audit and teacher-management features. See [DependencyInjection.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs) and the [design system](../design-system.md).

**Rationale**: Existing boundaries provide hosting, persistence, identity, auditing, and presentation conventions without another service or package.

**Alternatives considered**: A separate notification service or message broker was rejected because no deployment contract exists and the constitution requires a clear benefit and approval for new infrastructure.

## Email Delivery and Channel Independence

**Decision**: Use Brevo for all email delivery in Development only. Keep producers and the application `IEmailSender` contract provider-neutral; implement a Brevo adapter in Infrastructure using the standard .NET HTTP client, with provider selection and sender configuration supplied by environment configuration and secrets. Replacing Brevo requires swapping the infrastructure adapter/configuration only; producer event requests, notification records, and templates remain unchanged. Production must use a different, separately approved provider unless Brevo is explicitly approved against FR-018.

**Evidence**: [PasswordRecoveryService.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/PasswordRecovery/PasswordRecoveryService.cs) sends through the existing `IEmailSender`, which provides a provider-neutral application boundary. Brevo's official [transactional send endpoint](https://developers.brevo.com/reference/sendtransacemail) is `POST https://api.brevo.com/v3/smtp/email`, accepts an `api-key` authentication header, and returns a message ID when accepted. Its official [sent-content endpoint](https://developers.brevo.com/reference/gettransacemailcontent) can return the sent subject and body. The documentation reviewed did not establish a content-retention duration; do not assume provider-side content expires promptly.

**Rationale**: Brevo satisfies the user's Development provider selection while the existing `IEmailSender` boundary allows a provider change without modifying producers. A direct HTTP adapter avoids a new SDK dependency and keeps provider details in Infrastructure.

**Alternatives considered**: A provider SDK was not selected because the standard HTTP client is sufficient. SMTP is not selected for the new Development delivery path. Brevo is not the Production choice because its content endpoint exposes sent message bodies and the retention duration is not established by the reviewed documentation.

**Development safeguards**: Use only synthetic/test identities and data with Brevo. The adapter must fail closed unless every destination matches an explicit allowlist of controlled, non-production test mailboxes; an empty allowlist rejects all sends. Store its API key and sender identity in local user secrets or environment configuration, never source control or logs. Do not use Brevo's sent-content endpoint, tracking, or message-content retrieval APIs. Restrict the API key to Development and rotate it if exposed. Test that a destination outside the allowlist is rejected before any provider request.

**Production gate**: Select a different provider before release and verify its retention/access behavior against FR-018, sender-domain configuration, credential storage, and alert handling. Production startup/configuration must reject the Development-only Brevo adapter. No Production provider is selected by this decision.

## Durable Processing, Idempotency, and Retry

**Decision**: Add a notification-specific PostgreSQL dispatch store and hosted worker. Claim due work in bounded batches with an expiring lease and conditional update; recover claims after worker failure. Enforce one original notification per stable event occurrence, recipient, and channel with a database uniqueness constraint. Store attempts separately from mutable dispatch state.

**Retry policy**: Use eight total automatic attempts (initial send plus at most seven retries), beginning at five seconds and doubling after each retry, capped at fifteen minutes. Poll every five seconds, claim at most 50 items per pass, and use a two-minute lease. Permanent recipient/provider failures are terminal. An outcome whose provider acceptance cannot be established is `Uncertain`, is not automatically retried, and is not reported as sent. Exhausted retryable failures remain visible for the specified Admin resend flow.

**Evidence**: The [security audit outbox processor](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Auditing/SecurityAuditOutboxProcessor.cs) uses leases and bounded exponential backoff. [AuditDeliveryOptions.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Auditing/AuditDeliveryOptions.cs) defines eight attempts, five-second base delay, fifteen-minute cap, five-second poll, batch size 50, and two-minute lease. That worker delivers audit events, not email. The existing [PasswordRecoveryQueue.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/PasswordRecovery/PasswordRecoveryQueue.cs) is in-memory, capacity 500, and drops work on overflow; it cannot meet durability requirements.

**Rationale**: The audit worker demonstrates an existing PostgreSQL claim/retry pattern. A separate notification store preserves distinct content-retention and delivery semantics.

**Alternatives considered**: The in-memory recovery queue was rejected because it loses work on restart/overflow. Reusing the audit outbox was rejected because audit facts and email attempts have different retention, security, and search needs.

**Failure signal and reconciliation**: Failure to retain dispatch work MUST NOT roll back the owning action. Emit a secret-free critical structured event and health/metric signal; reconcile from the stable authoritative source occurrence through an owner-supplied replay contract. Production requires a durable operational signal and named recovery owner.

**Password-recovery handoff**: Replace the existing in-memory `PasswordRecoveryQueue`/worker as a delivery path with durable notification work; do not leave a parallel direct-send path. Keep `PasswordRecoveryService`'s same generic public response for known and unknown addresses. The notification worker resolves eligibility and generates the Identity proof only for an authorized attempt, then sends through the provider-neutral `IEmailSender` adapter. Preserve fresh-proof behavior, idempotency, and the ban on storing/logging proof or rendered security content. Tests must prove one logical send after replay and unchanged public responses.

## Security Content and Identity Workflows

**Decision**: Permanent records for security events contain safe metadata and `contentWithheld`; never persist rendered subject/body, proof, or URL. Generate invitation/confirmation/reset proof only for an authorized delivery attempt, render in memory from the accepted template revision, and discard rendered content after the call. Retry/resend must request fresh security material from the owning workflow. Never log message bodies, proofs, provider credentials, or feedback submissions.

Password-reset requests retain the submitted address only in protected, short-lived dispatch work until the worker resolves it. Unknown/ineligible addresses create no permanent notification record and retain the current anti-enumeration response. Do not store reset proofs. The service registers ASP.NET Core Data Protection; production needs a durable/shared protected key ring across restarts and instances.

**Evidence**: [PasswordRecoveryService.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Application/PasswordRecovery/PasswordRecoveryService.cs) resolves the user and generates an Identity reset proof in its worker step. [DependencyInjection.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/DependencyInjection.cs) registers Data Protection and Identity token providers. The existing recovery queue is not durable.

**Rationale**: Just-in-time proof generation avoids a permanent proof archive. Protected transient request work preserves recovery requests without exposing unknown-account lookup results.

**Alternatives considered**: Persisting rendered security mail/proofs in permanent records, attempts, previews, or logs violates FR-018. Using a real proof in preview violates FR-014.

**Credential-setup dependency**: Source and Teacher Management documentation show that teachers are provisioned as pending, but no initial credential-setup proof or activation workflow exists. Do not reuse password-reset tokens or invent activation rules. The identity/account owner must supply a secure setup-proof/link contract before teacher invitation delivery can pass end-to-end acceptance; until then, test only the producer contract and keep the account pending.

## Templates, Rendering, and Revisions

**Decision**: Seed one email template per event type. Keep a stable template definition and immutable revisions; each save creates a revision and advances the current pointer. Accepted notifications pin a revision. Use allowlisted named placeholders, plain-text bodies, and no executable expressions or HTML rendering. Validate required action-link placeholders for security templates. Preview may return synthetic content, including a visibly synthetic, nonfunctional action-link placeholder, but never resolves accounts, generates proofs, or persists preview output. Security templates remain enabled; only ordinary academic templates may be disabled.

**Concurrency**: Updates carry the revision the Admin viewed. A stale revision returns conflict without replacing the current revision. The reminder policy is one portal-wide positive integer number of minutes; no default is active until an Admin saves a value.

**Rationale**: Immutable revisions preserve ordinary message snapshots across edits. Plain text matches the existing sender contract and avoids executable supplied markup.

## Branded Email Layout (added 2026-10-16)

**Decision**: Keep template wording as validated plain text, and wrap every message in one fixed IIT-branded HTML layout (`EmailLayout`) at send time, sending the plain text as the alternative. The HTML is never stored (not even for ordinary messages), so history, snapshots, retries, and the no-stored-security-content rule are unchanged. The branding values are `Notifications:*` settings (`LogoUrl`, `BrandColor`, `PortalName`, `LogoAlt`, `FooterText`, `ButtonLabel`) with the IIT values as defaults, validated at startup. The logo is an `<img>` with a public HTTPS address (default `{Notifications:PortalUrl}/images/iit-logo.png`, override `Notifications:LogoUrl`), with the portal name as text when no usable address exists.

**Rationale**: Branding is the system's job, not the Admin's: it keeps FR-013's no-markup rule and preview guarantees, avoids an HTML sanitizer and email-injection risk, and keeps one consistent look. Brevo accepts `htmlContent` and `textContent` together and sends both as alternatives.

**Alternatives considered**: Admin-editable HTML (rejected: reverses the plain-text rule, needs sanitizing and a stricter preview); per-template approved header images (rejected for now); an inline/embedded logo (rejected: larger messages and unverified provider support); an HTML template stored with each revision (rejected: stored content for security messages).

**Limit**: mail clients load the logo from the internet, so a localhost address will not display for real recipients. A real-client rendering check (Gmail, Outlook, mobile) has not been done; the layout is table-based with inline styles for that reason.

## History, Administration, and UI

**Decision**: Expose explicit Admin REST resources protected by the current active-role Admin policy. Search history using stable keyset ordering `(requestedAtUtc DESC, notificationId DESC)`, default batch 25 and maximum 100, and an opaque cursor. Preserve recipient IDs as values without a cascading Identity foreign key. Use the existing audit recorder for template, policy, and resend mutations.

**Evidence**: [AuditEventsController.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/AuditEventsController.cs) and [AuditEventResponse.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/AuditEventResponse.cs) demonstrate cursor search. [audit-page.ts](../../iit-academic-portal/src/app/features/audit/audit-page.ts) and [infinite-scroll-sentinel.ts](../../iit-academic-portal/src/app/shared/infinite-scroll/infinite-scroll-sentinel.ts) implement shared infinite loading. [AuditTransactionTests.cs](../../iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Auditing/AuditTransactionTests.cs) covers business-audit transaction behavior.

**Rationale**: This follows REST, server authorization, bounded collections, append-only history, and the design system's required cursor/infinite-loading pattern.

## Producer Coverage and Testing

**Decision**: Wire existing teacher creation and password-recovery requests through the notification contract where the owning account workflow supplies the required event/proof data. Define acceptance contracts for student invitation, email confirmation, published results, and feedback-window events whose owners are absent. Missing owners remain dependencies; do not create student, results, or feedback workflows here.

**Evidence**: [Teacher Management research](../004-teacher-management/research.md) deferred setup email. Current source has no notification catalog/history, initial credential setup, email-confirmation producer, results-publication producer, or feedback-window producer. API tests use xUnit, `WebApplicationFactory`, SQLite, controllable time, and a fake email sender; frontend tests use Vitest and `HttpTestingController`.

**Testing decision**: Reuse those frameworks. Add PostgreSQL 18 coverage for migration, uniqueness/retention constraints, leases, cursor query, and runtime grants; deterministic API/UI tests cover delivery state, privacy, template validation, reminders, resend, and loading/error states. Add a fake non-production channel adapter contract test to verify producers remain channel-independent without enabling a production channel. Use the Brevo adapter for Development smoke tests with synthetic identities only; use fake adapters for automated tests.

## Source References

- [PortalDbContext.cs](../../iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs)
- [runtime-grants.sql](../../iit-academic-portal-service/database/postgresql/runtime-grants.sql)
- [PortalApiFactory.cs](../../iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Infrastructure/PortalApiFactory.cs)
- [PostgresFixture.cs](../../iit-academic-portal-service/tests/IitAcademicPortal.Api.Tests/Infrastructure/PostgresFixture.cs)
- [Teacher Management API contract](../004-teacher-management/contracts/teacher-management-api.md)
- [IIT design system](../design-system.md)
