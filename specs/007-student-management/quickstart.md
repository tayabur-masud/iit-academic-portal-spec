# Quickstart: Student Management Validation

**Feature**: [Student Management](spec.md)  
**Plan**: [Implementation plan](plan.md)  
**Data model**: [data-model.md](data-model.md)  
**API contract**: [student-management-api.md](contracts/student-management-api.md)

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