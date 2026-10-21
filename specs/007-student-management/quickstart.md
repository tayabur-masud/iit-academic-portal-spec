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
- At least two active Degrees with configured required credits and minimum CGPA values, matching Batches, a configured Grade scale, and an active Hall for the Hall assignment scenario.
- A published academic-results provider that returns admission-specific passed-course credits and current cumulative CGPA for positive eligibility scenarios. Without it, eligibility must remain ineligible and identify each unavailable academic value.
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
8. Evaluate eligibility separately for two admissions. Verify missing result data is unavailable; credits below the Degree requirement and cumulative CGPA below the Degree minimum are each unmet; equality with the minimum CGPA meets that requirement; missing Hall or registration number keeps the admission ineligible; and only an admission meeting every requirement is eligible. Publish another result and verify the current cumulative CGPA and eligibility reflect it without copying CGPA into Student Management. Verify no cross-admission credit/CGPA carryover and that final-result calculation/publication is unaffected.
9. Attempt each read and mutation anonymously and as a non-Admin. Verify access is denied without exposing student personal data.
10. Enter separate mailing and permanent address details. Select a Division and District for each address; verify both Upazilas and City Corporations are loaded under that District and can be selected as the address locality. Verify the response includes the selected locality and its derived District and Division. Try submitting both locality IDs, neither locality ID, a nonexistent/deleted locality, and a locality from another District; each invalid address is rejected without changes. Verify a referenced City Corporation cannot be deleted and offers deactivation instead.

## Database Migration

The existing `StudentManagement` schema is already applied. The address addendum's migration, `StudentAddressCityCorporations`, adds nullable City Corporation foreign keys for mailing and permanent addresses, indexes, restrictive relationships, and locality exclusivity constraints while preserving existing Upazila references. CGPA remains provider data and requires no Student Management column or backfill. Apply migrations only through the existing schema-owner workflow after schema and test review.

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

## Addendum Validation Results (2026-10-21)

City Corporation address localities and the minimum-CGPA eligibility requirement. Run on Windows 11 with .NET 10 SDK, Node 24, and a local
PostgreSQL 18 server.

| Check | Result |
|---|---|
| `dotnet test` (Release) with `AUDIT_TEST_POSTGRES` | 1183 passed, 1 skipped (the feature 006 Coordinator release-gate test) |
| `npm test -- --watch=false` | 334 passed (29 files) |
| `npm run build` | Succeeded; initial bundle 455.87 kB (budget 500 kB) |
| Migration `StudentAddressCityCorporations` | Additive: two nullable columns, two indexes, two restrictive keys to `CityCorporations`, and two at-most-one-locality checks; applied to the disposable PostgreSQL test databases, and on 2026-10-21 to the Development database at the project owner's request (existing Upazila references unchanged) |
| Results provider | `UnavailablePublishedAdmissionResultsProvider` is registered: credits and CGPA are both reported unavailable, so no admission is eligible in the running portal. Positive integration waits for the academic-results owner (T022, T026) |

Automated coverage:
- **Addresses (`StudentAdmissionTests`):** both locality types; switching between them storing only the new one; 12 refused inputs, each keyed to its field (no district, no locality, both, no details, unknown or deleted locality, a locality of another district, a newly chosen inactive locality, and a district id that is not a District); no Division id accepted; a locality deactivated later kept on update while a new inactive one is refused with nothing saved; and deleting a City Corporation used by an address refused.
- **PostgreSQL (`StudentPostgresTests`, `ConfigurationDeleteIntegrityTests`):** the both-localities check on each address; restrictive keys; the migration keeping existing Upazila references; and a referenced City Corporation blocking both the portal delete and a physical delete.
- **Eligibility (`GraduationEligibilityTests`, with a fake provider):** CGPA below, equal to, and above the minimum; credits available while the CGPA is not, and the reverse; a missing Degree; per-admission values with no carry-over; a newly published value changing the next evaluation with nothing stored; and the API's unavailable result.
- **Angular (`student-form.spec.ts`, `graduation-eligibility.spec.ts`):** the locality selector loading both lists for the chosen District, clearing on District or Division change, sending one locality with `districtId`, and showing missing and refused localities; the CGPA line and requirement for below, equal, above, and unavailable values; and re-checking after a save.

**Not done:** T045, the hand-run address and eligibility scenarios. The published-result scenarios also need the academic-results owner.

