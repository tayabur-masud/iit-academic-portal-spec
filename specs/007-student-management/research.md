# Research: Student Management

**Date**: 2026-10-19  
**Spec**: [spec.md](spec.md)

## Decisions

### Student identity and program admissions

- **Decision**: Model one StudentProfile per Student portal account and one StudentAdmissionRecord per Degree/Batch enrollment. The first admission creates the account, profile, and admission in one application-owned transaction and requests credential setup after commit. A returning Student's normalized email reuses the Student account/profile and creates a distinct admission; it does not change account status, credentials, or request another setup email.
- **Rationale**: The same person may complete PGDIT and later enroll in EMIT. A one-account-per-admission model would either reject the existing email or create duplicate login identities. `UserProvisioningService.CreateAsync` commits an account-only transaction, so it cannot be called separately from admission creation. `TeacherProvisioningService` demonstrates creating a pending account and linked profile atomically, then requesting setup after commit.
- **Alternatives considered**: Requiring a new email/account for every program was rejected because it duplicates identity and credential records. Automatically merging an account with a non-Student role was rejected because it risks cross-role account takeover. Reusing a generic account-only transaction was rejected because it could leave partial data.

### Account identity and contact data

- **Decision**: Keep email, username, role, and account status on the linked `PortalUser`; link the one-to-one StudentProfile to that account, then link multiple admissions to the profile. Store applicant mobile and person-level fields on StudentProfile, without treating mobile as verified.
- **Rationale**: The existing Identity account is the canonical email and sign-in identity. A StudentProfile supplies the stable person identity shared by program enrollments; it avoids copying mobile and personal details into every admission. The specification explicitly separates applicant mobile from address and guardian phone numbers and does not require mobile verification.
- **Alternatives considered**: A second profile-owned email was rejected as duplicate identity data; phone verification or SMS was rejected as out of scope.

### Structured mailing and permanent addresses

- **Decision**: Store mailing and permanent address details separately on StudentProfile. Each supplied address has one free-text details value and exactly one geographic locality reference: either an Upazila or a City Corporation. Preserve the existing Upazila columns and add separate nullable City Corporation foreign keys; enforce that both are never set together. Derive District and Division from the selected locality.
- **Rationale**: Upazilas and City Corporations are separate Configuration catalogs but both are children of a District. The UI first selects Division and District, then loads both locality catalogs filtered to that District. Separate foreign keys preserve real relational integrity and allow restrictive deletion checks; a check constraint prevents ambiguous dual-locality addresses. Existing Upazila references need no backfill.
- **Alternatives considered**: Free-texting geographic names or storing a bare untyped locality ID was rejected because the database could not validate the referenced catalog row. Replacing Upazila references with City Corporation references was rejected because existing address values must remain valid. Independent client-submitted District/Division references were rejected because they could contradict the selected locality.

### Degree, Batch, Hall, and registration relationships

- **Decision**: Reference existing Degree and Batch records, require their relationship to agree, and reference the existing Hall catalog when a Hall is assigned. Enforce registration-number uniqueness across student admissions in application validation and PostgreSQL.
- **Rationale**: Configuration already owns these catalogs. Reusing their stable identifiers prevents copied labels and keeps student admission data aligned with Admin-managed offerings. The later Hall and registration updates remain independent.
- **Alternatives considered**: Copying Degree, Batch, or Hall names into student rows was rejected because it duplicates configuration state. Requiring Hall or registration number at initial admission was rejected by the specification.

### Profile-level prior academic history

- **Decision**: Store zero or more `StudentAcademicHistory` entries as children of StudentProfile, shared across the person's program admissions. `Category` is a fixed six-value enum. School/college, university, institution, board, and group use a Configuration catalog reference plus a saved name snapshot when the value is listed; an unlisted value is stored as typed text with no reference. School/college and university references must match the field's institution type. Every other applicable source-form value is an optional text snapshot.
- **Rationale**: Prior qualifications describe the person rather than one enrollment. Catalog choices allow validated selection for standardized institutions, boards, and groups while name snapshots preserve the historical label if a catalog entry is later renamed or deactivated. Typed fallback retains values from paper forms that are absent from catalogs. Optional text snapshots preserve values and formatting for other fields the form may omit.
- **Alternatives considered**: Requiring every value to exist in a mutable catalog was rejected because paper-form values may be unlisted. Storing all listed values only as text was rejected because it loses the catalog relationship and validation. One wide mandatory row was rejected because it would require irrelevant fields. Scans or photos were rejected by the specification.

### Completed-credit source and eligibility

- **Decision**: Keep eligibility computed and read-only. Read the required credit total and minimum CGPA from the selected Degree, and read both passed-course completed credits and the current cumulative CGPA for that admission from published academic results. Replace the narrow `ICompletedCreditsProvider` contract with an admission-keyed `IPublishedAdmissionResultsProvider` returning those two values, or an explicit unavailable result. Student Management compares the published CGPA with the Degree minimum; the academic-results owner calculates CGPA from published course results using the configured Grade numeric values. Do not calculate or persist CGPA, copied credit totals, or course results in Student Management.
- **Rationale**: The user's clarification assigns CGPA calculation to published course results and confirms that the Degree's minimum is compared to the current cumulative CGPA for the same admission. Configuration owns the required credit and minimum-CGPA settings; academic results owns the published result and Grade numeric-value calculation. The provider boundary keeps this ownership separation while letting missing data remain explicitly unavailable.
- **Unavailable-source behavior**: Until the academic-results owner supplies and binds its read-only source, the registered provider returns unavailable for credits and CGPA. The eligibility response identifies each unavailable academic requirement and never returns eligible. Re-evaluating after another semester result is published reads the then-current cumulative CGPA; no copied CGPA becomes stale in Student Management.
- **Alternatives considered**: Calculating CGPA from marks or course-grade rows in Student Management was rejected because the academic-results owner publishes the result and owns configured Grade numeric-value use. A Student Management `CompletedCredits` or `Cgpa` field, duplicate result tables, a last-semester-only CGPA, or cross-admission values were rejected by the clarified ownership and current-cumulative rule. Treating missing data as zero or eligible was rejected by FR-006.

### Returning Student and previous programs

- **Decision**: Keep every IIT program enrollment as its own StudentAdmissionRecord under the StudentProfile. A previously recorded PGDIT admission remains visible as a separate prior program and is not duplicated as StudentAcademicHistory. Completed credits are keyed to the current admission and do not carry from PGDIT into EMIT without a separately approved transfer-credit rule.
- **Rationale**: Degree, Batch, Hall, registration number, and graduation eligibility are program-specific. Distinct admissions preserve the old program's identity and outcomes while allowing the same account to enter another program.
- **Alternatives considered**: Overwriting the PGDIT admission with EMIT, duplicating the PGDIT admission as a `StudentAcademicHistory` row, or copying PGDIT credits into EMIT was rejected because it loses program history, duplicates history, or invents a credit-transfer rule.

### API, authorization, and UI conventions

- **Decision**: Use resource-oriented Admin student routes, explicit request/response DTOs, active-Admin authorization on the server, and Angular forms/lists matching existing User and Teacher Management features. Use the existing audit and notification services and bounded pagination for the directory.
- **Rationale**: This follows the repository's REST, authorization, validation, and UI conventions and keeps the feature in the current layered monolith.
- **Alternatives considered**: A separate service, a new identity flow, a custom audit system, or new UI framework were rejected because equivalent project services and patterns already exist.

## Repository Findings

- `UserProvisioningService` creates generic Admin or Student accounts in `PendingCredentialSetup`, sets no password, records account creation, commits, then requests the shared setup email.
- `AccountCredentialSetupService` activates pending accounts only after a valid, single-purpose setup proof and password; Admin reissue is already supported.
- `TeacherProvisioningService` creates a linked account and profile in one transaction and requests setup notification after commit; this is the closest atomic provisioning pattern.
- Degree, Batch, and Hall are existing Configuration records. Degree owns the required-credit total; Batch already refers to Degree.
- Configuration entities use audited base classes and migrations are generated through the existing EF Core setup.
- No Student profile/admission entity, academic-history entity, CourseResult entity, or completed-credit provider was found in the current source tree.
- Existing frontend features under `src/app/features/users` and `src/app/features/teachers` provide the list/form and typed API patterns to follow.

## Open Integration Dependency

The academic-results owner must expose a read-only published-results source keyed to the Student admission identifier, returning passed-course completed credits and the current cumulative CGPA. Until that source is available and bound, eligibility must report the corresponding academic requirement(s) as unavailable and remain ineligible. This does not authorize Student Management to calculate or store either value.