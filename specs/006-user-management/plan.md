# Implementation Plan: User Management

**Branch**: `feature/006-user-management` | **Date**: 2026-10-18 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/006-user-management/spec.md`

**Status**: Phase 1 design and task breakdown complete.

## Summary

Provide active Admins with a bounded view of portal accounts and safe identity/access lifecycle operations. Reuse ASP.NET Core Identity, `PortalUser.AccountStatus`, existing role invariants, audit transactions, session revocation, and the active-Admin policy. Keep TeacherProfile ownership in Teacher Management and Coordinator assignment ownership with the future batch-assignment workflow. Provision new Admin/Student identities pending, and use one purpose-specific secure setup-email flow to set passwords and activate accounts; an Admin may reissue only while an account remains pending. When the batch-assignment owner provisions a new Coordinator account, that owner requests the same setup event after its account/assignment transaction commits. See [research.md](research.md), [data-model.md](data-model.md), and [contracts/](contracts/).

## Technical Context

**Language/Version**: Angular 21 / TypeScript 5.9 / Node.js 24 LTS; ASP.NET Core 10 / C# 14.

**Primary Dependencies**: Existing ASP.NET Core Identity 10, EF Core 10, Npgsql EF Core 10.x, current audit/session/notification services, Angular HttpClient/reactive forms/router, Tailwind CSS 4, and Vitest. No new package or service is planned.

**Storage**: PostgreSQL 18 and the existing Identity, role, `DefaultRole`, `AccountStatus`, `AuthSessions`, audit, and notification tables. No new account column/table is planned. Add an EF migration to seed the common `AccountCredentialSetup` notification template and initial revision.

**Testing**: Existing xUnit API tests via `WebApplicationFactory` and SQLite; PostgreSQL 18 tests gated by `AUDIT_TEST_POSTGRES` for grants, migration seeds, and shared last-Admin locking; Angular/Vitest and `HttpTestingController` tests.

**Target Platform**: Existing Angular browser application and ASP.NET Core layered-monolith API, with the existing SMTP/Brevo Development email abstraction.

**Project Type**: Existing multi-project web application: Angular frontend and ASP.NET Core Domain, Application, Infrastructure, and API layers.

**Performance Goals**: No new account-specific latency target is defined. Use bounded account searches (default 25, maximum 100), retrieve only identity/role/status fields, and meet SC-002's two-minute representative Admin task-completion target.

**Constraints**: Enforce the active Admin role server-side. Student accounts are Student-only; active Teachers retain Teacher and may receive/remove Admin; Coordinator remains assignment-derived. Preserve the last active Admin. New accounts remain pending until a purpose-specific setup proof is successfully consumed; never reuse password-reset proofs or expose secrets. TeacherProfile stays in Teacher Management. Production's first-Admin bootstrap remains an operations prerequisite because these APIs are Admin-only.

**Scale/Scope**: Four supported roles and the existing portal account store. Search is bounded and Admin-only. Generic account flows create Admin/Student identities; Teacher creation/profile lifecycle delegates to Teacher Management; the batch-assignment owner creates Coordinator accounts/assignments and requests the common setup event after commit. User Management does not create Coordinator accounts or assignments.

## Constitution Check

*GATE: evaluated before Phase 0 and after Phase 1 design.*

| Principle / gate | Assessment |
|---|---|
| IIT identity and academic usability | PASS. Reuse the current Admin shell, form controls, design tokens, accessible validation, and bounded list patterns. |
| Maintainable layered architecture | PASS. Use existing Identity and application services; keep controllers thin; no new project or provider dependency. |
| RESTful API contracts | PASS. Define resource-oriented Admin account endpoints and a separate public credential-setup completion endpoint with explicit DTOs and safe errors. |
| Security and data integrity | PASS. Enforce active Admin on every management request, preserve role invariants, keep tokens out of storage/logs, and make identity/status/audit writes atomic. |
| Reliability, tests, and observability | PASS. Add role/authorization/transaction tests, token replay/reissue tests, and PostgreSQL coverage for the shared last-Admin invariant. |
| Dependency and scope discipline | PASS. Reuse existing account/session/audit/notification infrastructure; do not create TeacherProfile, StudentProfile, or batch-assignment workflows here. |

**Pre-research gate**: PASS. The three specification decisions are recorded and no `[NEEDS CLARIFICATION]` markers remain. Credential setup and Coordinator ownership are explicit design dependencies below, not permission to invent their owning workflows.

**Post-design gate**: PASS. The plan remains within the layered monolith and approved stack; account schema is reused. One EF migration seeds the setup notification definition/template, and release gates for feature-owned integrations remain visible.

## Project Structure

### Documentation (this feature)

```text
specs/006-user-management/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── user-management-api.md
│   └── account-credential-setup.md
└── tasks.md              # Story-organized implementation checklist
```

### Source Code (repository root)
```text
iit-academic-portal/
└── src/app/features/users/

iit-academic-portal-service/
├── src/IitAcademicPortal.Application/UserManagement/
├── src/IitAcademicPortal.Api/Controllers/UserManagementController.cs
├── src/IitAcademicPortal.Api/Controllers/AccountCredentialSetupController.cs
├── src/IitAcademicPortal.Api/Contracts/UserManagementContracts.cs
├── src/IitAcademicPortal.Api/Contracts/AccountCredentialSetupContracts.cs
├── src/IitAcademicPortal.Application/Notifications/
├── src/IitAcademicPortal.Infrastructure/Persistence/Migrations/
└── tests/IitAcademicPortal.Api.Tests/UserManagement/
```

**Structure Decision**: Add an Admin account-management feature under the existing Angular feature tree and ASP.NET Core Application/API layers. Reuse `PortalUser`, `UserManager`, role tables, `AccountStatus`, notification, audit, and session services; add no profile entity. Keep Admin account DTOs separate from the request-only credential setup DTO. Teacher-specific mutations delegate to Teacher Management, and Coordinator account/assignment creation plus setup-event production remain owned by the batch-assignment workflow.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
No constitution violations.
