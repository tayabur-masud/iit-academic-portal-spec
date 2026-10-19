# Course Management Quickstart

This guide validates the Course catalog through the existing Angular Admin UI and ASP.NET Core API. The service test suite uses in-memory SQLite by default; PostgreSQL-specific integration tests require the optional environment described in the service README.

## Prerequisites

- Node.js 24 and npm installed.
- .NET 10 SDK installed.
- PostgreSQL 18 configured for local end-to-end use, with the service connection string set as described in `iit-academic-portal-service/README.md`.
- An Admin test account and an active Degree available in the development database.

## Automated Validation

From `iit-academic-portal`:

```powershell
npm test
npm run build
```

Expected: focused catalog tests pass, including Course field rendering, optional prerequisite selection, Degree filtering, client validation, and server field-error display; the Angular production build completes.

From `iit-academic-portal-service`:

```powershell
dotnet test
```

Expected: API tests pass for valid Course creation/update, global code uniqueness, credit values, Degree/prerequisite checks, self-reference and cycle rejection, dependent-course deactivation conflict, Admin authorization, and audit rollback. PostgreSQL integration coverage verifies the migration, database checks, unique index, and restrictive FKs when its optional test connection is configured.

## Manual End-to-End Scenarios

1. Apply the service's pending EF Core migrations to the configured Development database using the commands in the service README, then start the API with the `https` launch profile and start the Angular development server.
2. Sign in with an Admin account and open `/admin/configuration/courses`.
3. Create a Course with a unique code, an active Degree, Mandatory or Elective type, and a total credit value greater than zero. Verify it appears in the active list with its Degree, code, type, total, and audit metadata.
4. Create a Course with only theory or only lab hours where the supplied value is no greater than the total; verify it saves. Create one with both values whose sum equals the total; verify it saves. Try a component greater than total, mismatched sum, negative component, over-precision value, duplicate catalog code, missing Degree, or invalid type; verify the API rejects the request and the form shows field errors.
5. Set a prerequisite from the same Degree. Verify that inactive, cross-Degree, self, and cycle-forming selections are rejected by the server. Change a Course's Degree and verify any change that would invalidate its own or an active dependent Course's prerequisite is rejected without modifying affected Courses.
6. Try to deactivate a Course that is a prerequisite for an active Course. Verify the API returns `409 Conflict`, the form reports the failure, and neither Course changes. Remove dependent references, deactivate, and verify the Course leaves active prerequisite choices while remaining visible under the inactive filter.
7. Repeat a read and a state-changing request using a non-Admin active role; verify the server denies access. Confirm successful Course mutations appear in the existing Admin audit search and that a simulated audit-save failure leaves the Course unchanged.

## Validation Results (2026-10-19)

Run on Windows 11 with .NET 10 SDK, Node 24, and a local PostgreSQL 18 server.

| Check | Result |
|---|---|
| `dotnet test` (Release) with `AUDIT_TEST_POSTGRES` | 1069 passed, 1 skipped (the feature 006 Coordinator release-gate test) |
| `dotnet test` (Release) without `AUDIT_TEST_POSTGRES` | 989 passed, 81 skipped (80 PostgreSQL-gated tests, plus the Coordinator release-gate test) |
| `npm test -- --watch=false` | 305 passed (28 files) |
| `npm run build` | Succeeded; initial bundle 454.63 kB (budget 500 kB) after making the Configuration screens lazy-loaded |
| Migration `Courses` | Applied to the disposable PostgreSQL test databases and, on 2026-10-19, to the Development database; it creates only the `Courses` table, its checks, indexes, and restrictive keys |

New automated coverage: business rules in `CourseCatalogTests`, the HTTP contract (including no DELETE and unchanged student records) in
`CourseCatalogApiTests`, Course authorization in `ConfigurationAuthorizationTests`, audit and rollback in
`ConfigurationAuditTransactionTests`, database constraints in `ConfigurationPostgresTests`, and the Angular course form and list in
`catalog-form.spec.ts` and `catalog-list.spec.ts`. The tests were run in the Release configuration because a running Debug API locks the
Debug build output.

### Not done

- **Manual scenarios 1-7** above were not run in a browser (the migration is applied to the Development database, so they can be).
- **SC-006 usability study (T017)** was not run; `usability-results.md` does not exist yet.

