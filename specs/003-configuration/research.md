# Research: Configuration

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Research date**: 2026-10-13

## Findings and Decisions

### Application architecture

- **Finding**: The service is an ASP.NET Core 10 layered monolith: Domain, Application, Infrastructure, and API projects. The frontend is an Angular 21 SPA in a separate submodule.
- **Decision**: Add a `Configuration` feature slice to those existing projects and `src/app/features/configuration/` in Angular. Keep controllers thin and use explicit request/response DTOs.
- **Rationale**: This matches the existing architecture and the constitution without a new service, project, or framework.
- **Alternatives considered**: A separate configuration service was rejected because the feature is reference data within the existing portal and the constitution requires a layered monolith. A generic key/value catalog was rejected because it would weaken typed relationships and entity-specific constraints.

### Catalog persistence and validation

- **Finding**: `PortalDbContext` owns EF mappings and migrations. The repository has no current academic catalog entities or catalog normalization examples.
- **Decision**: Model each of the eight independent catalogs, Batch, Division, District, and Upazila as typed entities. Name the academic-session entity/table `AcademicSession`/`AcademicSessions` to distinguish it from authentication sessions. Persist normalized comparison fields and express uniqueness/parent relationships in PostgreSQL indexes and foreign keys. Use restrictive delete behavior; lifecycle changes use `IsActive`.
- **Normalization rule**: For names, including Batch names, trim, collapse repeated whitespace, and compare using invariant case normalization. For optional codes, treat whitespace-only as absent, trim, and compare invariantly without changing the display value. Do not accent-fold or invent cross-catalog uniqueness.
- **Rationale**: This implements the spec's whitespace/case duplicate cases at both the application and database boundaries while preserving entered display text.
- **Alternatives considered**: Application-only duplicate checks were rejected because concurrent writes can race. Cascading deletes/deactivation were rejected by the lifecycle requirements. Exact field length caps are not specified; choose and enforce bounded API/database values together during implementation, informed by the owner seed values, without adding a new business constraint in this plan.

### Authorization and audit

- **Finding**: `PortalPolicies.AdminModule` authorizes only the server-validated session's active Admin role. `IAuditEventRecorder.Stage` adds a business event to the current unit of work without saving; the caller saves the change and event together through `PortalDbContext`.
- **Decision**: Apply the existing Admin policy to all configuration mutations and re-check it on every request. Stage one allowlisted business audit event for each successful catalog, batch, geographic, or bulk-seed write, then persist it in the same transaction. Reuse the existing `AuditTransactionTests` pattern for rollback on audit failure.
- **Rationale**: This preserves the portal's authorization boundary and its atomic-audit guarantee.
- **Alternatives considered**: UI-only authorization, client-supplied roles, a second audit implementation, and security-outbox recording for business changes were rejected.

### API and validation contracts

- **Finding**: Existing APIs use resource-oriented routes, explicit DTOs, active-session policies, safe Problem Details, and field-keyed `ValidationProblem` responses. Audit has a versioned OpenAPI 3.1 contract.
- **Decision**: Define REST collection/item routes under `/api/admin/configuration`, resource-specific request/response shapes, active/inactive filters, and bounded paging where collection size warrants it (numbered pages with a total count for the Admin screens, as audit history uses, and cursors for clients that need pages that never shift). Use the existing OpenAPI generation in the service and keep the planning contract in the feature's `contracts/` directory.
- **Rationale**: This is consistent with existing clients and error handling; duplicate and invalid-reference errors remain actionable at the field level.
- **Decision**: Expose physical delete only for unreferenced records. Restrictive foreign keys make the database the final guard; a referenced-record delete returns a conflict with deactivation as the safe alternative. Audit each successful delete in the same unit of work.
- **Alternatives considered**: Cascading deletion and disabling all physical deletion were rejected: the former violates integrity/history, while the latter does not satisfy the Admin's requested delete operation. A client-supplied role or one untyped catch-all DTO was also rejected.

### Geographic source and import

- **Finding**: No owner-provided JSON file or reusable geographic loader exists in the repositories. The feature spec assigns the approved official source and version to the data owner.
- **Decision**: Provide active-Admin-authorized validate and import API operations accepting the documented JSON body. Validation produces a report without writes. Import validates the complete hierarchy and uniqueness rules before opening one transaction; it writes nothing on any validation error, records the source/version and counts in an allowlisted audit event, and does not overwrite pre-existing Admin-managed rows. Re-running against populated rows reports conflicts rather than silently updating them.
- **Rationale**: This keeps a large external seed out of schema migrations, prevents partial hierarchy loads, preserves Admin edits, and applies the same active-role boundary to bulk and individual configuration writes.
- **Alternatives considered**: An operator-only CLI that bypasses the active Admin authorization boundary, invented or manually transcribed geography, partial per-row imports, silent upserts, and embedding an unapproved dataset in an EF migration were rejected.
- **External prerequisite**: The owner must supply the approved JSON and source/version citation before seed validation can pass release acceptance. The importer format and behavior can be designed and tested with synthetic fixtures, but production source fidelity cannot be claimed until then.

### Frontend and testing

- **Finding**: Angular uses typed `HttpClient` services, reactive forms, accessible state announcements, and Vitest. Backend API tests use xUnit, `WebApplicationFactory`, SQLite for pipeline coverage, and dedicated PostgreSQL 18 tests where provider behavior matters.
- **Decision**: Reuse these patterns for catalog lists/forms, loading/empty/error/inactive states, and Admin navigation. Label the Admin menu entry “Configuration” and use `/admin/configuration`; hide it for other active roles while keeping route/API guards authoritative. Add PostgreSQL integration coverage for indexes, foreign keys, deletes, and migrations; do not treat SQLite as proof of PostgreSQL-specific behavior.
- **Rationale**: This tests behavior at the existing boundaries without adding dependencies.
- **Alternatives considered**: New UI/test frameworks and SQLite-only database verification were rejected.

## Technical Goals and Open Inputs

- The feature specification sets no numeric latency or throughput SLO. Use bounded paging and measure representative API interactions; do not invent a target.
- No design clarification remains open. The approved geographic JSON is an owner-supplied release input, not a choice for implementation to guess.
