# Implementation Plan: Notification Management

**Branch**: `feature/005-notification-management` | **Date**: 2026-10-16 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/005-notification-management/spec.md`

**Status**: Implemented. Invitation delivery and the student, publication, confirmation, and feedback-window producers remain external integration gates.

## Summary

Build a reusable, channel-independent notification capability for seven account and academic events. Use Brevo only for Development email delivery and replace it with an approved Production provider before release. Keep provider choice behind the existing email abstraction, persist accepted work/history in PostgreSQL, and process it with a notification-specific leased worker. Route password-recovery requests through this durable path, retiring the existing in-memory direct-send path to prevent duplicate messages while preserving the public generic response and Identity proof rules. Reuse the active-role Admin policy, audit, and Angular infinite-loading patterns. See [research.md](research.md), [data-model.md](data-model.md), and [contracts/](contracts/).

## Technical Context

**Language/Version**: Angular 21 / TypeScript 5.9 / Node.js 24 LTS; ASP.NET Core 10 / C# 14.

**Primary Dependencies**: Existing Angular and Tailwind CSS 4 setup; ASP.NET Core Identity 10, EF Core 10, Npgsql EF Core provider 10.x, existing `IEmailSender`, and a Brevo transactional-email adapter using the standard .NET HTTP client. No provider SDK package is planned.

**Storage**: PostgreSQL 18 through `PortalDbContext`; a version-controlled EF Core migration is required for templates, revisions, notification records, attempts, retry state, and reminder policy.

**Testing**: xUnit API tests through `WebApplicationFactory` and SQLite; PostgreSQL 18 migration/query/grant tests through the existing optional PostgreSQL fixture; Angular/Vitest and `HttpTestingController` tests.

**Target Platform**: Browser-based Angular portal and ASP.NET Core API in the existing layered monolith. Development uses Brevo; Production uses a separately approved provider selected before release.

**Project Type**: Existing multi-project web application: Angular frontend plus ASP.NET Core API, application, domain, and infrastructure layers.

**Performance Goals**: Meet SC-002's 200 ms maximum p95 originating-action overhead under email outage and SC-003's 95% of accepted immediate sends submitted within 60 seconds. Admin history uses bounded keyset batches (default 25, maximum 100).

**Constraints**: Email is the only initial channel. Brevo is Development-only; delivery must fail closed unless each destination is in an explicit allowlist of controlled non-production test mailboxes, with synthetic identities/content only. Production must use a different provider unless Brevo is separately approved against FR-018. Use durable PostgreSQL work, bounded retries, explicit uncertain outcomes, and the provider-neutral email boundary. Never persist security-message content or proof in portal history or ordinary logs. Admin access is enforced by current active role. Production provider configuration, durable Data Protection keys, runtime grants, and operational alerting are release prerequisites.

**Scale/Scope**: Seven event types and four Admin capabilities (template management/preview, history/search/detail, resend, reminder policy). No audience-volume estimate is supplied; use bounded worker batches and cursor-based history, and measure representative volume before production sizing.

## Constitution Check

*GATE: evaluated before Phase 0 and after Phase 1 design.*

| Principle / gate | Assessment |
|---|---|
| IIT visual identity and academic usability | PASS. Admin UI reuses the existing shell, design tokens, accessible forms, status labels, and shared infinite-loading pattern. |
| Maintainable layered architecture | PASS. Business rules stay in application/domain services; controllers and Angular components remain thin; email-provider and PostgreSQL adapters remain infrastructure concerns. |
| RESTful API contracts | PASS. Admin operations use resource-oriented GET/PUT/POST routes, explicit DTOs, validation, safe Problem Details, and bounded cursor search. |
| Security and data integrity | PASS. Server checks active Admin role; sensitive proofs are not persisted in history; Development Brevo rejects non-allowlisted recipients; records and attempts survive account removal; migrations and PostgreSQL grants are planned. |
| Reliability, tests, and observability | PASS. PostgreSQL-backed leased work, bounded retry, explicit uncertain outcomes, operational health signals, and API/UI/PostgreSQL tests address the specified risks. |
| Dependency and scope discipline | PASS. Reuse the existing email abstraction, Identity, audit, EF, Angular, and Tailwind infrastructure; call Brevo through standard HTTP without an SDK; no broker or new academic workflow is introduced. |

**Pre-research gate**: PASS. The specification is clarified; all technical choices fit the approved stack and no unresolved specification marker remains.

**Post-design gate**: PASS. The data model, API contracts, tests, and operations stay within the constitution. The PostgreSQL migration and deployment prerequisites are explicit. No constitution exception or complexity justification is required.

## Project Structure

### Documentation (this feature)

```text
specs/005-notification-management/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── notification-management-api.md
│   └── notification-producer-contracts.md
└── tasks.md              # Created later by /speckit-tasks
```

### Source Code (repository root)
```text
iit-academic-portal/
└── src/app/features/notifications/

iit-academic-portal-service/
├── src/IitAcademicPortal.Domain/Notifications/
├── src/IitAcademicPortal.Application/Notifications/
├── src/IitAcademicPortal.Infrastructure/Notifications/
├── src/IitAcademicPortal.Api/Controllers/
├── src/IitAcademicPortal.Infrastructure/Persistence/Migrations/
└── tests/IitAcademicPortal.Api.Tests/Notifications/
```

**Structure Decision**: Extend the existing Angular feature tree and the existing four-layer .NET service. Notification contracts and rules live in Domain/Application, database and email-provider adapters in Infrastructure, and HTTP mapping in API controllers. No separate service or project is added.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
No constitution violations.
