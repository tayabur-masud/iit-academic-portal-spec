# Quickstart: Student Management Validation

**Feature**: [Student Management](spec.md)  
**Plan**: [Implementation plan](plan.md)  
**Data model**: [data-model.md](data-model.md)  
**API contract**: [student-management-api.md](contracts/student-management-api.md)  
**Implementation notes**: [implementation-notes.md](implementation-notes.md)

## Prerequisites

- Node.js 24 LTS and .NET 10 SDK.
- PostgreSQL 18 configured using the service's documented development connection.
- Existing active Admin account and credential-setup email configuration.
- At least two active Degrees and matching Batches in Configuration; an active Hall is needed for the Hall assignment scenario.
- A passed-course results provider for positive completed-credit scenarios. Without it, eligibility must remain ineligible and identify the credit data as unavailable.
- Student Management schema migration applied through the existing schema-owner workflow before running the API against PostgreSQL.

## Automated Checks

Run from the workspace root:

```powershell
Push-Location iit-academic-portal-service
dotnet test tests/IitAcademicPortal.Api.Tests/IitAcademicPortal.Api.Tests.csproj
Pop-Location

Push-Location iit-academic-portal
npm test -- --watch=false
npm run build
Pop-Location
```

When `AUDIT_TEST_POSTGRES` is configured, run the PostgreSQL integration tests to verify migrations, account/admission constraints, and registration-number uniqueness against PostgreSQL 18.

## End-to-End Scenarios

1. Sign in as an active Admin. Create an initial admission using a valid email, mobile, active Degree, and matching Batch. Verify one StudentProfile, one pending Student account, and one admission are committed with audit; verify one setup email is requested after commit.
2. Create a second admission for the same person using the same normalized email and a different Degree/Batch (for example, EMIT after a recorded PGDIT admission). Verify the Student account/profile IDs are reused, a distinct admission ID is created, no second setup email is requested, and the PGDIT remains visible as a separate prior admission.
3. Verify the existing PGDIT admission is not duplicated as an academic-history entry. Verify its passed credits do not automatically count toward EMIT eligibility; only results associated with the EMIT admission count unless a separate transfer-credit rule is approved.
4. Omit mobile, use an invalid email, or use an email owned by a non-Student account. Verify field/conflict rejection and no partial profile, admission, or account. Use an existing Student email and verify account reuse rather than rejection.
5. Submit only the applicable fields for one profile-level academic-history category, including source values not present in Configuration catalogs, then add a second category. Verify blank and inapplicable fields are not required, supplied text and numeric-looking values are retained exactly, and the history appears from both the PGDIT and EMIT admission views.
6. Complete the setup email flow for a newly created Student account and verify activation only after successful password setup. Try an expired, invalid, or reused setup link; verify the account remains unchanged. Verify later admissions do not reset credentials or resend setup email.
7. Create an admission without Hall or registration number. Assign either value independently, then assign the other. Try a registration number already used by another admission and verify the update is rejected without changing either record.
8. Evaluate eligibility separately for two admissions, including missing credit data, fewer passed-course credits than required, sufficient credits but a missing Hall/registration number, and all requirements met. Verify no cross-admission credit carryover, only the fully satisfied admission is eligible, and final-result calculation/publication is unaffected.
9. Attempt each read and mutation anonymously and as a non-Admin. Verify access is denied without exposing student personal data.
10. Enter separate mailing and permanent address details. Select a Division, then a District, then an Upazila for each address. Verify the returned Upazila, District, and Division path matches Configuration and invalid/missing Upazilas are rejected.

## Database Migration

Implementation will add StudentProfile, StudentAdmissionRecord, and academic-history schema, foreign keys, and unique constraints through versioned EF migrations. The plan does not generate or apply migrations. Apply them only through the existing schema-owner workflow after the schema and test review.

## Validation Results (2026-10-19)

Run on Windows 11 with .NET 10 SDK, Node 24, and a local PostgreSQL 18 server.

| Check | Result |
|---|---|
| `dotnet test` with `AUDIT_TEST_POSTGRES` | 1009 passed, 1 skipped (the feature 006 Coordinator release-gate test) |
| `dotnet test` without `AUDIT_TEST_POSTGRES` | 933 passed, 77 skipped (76 PostgreSQL-gated tests, plus the Coordinator release-gate test) |
| `npm test -- --watch=false` | 287 passed (27 files) |
| `npm run build` | Succeeded; initial bundle 483.07 kB (budget 500 kB); the student pages are lazy chunks |
| Migration `StudentManagement` | Applied to the disposable PostgreSQL test databases, and on 2026-10-19 to the Development database at the project owner's request (together with the pending `DegreeCredits` migration from feature 003). `StudentHistoryCatalogReferences` (academic-history catalog references) was applied to the Development database the same day. No shared or Production database was changed |

Automated coverage by scenario: first admission, account reuse, PGDIT-then-EMIT, non-Student email, required fields, history, addresses,
directory, lookup, and authorization (1-5, 9, 10) in `StudentAdmissionTests`; atomic creation, audit, setup activation, reuse without
setup, and delivery failure (1, 2, 6) in `StudentProvisioningTransactionTests`; Hall and registration number (7) in
`StudentIdentifierTests`; eligibility per admission with a fake results provider, and the API's unavailable result (3, 8) in
`GraduationEligibilityTests`; database constraints in `StudentPostgresTests`; and the Angular specs under `src/app/features/students/`.

### Not done

- **Positive completed-credit integration (T022).** No academic-results owner exists. In the running portal every admission reports its
  credits as unavailable and is therefore not eligible.
- **SC-002 usability study (T025).** Not run; record participants, times, assistance, and results here when it is.
- **Manual end-to-end** with a real setup email and a real admission was not run.

