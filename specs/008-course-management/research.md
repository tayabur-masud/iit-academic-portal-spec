# Course Management Research

**Feature**: 008 Course Management  
**Date**: 2026-10-19

## Decisions

### Application structure

**Decision**: Add Course as a resource in the existing configuration catalog instead of creating a parallel catalog stack. Use a focused Course resource for its business rules and extend the shared request/response models and Angular descriptor/form only for Course-specific fields.

**Rationale**: Degree and other administrative catalogs already use the shared configuration pipeline for search, status filters, cursor paging, CRUD, Admin authorization, audit snapshots, field validation, and frontend loading/error states. Course is another administrative catalog, but its prerequisite graph and credit rules are not expressible by the existing generic fields alone.

**Alternatives considered**: A separate Course controller, service, and Angular page would isolate the richer behavior but duplicate established catalog workflows. Forcing Course into the current descriptor unchanged would omit the required breakdown and prerequisite UX. The focused resource plus additive descriptor fields keeps reuse at the workflow boundary without hiding domain rules.

### REST contract and authorization

**Decision**: Expose the Course catalog under the existing `/api/admin/configuration/courses` resource path. Reuse list/read/create/update response and paging conventions. Use update with `isActive: false` for deactivation; do not expose a Course delete operation because the feature specification requires deactivation and does not approve deletion. Add a per-resource delete opt-out to the shared controller and hide the delete affordance for Course in the Angular form.

**Rationale**: The API is a session-cookie REST API protected by the active Admin module policy. State-changing calls require the existing anti-forgery token. The shared controller maps validation failures to field-keyed 400 problem details, conflicts to 409, and missing records to 404.

**Alternatives considered**: A new `/api/courses` controller would duplicate authorization and response mapping. Reusing the shared delete operation would add behavior outside the specified Course operations, so Course routing must opt out of DELETE while retaining shared operations.

### Course fields and persistence

**Decision**: Model Course as an audited configuration record with required Degree, globally unique normalized code, positive total credit hours using the existing two-decimal numeric convention, optional theory and lab values, Mandatory/Elective type, and optional prerequisite Course. Use restrictive foreign keys and indexes for Degree and prerequisite lookups.

**Rationale**: Degree already uses `numeric(8,2)`, normalized-code uniqueness, positive-value validation, and restrictive references in PostgreSQL. The `ConfigurationRecord` base supplies active/deleted state and the common audit fields. The specification makes code uniqueness global; it does not establish name uniqueness, so do not add a new name uniqueness restriction for Course.

**Alternatives considered**: A separate audit model or duplicate code normalization would diverge from current project conventions. Requiring globally unique Course names would be an extra business constraint not stated in the specification.

### Credit breakdown validation

**Decision**: Accept theory and lab independently as nullable decimal values. Each supplied component must be between zero and total credit hours, inclusive, and have no more than two decimal places. When both are provided, require their sum to equal total credit hours; retain the specification's positive-total rule.

**Rationale**: This implements the accepted clarification and matches the Degree precision convention. Database checks should protect the total and the complete two-component case, with API validation returning field-keyed errors.

**Alternatives considered**: Requiring both components would contradict the clarification. Treating one supplied component as a complete allocation would also contradict the accepted rule that a single component is independently optional.

### Prerequisite integrity

**Decision**: Validate that a supplied prerequisite exists, is active, is not the Course itself, and belongs to the same Degree. Before saving a prerequisite change, traverse the existing prerequisite chain and reject a chain that reaches the Course being saved. Reject a Degree change that would make this Course's prerequisite or any active dependent Course's prerequisite cross-Degree, without modifying affected Courses. Reject deactivation while an active Course references the target as a prerequisite, using a Course-specific conflict hook in the shared update pipeline so the API returns 409 without saving.

**Rationale**: A nullable self-reference is straightforward to represent with a restrictive foreign key, but same-Degree, active-state, acyclic-graph, and dependent-deactivation rules are cross-record business invariants and cannot be expressed by that foreign key alone. Keep these checks in the application resource/store boundary and test them through API and persistence tests. Add only the targeted store query needed to traverse prerequisite links and find active dependents.

**Alternatives considered**: Cascading deactivation or clearing dependent prerequisites would silently mutate unrelated Course records and was rejected by the accepted clarification. A self-reference-only database check would not prevent multi-course cycles.

### Audit and failure handling

**Decision**: Reuse configuration audit snapshots and save the Course change and business audit event in the same unit of work. Resolve actor and timestamps server-side. Return validation/conflict responses without persisting partial changes.

**Rationale**: Existing `ConfigurationStore.SaveAsync` saves business changes and staged audit events together. The shared record base is stamped by the database context; clients must not supply audit fields.

**Alternatives considered**: A Course-specific audit table or client-provided timestamps would duplicate or weaken the platform audit guarantees.

### Frontend behavior

**Decision**: Add Course to the existing Admin configuration resource list and form. Include total, theory, lab, type, Degree, and an optional prerequisite selector. Filter prerequisite choices to active Courses in the selected Degree and exclude the Course being edited; preserve server validation as authoritative. Show course credit information in the list/detail form using existing design-system fields and states.

**Rationale**: The shared screen already supports search, status, cursor loading, reference selectors, audit metadata, and field-keyed API errors. Current reference selectors load all rows and only understand required references, so Course needs an optional reference and a Degree-scoped prerequisite lookup rather than unbounded client-side filtering.

**Alternatives considered**: A bespoke page would duplicate list and form interactions. Loading every Course and filtering in the browser would scale poorly and could offer invalid cross-Degree choices.

### Testing and validation

**Decision**: Extend focused Angular catalog tests and ASP.NET API tests. Cover valid and invalid credit combinations, global code collisions, Degree/prerequisite selection, cross-Degree and inactive references, self-reference and cycle rejection, dependent-course deactivation rejection, Admin authorization, audit rollback, and PostgreSQL constraints. Run `npm test`, `npm run build`, and `dotnet test` in their respective submodules.

**Rationale**: The frontend uses Angular's `ng test`; the backend test project exercises the real API pipeline with SQLite, and the service README documents optional PostgreSQL integration coverage. The requirement includes both user-visible and database-integrity behavior.

**Alternatives considered**: Unit-only validation tests would not prove HTTP authorization, transaction behavior, field errors, or the migration's PostgreSQL constraints.

## Repository Evidence

- `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResource.cs`: common list/read/create/update/delete flow, field validation, normalized names/codes, audit snapshots, and transactional save boundary.
- `iit-academic-portal-service/src/IitAcademicPortal.Application/Configuration/ConfigurationResources.cs`: Degree and other configuration resource patterns; Degree currently validates a positive credit total with up to two decimal places.
- `iit-academic-portal-service/src/IitAcademicPortal.Application/Abstractions/IConfigurationStore.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/ConfigurationStore.cs`: generic entity lookup/listing, existence checks, uniqueness checks, and persistence operations.
- `iit-academic-portal-service/src/IitAcademicPortal.Api/Controllers/Configuration/ConfigurationControllerBase.cs` and `iit-academic-portal-service/src/IitAcademicPortal.Api/Contracts/Configuration/CatalogContracts.cs`: REST route operations, validation/problem response mapping, and catalog wire fields.
- `iit-academic-portal-service/src/IitAcademicPortal.Infrastructure/Persistence/PortalDbContext.cs`: Degree precision, live-record uniqueness indexes, audit mapping, and restrictive reference behavior.
- `iit-academic-portal/src/app/features/configuration/configuration.models.ts`, `catalogs/catalog-form.ts`, and `catalogs/catalog-list.html`: resource descriptors, shared form validation/reference selection, and paginated catalog list states.
- `iit-academic-portal-service/README.md`: .NET 10/PostgreSQL 18 prerequisites, `dotnet test`, EF migration commands, and optional PostgreSQL integration setup.

## Resolved Unknowns

- No Course entity, `DbSet<Course>`, API resource, or Course catalog UI exists yet; this feature adds them.
- No new package or framework is required.
- The required schema change is one versioned EF Core migration for the Course table, indexes, constraints, and relationships.
- The feature has no numeric availability, latency, throughput, or catalog-volume target. Reuse existing paged catalog behavior; do not add an SLO absent from the spec.